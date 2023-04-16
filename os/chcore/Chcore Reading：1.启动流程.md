
启动流程分为：

- bootloader
- 内核代码：切换到 el1
- 内核代码：初始化 mini uart

### 1. bootloader

不同的 ARM 设备启动 OS 的方式不同，这里只介绍树莓派的工作，是 GPU 初始化后从 SD 卡加载内核。在树莓派 3B+ 真机上，通过 SD 卡启动时，上电后会运行 ROM 中的特定固件，接着加载并运行 SD 卡上的 `bootcode.bin` 和 `start.elf`，后者进而根据 `config.txt` 中的配置，加载指定的 kernel 映像文件（纯 binary 格式，通常名为 `kernel8.img`）到内存的 `0x80000` 位置并跳转到该地址开始执行。

`bootcode.bin`、`start.elf`、`fixup.dat` 都是在树莓派主机上的，而我们只能修改 `config.txt`。在加电之后，具体的流程为：

- **First stage bootloader**：树莓派上电后，SoC 中的 bootloader 首先被执行，其作用是挂载 SD 卡上的 FAT32 分区，从而加载下一阶段的 bootloader。这部分程序被固化在 SoC 的 ROM 中，用户无法修改。
- **Second stage bootloader (bootcode.bin)**：这个阶段的 bootloader（也就是 bootcode.bin） 会从 SD 卡上检索 GPU 固件（start.elf），将固件写入 GPU，随后启动 GPU。
- **GPU firmware (start.elf)**：本阶段中，GPU 启动后会检索附加配置文件（config.txt、fixup.dat），根据其内容设置 CPU 运行参数及内存分配情况，随后将用户代码加载至内存，启动 CPU。
- **User code (kernel8.img)**：通常情况下，CPU 启动后便开始执行 kernel8.img 中的指令，初始化操作系统内核，在某些情况下，也可以被 U-BOOT 代替，由 U-BOOT 来加载内核。在树莓派 1 代中，User code 部分被保存在 kernel.img 文件中，2 代中，该文件更名为 kernel7.img，3 代中，该文件更名为 kernel8.img。
- 代码被读取到内存的 `0x80000` 位置，PC 寄存器跳转到这个地址开始执行

而在 QEMU 模拟的 `raspi3b`（旧版 QEMU 为 `raspi3`）机器上，则可以通过 `-kernel` 参数直接指定 ELF 格式的 kernel 映像文件，进而直接启动到 ELF 头部中指定的入口地址，即 `_start` 函数（实际上也是 `0x80000` 这段位置，可以参考 linkder.ld 文件，将 `start` 设置为了 `TEXT_OFFSET`）

SD卡里的boot文件可以在这里下载到官方最新的[链接](https://github.com/raspberrypi/firmware/tree/master/boot)，包含如下文件：

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230413_090049.png)

## 2. 内核代码

ELF 的 `_start` 被设置为 `start.S` 中的第一行代码。ELF 文件将会根据 `_start` 函数的位置设置 `elf->entry`，在 `start.elf` 将会根据 `kernel.img` 的 `elf->entry`，找到第一行代码开始执行。

### 2.1 primary

```c
BEGIN_FUNC(_start)

  mrs x8, mpidr_el1
  and x8, x8, #0xFF
  cbz x8, primary /* jump if registers x8 is zero */

  /* hang all secondary processors before we introduce smp */
  b   .
```

ARM CPU 的每个核都会一起启动，从 `_start` 函数开始执行。这个函数将会从 `mpidr_el1` 这个函数中获取当前核心的 CPUID，将 0号CPU 作为主核，由于此时还没有引入多核系统，只有主核可以执行 `primary` 函数，其他核将会通过 `b .` 这个语句循环忙等。

`mpidr_el1` 寄存器的内容可以参考这个 [Arm Armv8-A Architecture Registers](https://developer.arm.com/documentation/ddi0595/2021-12/AArch64-Registers/MPIDR-EL1--Multiprocessor-Affinity-Register)

### 2.2 异常模式切换

进入主核后，OS 将会检查当前 CPU 的异常级别，我们知道 CPU 分为四个异常级别：EL0、EL1、EL2、EL3。

树莓派启动时默认设置成 EL3，也就是安全模式。此时需要从 EL3 切换到 EL1。

为了使得 `arm64_elX_to_el1` 函数更具有通用性，这里没有直接写死从 EL3 降至 EL1 的逻辑，而是根据异常级别的不同，跳转到对应代码执行。

```asm
BEGIN_FUNC(arm64_elX_to_el1)

  /* asuka: we should read exception level to x9 register */

  mrs x9, CurrentEL
  
  // Check the current exception level.
  cmp x9, CURRENTEL_EL1
  beq .Ltarget
  cmp x9, CURRENTEL_EL2

  beq .Lin_el2
  // Otherwise, we are in EL3.
```

首先是读取 `CurrentEL` 寄存器，这个寄存器存储的是当前 CPU 的异常级别。根据异常级别执行相应函数

This instruction will read the content of the `CurrentEL` to a general purpose register `x0`. The register `CurrentEL` is a 64-bit one, but most of the bits are hardwired to be 0 and only bits 2 and 3 contain the actual EL value (see [this doc](https://developer.arm.com/docs/ddi0595/h/aarch64-system-registers/currentel)).

无论从哪个异常级别跳到更低异常级别，基本的逻辑都是：

-  先设置当前级别的控制寄存器（EL3 的 `scr_el3`、EL2 的 `hcr_el2`、EL1 的 `sctlr_el1`），以控制低一级别的执行状态等行为
-  然后设置 `elr_elx`（异常链接寄存器）和 `spsr_elx`（保存的程序状态寄存器），分别控制异常返回后执行的指令地址，和返回后应恢复的程序状态（包括异常返回后的异常级别）
-  最后调用 `eret` 指令，进行异常返回

arm 中没有特殊的指令能够直接到指定的异常级别，对于高级别的异常切换到低级别，需要使用 `eret` 执行进行 fake 的异常返回，以此进行异常切换。

顺带提一下，ARMv8提供了 3 种软件产生的异常，发生此异常的原因是软件企图进入更高的异常等级。可以从低级别异常切换到高级别异常。

-   SVC 允许用户模式下的程序请求os服务
-   HVC 允许客户机（Linux os）请求主机服务
-   SMC 允许普通世界的程序请求安全服务

此时首先是位于 EL3，EL3 可以设置 `scr_el3` 寄存器控制低一级别的执行状态

```c
  // Set EL2 to 64bit and enable the HVC instruction.
  mrs x9, scr_el3
  mov x10, SCR_EL3_NS | SCR_EL3_HCE | SCR_EL3_RW
  orr x9, x9,  x10 // x10 与 x9 相或, 等价于给原 scr 寄存器增加了 3 个新位
  msr scr_el3, x9
```

scr_el3 用来控制 el2 的权限，具体的内容可以看这里[Arm Armv8-A Architecture Registers](https://developer.arm.com/documentation/ddi0595/2021-12/AArch64-Registers/SCR-EL3--Secure-Configuration-Register?lang=en)，scr_el3 寄存器新加的三个位的作用分别是：

- `SCR_EL3_NS`：bit0，表示比 EL3 低的异常级别是不安全，不能访问安全资源
- `SCR_EL3_HCE`：bit8，表示 HVC 指令可以使用
- `SCR_EL3_RW`：bit10，下一个异常级别是 AArch64，如果 EL2 存在，那么 EL2 就是 AArch64，他来控制 EL0 的权限。如果 EL2 不存在，那么下一个异常级别 EL1 就是 AArch64。

```c
  adr x9, .Ltarget
  msr elr_el3, x9
  mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
  msr spsr_el3, x9
```

这里是设置返回的异常级别和返回之后的一些操作：

- `elr_el3`：表示从异常返回后开始执行的地址， 这里是返回到 `.Ltarget`
- `spsr_el3`：表示返回后，对应的异常级别和设置。D 表示返回后设置 PSTATE.D 屏蔽 debug。A 表示返回后设置 PSTATE.A 屏蔽 `SError` 中断；I 表示返回后设置 PSTATE.I 屏蔽 `IRQ` 中断；F 表示返回后设置 PSTATE.F 屏蔽 `FIQ` 中断。`EL1H` 设置 M 位，M 位的最后一位是使用对应异常级别的 `SP`，第三四位是设置返回后的异常级别，这里是设置为 EL1

### 2.3 EL2 的初始化

因为 el3 需要控制 el1 执行状态，所以这里 el3 还执行了 el2 的初始化， el3 可以使用 el2 的硬件资源。

待补。。。。。。。。。。


```c
.Lno_gic_sr:

  // Set EL1 to 64bit.
  mov x9, HCR_EL2_RW
  msr hcr_el2, x9

  // Set the return address and exception level.
  adr x9, .Ltarget
  msr elr_el2, x9
  mov x9, SPSR_ELX_DAIF | SPSR_ELX_EL1H
  msr spsr_el2, x9

isb
eret
```

最后是使用 HCR_CL2 寄存器修改 el1 的执行状态，这里修改为 aarch64，然后设置 el2 的异常返回。实际上由于此时还是 el3，所以 el2 的异常返回不会执行，将会根据前面设置的 el3 的异常返回，直接跳过 el2 返回到 el1。

### 2.4 uart

待补。。。

[raspberry-pi-os/rpi-os.md at master · s-matyukevich/raspberry-pi-os (github.com)](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/translations/zh-cn/lesson01/rpi-os.md)