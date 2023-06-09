
## 1. 什么是操作系统

操作系统的主要职责：

- 对硬件进行管理和抽象
- 给应用提供服务并管理

### 1.1 硬件角度

在硬件角度而言，操作系统主要实现了管理硬件和硬件抽象。

计算机中的硬件：物理内存、外设、寄存器、缓存等。首先是介绍 OS 如何管理物理内存，对于物理内存、OS 给用户提供了什么样的抽象？

计算机提供**虚拟内存抽象**给用户程序使用，而 OS 就负责将虚拟内存映射到物理内存，同时负责管理物理内存。下面将可用来映射的物理地址称为物理内存，所有的物理地址合称为物理地址空间；物理地址空间上不是所有的地址都可以分配给用户使用的，物理地址空间包含了物理内存 DRAM 和 ROM、BIOS、内存映射式的 IO 等等，物理内存 OS 提供的**硬件管理其中一个就是管理物理地址空间**，记录可用的物理内存区域，使用物理内存分配器分配。

OS 还负责管理各种外设，使用 CPU 与外设用如下方式交互：

- 可以通过物理地址空间上特定的内存映射IO访问
- 另一个是通过特定的 IO 指令访问外设

通过这两种方式，OS 可以与外设交互，获得外设的信息和执行情况，进而管理外设硬件来进行资源调度，这提供了一层抽象，应用只需要负责调用接口，不需要在意具体实现细节。

### 1.2 软件角度

在软件的角度，OS 负责：给应用提供系统服务、管理应用

- **系统服务**：应用可以使用 OS 提供的系统服务接口，免于实现细节和资源管理，通过少量的代码，就能获取不同类型的功能
- **管理应用**：应用即为一个或多个进程，管理应用即为 OS 管理进程，这里仍然涉及到虚拟和接口，OS 能够从全局进行资源分配、应用安全隔离等等


## 2. 操作系统接口

### 2.1 系统服务

虚拟、接口、抽象是整个 OS、整个系统设计最重要的部分，在书中第三章会详细介绍 OS 主要使用到的抽象，这里简单讨论一下系统服务。

古早的系统服务就是进程使用 `系统调用接口` 获取系统提供的服务，以 `write` 为例：

- 进程使用 `write` 进入内核态
- 内核根据系统服务号知道要执行什么功能
- 执行系统中的 `sys_write` 函数
- 执行完后，返回到用户态

进程只需要调用，而不需要知道具体的细节，这种抽象给应用带来很大的帮助，随着新硬件的出现，应用不需要修改代码，调用简单的 `write` 就能执行相同的功能（比如 硬盘替换到非易失内存，执行相同的代码，速度更快！）

但是这也有一个缺陷，硬件种类和应用需求越来越丰富，可能用户对硬件需要特定的 `workload`，通用的接口带来的性能不够大。

因此有了一个新的系统服务架构：OS 分为三层：内核、系统服务、应用框架

![IMG](https://image-1309461627.cos.ap-nanjing.myqcloud.com/image/markdown/os/f3/clipboard_20230404_080608.png)
- 内核：提供通用的、稳定的接口，通过内核给定的接口向内核申请服务。
- 系统服务：对内核提供的功能进一步抽象与封装，或者整合不同内核的系统服务，方便应用在不同内核上执行。
- 应用框架：基于系统服务实现，根据自己的 `workload`，调优硬件，提升硬件的性能，封装面向不同领域的应用接口，使用更加简便，功能更加精简。

> 如果只有一个应用，应用直接控制硬件，还需要 OS 吗

不使用 OS：

- 优点：可以直接管理硬件，减少硬件抽象的性能消耗
- 缺点：一旦硬件改变，应用也要进行改变

解决办法，实现一个具有硬件管理和抽象的库，称为 `LibOS`


### 2.2 实例

书中介绍了三个层次中的几个类型：

- 系统调用接口：Linux、macOS
- 系统服务接口：POSIX 接口
- 应用程序接口：Andriod、ROS

> ABI：应用二进制接口，包含如何定义二进制格式（windows：EXE、Linux：ELF）和应用之间的调用约定（参数传递和返回值处理）、数据模式（大小端）
> 
> API：应用编程接口

## 3. 思考题

1.属于 OS ：b、linux 你和与所有设备的驱动

2.windows 无法运行 Linux 代码，是因为 ABI 不同导致的，执行代码与系统调用没有啥关系