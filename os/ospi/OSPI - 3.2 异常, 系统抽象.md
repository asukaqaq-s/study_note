
用户程序与 OS 都是大量指令构成的程序，运行在处理器、内存、设备中，区别在于一些功能只有操作系统才有权限执行，比如CPU中断的开关、对频率的调整、对内存的配置、对设备的操作等。

这些功能只能由 OS 才能执行，因为过于危险，当恶意的用户程序执行时，将会导致系统功能的崩溃。为了区别操作系统与应用程序的不同运行环境，现代 CPU 分为不同的特权级：

- **OS**：运行在高权限，支配所有硬件资源
- **应用程序**：运行在低权限，使用部分硬件资源
- **交互**：通过系统调用，应用程序主动获得 OS 的服务；通过异常，应用程序被动获得 OS 的服务。

交互机制需要切换特权级别，让用户程序进入内核态，这种机制叫做 `Trap`。

## 1. 特权级别与 ISA

如果不区分应用程序和操作系统运行的功能，即应用程序和操作系统的运行权限一致，那么应用程序将可以随意更改系统或其他应用的数据。为此，我们需要划分应用程序与操作系统的运行权限。

CPU 将其划分为两种不同的特权级别：用户态、内核态。ISA 作为 CPU 给软件的接口，也划分为 `系统ISA` 和 `应用ISA`。在用户态的软件只能使用用户 ISA，在内核态运行的软件能使用 `系统ISA` 和 `应用ISA`。

ARM 寄存器分为系统寄存器和用户寄存器，同样的，系统寄存器只能由系统指令访问。

**特权级别划分**

Arrch64 中的特权级别称为异常级别EL，CPU 当前的异常级别保存在 PSTATE 寄存器中，当 CPU 发生状态切换时，寄存器中的状态会随之更新，四个级别分为：

- **EL0**：用户态，运行用户程序
- **EL1**：内核态，运行操作系统
- **EL2**：用于虚拟化场景，运行虚拟机监控器
- **EL3**：与安全特性 TrustZone 有关，负责普通世界与安全世界的切换

计算资源被划分为普通世界和安全世界，普通世界不能访问安全世界的资源，而安全世界可以无限制的访问任何资源。
|寄存器||EL0|EL1|描述|
|----| ---- |----|----|----|
|通用寄存器 | X0~X30 | √| √ | |
|特殊寄存器 | PC | √ | √ | 程序计数器 |
| | SP_EL0 | √ | √ | 用户栈寄存器 |
| | SP_EL1 |   | √ | 内核栈寄存器 |
| | PSTATE | √ | √ | 状态寄存器   |
|系统寄存器 | ELR_EL1 | | √ | 异常链接寄存器 |
| | SPSR_EL1 |  | √ | 异常链接寄存器 |
| | VBAR_EL1 |  | √ | 已保存的程序状态寄存器 |
| | ESR_EL1  |  | √ | 异常症状寄存器 |

值得注意的是，在内核态和用户态使用不同的栈，在执行系统代码时会使用用户指令，涉及到用户寄存器 SP，此时 SP 的值是否应该切换为内核栈地址呢？

**x86 的回答是**：要切换，当用户态下发生中断时，将会自动从 TSS 获取内核栈修改 SP 寄存器，然后将用户栈地址存入内核栈中的中断上下文中，后面需要使用中断上下文恢复用户栈到 SP 寄存器。注意，x86 只有一个 SP 寄存器，内核栈和用户栈轮流使用。

**AArch 64 的回答是**：不用切换。ARM 提供了四个 SP 寄存器，其中包括用户栈寄存器：`SP_EL0` 和内核栈寄存器：`SP_EL1`。用户态执行时使用 `SP_EL0`，内核态执行时使用 `SP_EL1`，特权级变化时不需要保存用户栈地址和恢复栈地址，从而降低切换的时延。注意，aarch64 有多个 SP 寄存器，而且没有 TSS 这种设计。

AArch64 的系统寄存器负责保存硬件系统状态，以及为操作系统提供管理硬件的接口。系统 ISA 提供了两条特权指令，用来读写系统寄存器：

- **mrs D，R**：将状态寄存器 R 的值写入到寄存器 D 中
- **msr D，R**：将寄存器 R 的值写入到状态寄存器 D 中

系统 ISA 的指令只有在特权态才能运行，CPU 执行指令时会根据 PSTATE 中的状态来判断是否合法。如果检测失败，将会发生特权级异常。

AArch64 有多个特权级，对于系统寄存器，可以通过后缀判断是在哪一个特权级下使用的，例如 TTBR0_EL1 和 TTBR0_EL2。

## 2. 异常

在用户态下当程序需要和外设进行交互时，如何执行呢？如果用户态直接访问 MMIO 或者 PIO，因为特权级的问题，将会权限不够发生错误，导致进程终止。

那么这时候就需要一种方式让进程通知内核，内核帮助进程完成与外设的交互。也就是进程要切换特权级，从用户态到内核态。在 AArch64 平台中，用户态与内核态的切换使用的是异常机制，其中从用户态切换至内核态的过程称为 **陷入(Trap)**，从内核态切换到用户态的过程称为 **内核态返回**。

为什么称为异常？从应用程序的角度来看，当特殊情况，比如和外设交互，发生应用程序无法处理的错误时，需要下陷到内核态处理。从应用程序的角度来看，下陷到内核是因为出现了**异常**，需要内核处理，内核处理异常的代码称为 **异常处理程序（Exception Handler）**。

> 内核态下执行内核代码也会发生异常，然后跳转到内核中的异常处理程序。唯一的区别是不需要下陷，即切换特权级别。也就是说执行异常处理程序时也会发生异常，一些 CPU 比如 x86 中连续递归三次异常，就会被认为是严重错误：`Triple fault`。

### 2.1 异常分类

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_061029.png)

异常分为三类：

- **中断**：用户程序正常执行时，CPU 收到一些来自外部的事件，打断正在运行的程序，使 CPU 下陷到内核态由操作系统处理，之后恢复用户程序，从被打断的位置继续执行。整个过程用户程序未自知，这类来自 CPU 外部的事件一般称为 **中断（Interrupt）**。
- **异常**：程序在执行中可能会遇到自身无法处理的错误，例如发生了一条格式错误的非法指令，或试图将数据写入到只读的内存。CPU 执行指令中检测到这些事件，同样会触发下陷，操作系统完成后，应用程序继续执行。这类来自 CPU 内部的事件一般称为 **异常（Exception）**。
- **系统调用**：用户程序主动触发异常，向操作系统发出执行特定操作的请求，CPU 通常会为这种情况提供一条特殊的指令（如 Arrch64 的 svc 指令）。这种情况就是**系统调用（System Call）**。因为系统调用在内部发生，所以也算是异常的一种。

中断是异步的，产生原因和当前执行指令无关，如程序被磁盘读打断。而异常是同步的，产生和当前执行或试图执行的指令相关。

异常的控制流不受应用程序控制，发生异常后的目的地址由程序控制，另一方面，这种控制流突变可能在应用程序毫无准备的情况下发生。异常机制中，控制流变化还体现在返回用户态上。

返回用户态后可能有两种情况：

- 执行应用程序触发下陷的指令之后的指令：比如系统调用，中断（因为中断是执行完当前指令的最后一个时钟周期后，CPU 回去查看是否有发生中断，所以是执行下一条指令）
- 重新执行触发下陷的指令

当异常发生后，CPU 只允许从固定的入口开始执行。为此，操作系统需要提前将代码的入口地址告诉处理器。对于不同类型的异常事件，CPU 通常支持设置不同的入口。这些中断入口以一张表的形式记录在内存中，也就是异常向量表，由操作系统负责构造。系统启动后，操作系统会将异常向量表的内存地址写入到 CPU 一个特殊寄存器—**异常向量表基地址寄存器（Arrch64 中是 VBAR_EL1 寄存器）**，然后开启中断，完成异常机制的初始化。

### 2.2 从键盘看 IO

为什么需要异常？异常实际上不仅确保了安全：内核隔离、进程隔离，防止恶意程序修改内容。同时也带来一定抽象，封装底层细节。

**简单场景：应用是如何获得键盘输入？**

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_060211.png)

对于 C 语言来说，一般是使用 `getchar`、`scanf` 这些libc函数获取，他们是对系统调用 `read()` 的封装，此时调用 `read()` 之后，进程进入内核态，通知外设，请求获得外设的输入。

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_055734.png)

OS 如何获取外设给应用的输入，这里有两种方法：轮询或者中断（异步异常）。对于键盘来说，很显然使用中断的方式更好，因为键盘每一次的输入都间隔很长时间，中断机制能够减少无效轮询的情况。

轮询一般适用于 IO 密集型和实时交互型的进程，此时 OS 轮询能够快速获取输入返回到应用程序。

### 2.3 中断信号的产生

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_080140.png)

外设连接在中断控制器的端口上，外设首先发出中断信号到中断控制器，然后由中断控制器路由到对应的 CPU 上。中断控制器需要考虑上面的问题。

**x86**：x86 的中断信号由中断控制器转交给 CPU，中断控制器分为 LAPIC 和 IOAPIC。IOAPIC 是可编程可设置的，OS 在向外设发出请求前会设置好，中断结束信号发生给哪个 CPU。IOAPIC 根据 OS 设置好的重定位表项将对应的中断转交给对应的 LAPIC。


![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_071626.png)

和 x86 一样，arm 也有中断控制器，外设发出中断信号给中断控制器，中断控制器会转交给 CPU，根据来源不同，转交的对象也不同。GIC 的中断来源分为：

- **SPI：共享设备中断**：由所有 CPU 共同连接的设备触发，被路由到一个或多个核上，找到对应的核处理。这个路由是可配置的，中断 ID：32-1019 4096-5119。
- **PPI：私有设备中断**：由每个处理器核上的私有设备触发，指定核处理，如通用定时器（Generic Timer）。中断ID ：16 -31 1056 -5119。
- **SGI：软件产生中断**：发送核间中断进行通信，中断ID：0-15。

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_072325.png)
ARM 中可以根据 MMIO 来进行路由配置，比如上面是启动 timer 的代码，通过 MMIO 设置 GIC 中寄存器。

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_072304.png)
同时可以根据 MMIO，从 GIC 中的寄存器获取中断信息。

### 2.4 x86 中断

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_064812.png)

不同体系结构中中断和异常的叫法差别很大，X86 分为中断和异常，而存储中断入口程序表基地址的寄存器称为 **中断向量表基地址寄存器**（GDT）。

当 LAPIC 收到中断号后，会将对应的中断向量发送给 CPU，CPU 检测到有中断后会让正在执行的进程陷入中断，具体流程是：

- 进程发生中断，检测到发生特权级变化，CPU 从 TSS 中获取内核栈进行切换。将 cs：ip、ss：sp、eflags 寄存器自动压入内核栈中
- CPU 根据 vector 加中断向量表进入对应的中断入口程序
- 操作系统准备好中断上下文
- 调用中断处理程序
- 返回后根据上下文恢复用户进程的执行，执行 iret 指令弹出栈中的 cs：ip、ss：sp、eflags

### 2.5 arm 中断

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_065822.png)
![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_065908.png)
arm 分为同步异常和异步异常。异步异常是进程间交互或者设备发出的中断，同步异常是异常或者系统调用。AArch64 中可以使用上面的指令主动产生异常。对于异常的结果可能是终止进程、修改错误后重新执行等等。

ARM 和 x86 架构的异常类型可能不同。比如ARM 中没有除0异常，除0的结果还是0，而 x86 执行除0时却会触发异常。

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_071210.png)

外中断分为 IRQ 和 FIQ。一般都是 OS 初始化时在中断控制器中配置的，不同的 OS 对应的中断类型不同，而同步异常类型是相同的，由指令架构决定。


### 2.6 arm 的异常处理流程

异常处理需要从用户态切换到内核态。大致分为：保存用户程序的状态；准备操作系统的运行环境。

其中要保存的状态主要是用户程序与操作系统共同使用、可能被操作系统覆盖的处理器状态。

准备操作系统的运行环境通常包含更复杂的操作。

1. 需要准备许多与异常事件相关的信息，例如异常事件的种类、触发异常的指令地址
2. 需要切换成系统栈
3. 根据不同的异常时间找到对应的异常处理函数

这两步由 OS 和处理器协同完成。出于通用性和复杂性考虑，处理器只完成一部分，而OS负责完成剩下的部分。

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_073717.png)

**处理器要执行的任务**：

- 将发生异常时的指令地址，也就是 PC 寄存器值，保存在 ELR_EL1（异常链接寄存器）中
- 将异常事件的原因（例如 svc 导致还是访存缺页导致）保存在 ESR_EL1（异常症状寄存器）中
- 将处理器的当前状态（PSTATE） 保存在 SPSR_EL1（已保存的程序状态寄存器）中
- 保存与特定异常相关的信息，如果引发异常的内存地址保存在 FAR_EL1（错误地址寄存器）中
- 栈寄存器转而使用 SP_EL1
- 修改 PSTATE 寄存器的特权级标志位，设置为内核态（EL1）
- 根据 VBAR_EL1 寄存器中保存的异常向量表基地址，以及发生异常事件的类型，找到异常处理程序的入口地址，来更新PC，执行操作系统的代码。

>为什么不使用应用程序的栈？
>保证内核的数据不会让用户态代码读写

返回时使用特殊的汇编指令—eret

- 将 SPSR_EL1 中的处理器状态写入 PSTATE，处理器状态从 EL1 切换到 EL0
- 转而使用 SP_EL0，SP_EL1 的值没有变化
- 将 ELR_EL1 寄存器的值写入PC，可能已经被修改

**操作系统要做的任务**

在 CPU 执行完自己的任务后。操作系统需要进一步保存处理器上下文。处理器上下文中的寄存器包括：

- 通用寄存器：x0~x30
- 特殊寄存器：PC、SP
- 系统寄存器，比如页表基址寄存器等

对于 PC、SP、PSTATE、系统寄存器等，假如是执行完异常处理程序后立即返回到这个进程，那么不需要保存这几个寄存器到内核栈中，因为可以直接使用异常链接寄存器、程序状态寄存器等恢复。而假如执行完异常处理程序后发生了调度，特殊寄存器和系统寄存器就需要保存到内核栈中，否则会被新进程的上下文覆盖。

由此可见，不同异常的入口程序和返回程序可能不同。或者是特殊寄存器和系统寄存器的保存与恢复是在异常处理程序中完成的，保证入口程序和返回程序的通用性（类似于xv6分为切换上下文和中断上下文）。

### 2.7 Linux 的中断处理

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_074831.png)

中断处理程序是否都要关中断执行呢？
为了防止处理程序执行过久影响并发性能，Linux 将中断处理分为两部分：上半部和下半部。

- **TOP Half**：上半部通常是最小的公共行为（保存和恢复）或者是最重要不允许打断的行为。因为中断被屏蔽，所以不要做太多事情（时间、空间）
- **Bottom Half**：可以允许延迟完成的行为，因为这些任务可以被打断。比如工作队列、内核线程。

## 3. 系统调用

### 3.1 系统调用

系统调用指运行在用户空间的程序向操作系统内核请求需要更高权限运行的服务。系统调用是 OS 提供给应用程序的接口。例如 printf 是通过 write 系统调用实现的。系统调用是一种用户程序主动触发的异常，与一般的异常不同。系统调用类似于函数调用，需要用户提供参数，系统给出返回值。

与函数调用的区别：

- 一个是内核态一个是用户态
- 使用不同的栈

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_084946.png)

在程序员的角度是直接调用库函数，里面封装了 svc 汇编指令，使用 read 函数调用。比如：

```c
mov x0, #__NR_WRITE 系统调用id
mov x1, #1 参数1
mov x2, #2 参数2
mov x3, #3 参数3
svc #0  执行系统调用
```

执行完 svc 后，处理器下陷到内核态，根据异常向量表找到对应的处理函数。系统调用函数会根据中断上下文中保存的寄存器值，读取到对应的参数。最后与函数调用类似，系统调用将返回值存放在调用规约规定的 X0 寄存器，通过 eret 返回到用户态。

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_085415.png)

LINUX 可以使用 strace 跟踪系统调用


### 3.2 参数检测与安全

由于函数调用的参数寄存器只有8个，更多的寄存器需要存储到栈上传递。而系统调用由于要切换栈，所以这些多出的寄存器不能放在内存中。

因此 AArch64 限制了系统调用参数个数，比如加上调用号最多8个。如果需要更多的参数就是用结构体打包，将结构体指针作为参数。

那么引出了一个问题，如果保证内存安全性？**指针需要经过检测!保证内核和进程隔离** 比如指向NULL，内核访问将会崩溃，指向内核虚拟地址属于安全漏洞。

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_090007.png)

因此需要进行如上的检测。

![img](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230412_090252.png)

### 3.3 系统调用与性能

系统参数需要切换特权级别，带来很大的性能开销。怎么优化呢？

- 新的系统调用指令：syscall、sysenter 代替int
- 软件优化：比如进程地址空间有一个特殊的只读内存页，可以让进程访问到内核存储在此的信息，而内核每次获得控制权都会更新他。
- 软件优化：通过以某一个内存页作为信箱，CPU 有多核，分别运行进程和内核，进程在信箱中放入请求，轮询等待完成，而内核轮询信箱获得新的请求并且执行。这种方法可以降低系统调用的时延，在应用将请求写入内存后，内核就会读取到这个请求并且开始处理函数。内核将结果写回信箱后，另一个CPU上正在轮询的进程就可以读到结果。为了提高效率，可以改成批处理请求或者多内核并行轮询。
- 内核批处理系统调用，没有处理完调用前，进程处于阻塞态。这样子总体的cpu利用率会提升

## 4. 系统抽象

抽象和接口是系统设计的一个重要内容，对用户抽象，提供接口。能够保证用户不知道接口的实现，而能够简单使用接口的内容。

基于异常机制和系统 ISA，系统为用户提供底层硬件的使用接口，为用户隐藏管理细节，提供底层硬件的抽象，使应用程序更方便地使用硬件。这里主要有三种抽象：

- **进程/线程**：对 CPU 的抽象
- **虚拟内存**：对内存的抽象
- **文件描述符**：对设备和文件的抽象


### 4.1 进程：对 CPU 的抽象

OS 中通常有几百个应用程序同时运行，然而处理器的个数是有限的。如果某几个应用程序持续霸占处理器，就会导致其他程序无法得到执行的机会。所以应用程序并非每时每刻都需要使用处理器，我们如何设计一种机制，让多个应用程序高效、公平、按需的运行在有限的处理器上呢？

操作系统的做法是轮流使用，又称分时复用。每个进程有一个固定的时间片，当被调度到时，进程最多执行这么多时间片然后就被迫放弃 CPU，CPU 被调度给其他进程。OS 设置每个时间片的时间很短，那么一秒内可以运行几百个进程，宏观上认为每个进程被同时推进了。

OS 为了管理每个程序，创建了进程的抽象，将每个进程运行的信息记录在进程控制块 PCB 中。对于进程提供了独享 CPU、内存的抽象，从而简化了程序的编写。

为了保证调度时不会影响进程的执行，调度发生时需要保存当前进程的上下文以便下次调度回该进程时恢复执行。

OS 为进程提供了一系列服务，即系统调用，通过调用进程就能使用内核资源。

### 4.2 虚拟内存：对内存的抽象

虚拟内存确保了进程隔离、进程与内核隔离。开启虚拟内存之后，CPU读取的地址都是虚拟地址，而实际的物理地址需要虚拟地址经过翻译才能得到。这就确保了：不同进程同样的虚拟地址可以映射到不同的物理地址。地址翻译是 CPU 和 OS 协同完成的，在后面将会看到。

虚拟地址对于编程也带来很多好处：

- 程序员不需要考虑物理内存如何分配，直接可以使用连续的虚拟地址空间
- 每个进程的虚拟地址空间彼此隔离，指向不同的物理地址
- 操作系统选择只将程序实际正在使用的虚拟地址映射到物理内存地址，未使用的物理地址可以不分配
- 部分虚拟内存的数据暂存到磁盘上，可以允许进程使用的虚拟地址总数大于物理地址
- 可以给不同的虚拟地址区域，即不同的虚拟页，设置不同的权限

### 4.3 文件描述符：对设备和文件的抽象

为了方便进程使用外设，OS 为进程提供了一些通用的接口，将管理硬件的细节隐藏起来，程序员通过统一、便利的方式访问这些设备。OS 提供了文件这层抽象，每个文件都拥有自己的文件名、文件地址。

OS 为进程提供了一些系统调用来与文件交互，譬如 `open`、`read`、`write`。

通过 `open` 系统调用，进程可以获得一个独属于他的、关于该文件的文件描述符，通过 `read(文件描述符, 内容)` .. OS 能够识别这个文件描述符是对应哪一个文件，从而与进程进行读写。

除了 `open`、`read` 外，有的 OS 支持 mmap，就是在进程的虚拟地址空间中找出一段区域，执行作为一个文件的缓冲区，当进程向这段区域写内容后，OS 将会自动将脏页写回到对应的磁盘文件中。

Linux 秉持着万物皆文件的思想，将设备都看作为文件，这有两个方面的原因：

- 用户角度而言，设备有与文件类似的操作
- 实现而言，可以复用 OS 中对文件进行操作的大量代码

这种通用的接口带来很多好处，通用、抽象、接口无疑是解耦合，低内聚的！但是对于一些设备而言，并不能通过通用的接口处理，比如操作系统还专门提供了**套接字**调用给网卡设备，还提供更通用的底层接口 `IOCTL`。


## 5. FAQ

**5.1 右移分为算术右移、逻辑右移，而左移只有逻辑左移？**

因为有符号数，符号位在第一位，算术右移能够保证不影响补码的大小

**5.2 数据流分支执行与控制流分支执行的优劣？**

数据流分支指令能够有更多的指令更灵活，但是对应的不利于上手。

**5.3 哪些局部变量存储在栈的局部变量区？**

在寄存器不够保存下所有的寄存器变量时
或者涉及到地址的局部变量

**5.4 为什么分为调用者保存与被调用者保存**

兼顾短变量和长变量

**5.5 为什么需要多个特权级分别运行用户程序和操作系统？**

防止用户程序权限过大，修改系统资源，导致系统崩溃。
特权等级的最大作用就是保护。保护的对象可以是各种内存、IO等资源，其出发点就是对用户不放心！觉得用户写的代码会出各种bug，会破坏系统运行，因此控制用户的权限。而类似操作系统这样的代码，由于经过千锤百炼，比较完善，CPU可以放心地将最高权限交给他。

**5.6 同步异常和中断的区别？**

同步异常：CPU 内部产生的
中断：外部事件发送给 CPU 的

**5.7 特权级切换如果不保存 PC 和 SP？**

如果切换的瞬间不保存 PC、SP，那么 PC 将被覆盖，并且将会在用户栈上面执行内核代码。

**5.8 为什么需要软硬件一起保存状态，而不是操作系统完成所有任务？**

很简单，因为如果让操作系统完成所有任务，是办不到的，必要的任务一定是要 CPU 来完成。比如换栈，比如更改SP。对于保存状态，OS 不会知道发生错误的虚拟地址在哪里，地址指令在哪里

**5.9 为什么要系统调用？如果像普通函数调用来实现系统调用，会出现什么问题？**

系统调用是用户进程来获取操作系统服务的一个接口，如果实现为函数调用，那么进程的权限将会是内核级，或者是级别不够无法完成所有内容

**5.10 为什么需要进程调用 waitpid 才释放子进程所有资源？**

- 因为子进程不能保证 exit 调用一定能够成功，可能会发生失败，父进程回收保证一定能够回收干净，这是一种接口简单、实现复杂的设计。
- 子进程执行 exit 时还会用到自己的一些资源，可能是页表、内核栈等，需要让父进程负责回收

**5.11 使用多个处理器、需要使用多进程。请思考可能带来的弊端？**

多进程之间很难进行共享，有时候需要多进程之间共享运行状态。
从而有了线程。




