# 第 6 章 中断与设备驱动

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/interrupt.html>。原书采用 CC BY 3.0 许可。

**驱动**（driver）是操作系统中管理某个特定设备的代码：它配置设备硬件，让设备执行操作，处理随之产生的中断，并与使用该设备的进程交互。驱动代码可能很棘手，因为驱动与设备并发执行，而且常常还与使用设备的进程并发执行。此外，驱动必须理解设备的硬件接口，而这些接口可能很复杂，文档也写得很差。

需要操作系统关注的设备通常可以配置为产生中断，中断是陷阱的一种。内核的陷阱处理代码能识别出设备发出了中断，并调用驱动的中断处理程序；在 xv6 中，这个分派工作在 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188) 中完成。

很多设备驱动在两种上下文中执行代码：在进程内核线程中运行的**下半部**（bottom half），以及在中断时执行的**上半部**（top half）。下半部通过 `read`、`write` 这类希望设备执行 I/O 的系统调用被调用。这部分代码可能会要求硬件启动一个操作（例如让磁盘读取一个块），然后等待操作完成。设备最终完成操作并发出中断。驱动的中断处理程序充当上半部，弄清楚哪个操作完成了，在合适时唤醒等待的进程，并让硬件开始处理下一个操作（如果有的话）。

## 6.1 代码：控制台输入

控制台驱动[（console.c）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c)是展示驱动结构的一个简单例子。控制台驱动通过连接在 RISC-V 上的 **UART** 串口硬件，接收人敲入的字符。控制台驱动每次积累一行输入，并处理退格、control-u 等特殊输入字符。shell 等用户进程使用 `read` 系统调用从控制台读取输入行。在 QEMU 中向 xv6 输入时，你的按键是通过 QEMU 模拟的 UART 硬件传给 xv6 的。

驱动与之打交道的 UART 硬件是 QEMU 模拟的 16550 芯片（Michael and Durich 1987）。在真实的计算机上，16550 管理一条连接终端或其他计算机的 RS232 串行链路。在 QEMU 中运行时，它连接的是你的键盘和显示器。

在软件看来，UART 硬件是一组**内存映射**（memory-mapped）的控制寄存器。也就是说，有一些物理地址连接到 UART 设备，对这些地址的 load 和 store 操作与设备硬件交互，而不是与 RAM 交互。UART 的内存映射地址从 0x10000000 开始，即 [`UART0`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L21)。UART 控制寄存器有好几个，每个一字节宽。它们相对 `UART0` 的偏移定义在[（uart.c:25）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L25)中。例如，`LSR` 寄存器中有一些位，指示是否有输入字符等待驱动读取。这些字符（如果有）可以从 `RHR` 寄存器读出。每读出一个字符，UART 硬件就把它从内部的等待字符 FIFO 中删除，FIFO 为空时则清除 `LSR` 中的“就绪”位。发送时，驱动向 `THR` 寄存器写入一个字节，UART 会把这个字节追加到待发送的字节 FIFO 中，然后通过 RS232 串行链路发出去。UART 的发送硬件和接收硬件基本上相互独立。

xv6 的 `main` 调用 [`consoleinit`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L193) 初始化 UART 硬件。这段代码把 UART 配置为：每收到一个输入字节就产生一个接收中断，每发送完一个输出字节就产生一个**发送完成**（transmit complete）中断[（uart.c:50）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L50)。

xv6 shell 通过 [`init.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/init.c#L19) 打开的文件描述符从控制台读取。`read` 系统调用经由内核到达 [`consoleread`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L88)。`consoleread` 等待输入（通过中断）到达并缓存在 `cons.buf` 中，把输入复制到用户空间，然后（在一整行都到达之后）返回用户进程。如果用户还没有输入完整的一行，所有读取进程都会在 `sleep` 调用中等待[（console.c:106）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L106)（第 9 章会解释 `sleep` 的细节）。

用户敲入一个字符时，UART 硬件请求 RISC-V 发出一个中断，从而激活 xv6 的陷阱处理程序。陷阱处理程序调用 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188)，它查看 RISC-V 的 `scause` 寄存器，发现中断来自外部设备。然后它请求一个叫 PLIC（Waterman et al. 2024）的硬件单元告诉它是哪个设备发出的中断[（trap.c:196）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L196)。如果是 UART，`devintr` 就调用 `uartintr`。

[`uartintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L143) 从 UART 硬件读出所有等待中的输入字符，交给 [`consoleintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L147)；它不会等待字符，因为之后的输入会引发新的中断。`consoleintr` 的工作是把输入字符积累在 `cons.buf` 中，直到一整行到达。`consoleintr` 对退格和其他几个字符做特殊处理。换行符到达时，`consoleintr` 唤醒正在等待的 `consoleread`（如果有的话）。

被唤醒后，`consoleread` 会在 `cons.buf` 中看到完整的一行，把它复制到用户空间，然后（通过系统调用机制）返回用户空间。

这里有一个值得注意的模式：通过缓冲和中断，把设备活动与进程活动解耦。即使没有进程在等待读取，控制台驱动也能处理输入；之后的读取会看到这些输入。这种解耦让进程可以与设备 I/O 并发执行，从而提高性能；当设备很慢（比如 UART）或需要立即响应（比如回显敲入的字符）时，这一点尤其重要。这个思想有时称为 **I/O 并发**（I/O concurrency）。

## 6.2 代码：控制台输出

对连接到控制台的文件描述符执行 `write` 系统调用，会到达 [`consolewrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L63)，它分批从用户空间复制字节，并把每一批交给 [`uartwrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L81)。在把每个字节写入 UART 的 THR 寄存器之前，`uartwrite` 必须等待 UART 准备好接受更多输出。因为 UART 相对较慢，`uartwrite` 用 `sleep`（它会让出 CPU）来等待，而不是忙循环。UART 发送完最近一个字节后会发出中断。中断例程 [`uartintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L143) 与 `uartwrite` 相互配合：发送每个字节之前，`uartwrite` 读取 UART 的 LSR 寄存器，看 UART 能否再接受一个字节，不能就调用 `sleep`；当 UART 发出信号表示可以接受更多输出时，中断例程调用 `wakeup`，于是 `uartwrite` 被唤醒，重新检查 LSR 寄存器，然后发送下一个字节。

## 6.3 驱动中的并发

你可能已经注意到 `consoleread` 和 `consoleintr` 中对 `acquire` 的调用。这些调用获取一把锁，保护控制台驱动的数据结构不被并发访问。这里有三种并发危险：不同 CPU 上的两个进程可能同时调用 `consoleread`；某个 CPU 正在 `consoleread` 内部执行时，硬件可能要求这个 CPU 处理一个控制台（实际上是 UART）中断；`consoleread` 正在执行时，硬件可能在另一个 CPU 上投递一个控制台中断。第 7 章会解释如何用锁来确保这些危险不会导致错误结果。

驱动中需要小心对待并发的另一个方面是：一个进程可能在等待设备的输入，但通知输入到达的中断可能在另一个进程正在运行（或根本没有进程在运行）时到来。因此，中断处理程序不能对它所打断的进程或代码做任何假设。例如，中断处理程序不能安全地用当前进程的页表调用 `copyout`。中断处理程序通常只做比较少的工作（例如只是把输入数据复制到缓冲区），然后唤醒下半部代码去完成剩下的工作。

## 6.4 时钟中断

xv6 用时钟中断来维护它对当前时间的认识，并在计算密集型进程之间切换。时钟中断来自连接在每个 RISC-V CPU 上的时钟硬件。xv6 对每个 CPU 的时钟硬件编程，让它周期性地中断 CPU。

[`start.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c#L56) 中的代码设置一些控制位，允许 supervisor 模式访问时钟控制寄存器，然后请求第一个时钟中断。`time` 控制寄存器包含一个由硬件以恒定速率递增的计数，用作当前时间。`stimecmp` 寄存器包含 CPU 将要发出时钟中断的时间；把 `stimecmp` 设为 `time` 的当前值加上 *x*，就安排了在 *x* 个时间单位之后发生一次中断。在 `qemu` 的 RISC-V 模拟中，1000000 个时间单位大约是十分之一秒。

与其他设备中断一样，时钟中断经由 `usertrap` 或 `kerneltrap` 以及 `devintr` 到达。时钟中断到达时，`scause` 的低位被设为 5；`trap.c` 中的 `devintr` 检测到这种情况，就调用 [`clockintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L167)。后者递增 `ticks`，让内核能够跟踪时间的流逝。递增只在一个 CPU 上进行，以免有多个 CPU 时时间走得更快。`clockintr` 唤醒所有在 `pause` 系统调用中等待的进程，并通过写 `stimecmp` 安排下一次时钟中断。

对于时钟中断，`devintr` 返回 2，告诉 `kerneltrap` 或 `usertrap` 应当调用 `yield`，以便在可运行的进程之间多路复用 CPU。

内核代码可能被时钟中断打断，并通过 `yield` 被迫进行上下文切换，这也是 `usertrap` 开头的代码在开中断之前小心保存 `sepc` 等状态的部分原因。这些上下文切换还意味着，编写内核代码时必须意识到，它可能在毫无预兆的情况下从一个 CPU 移到另一个 CPU 上。

## 6.5 现实世界

与很多操作系统一样，xv6 在内核中执行时允许中断，甚至允许上下文切换（通过 `yield`）。这样做是为了在长时间运行的复杂系统调用期间仍能保持较快的响应。不过，如上所述，在内核中允许中断会带来一些复杂性；因此也有少数操作系统只在执行用户代码时才允许中断。

要完整支持一台典型计算机上的所有设备，工作量很大，因为设备很多，每个设备的功能很多，设备与驱动之间的协议可能很复杂，文档也写得很差。在很多操作系统中，驱动的代码量比核心内核还多。

UART 驱动通过读取 UART 控制寄存器，一次取一个字节的数据；这种模式称为**程序控制 I/O**（programmed I/O），因为是软件在驱动数据的搬运。程序控制 I/O 很简单，但速度太慢，无法用于高数据速率。需要高速搬运大量数据的设备通常使用**直接内存访问**（direct memory access，DMA）。DMA 设备硬件直接把收到的数据写入 RAM，也直接从 RAM 读取要发出的数据。现代磁盘和网络设备都使用 DMA。DMA 设备的驱动会在 RAM 中准备好数据，然后只需对一个控制寄存器写一次，就能告诉设备去处理准备好的数据。

当设备需要在无法预测的时刻得到关注、而且这种情况不太频繁时，用中断是合理的。但中断的 CPU 开销很高。因此，网络和磁盘控制器这类高速设备会使用一些技巧来减少对中断的需求。一种技巧是为一整批输入或输出请求只发出一次中断。另一种技巧是驱动完全关闭中断，周期性地检查设备是否需要关注。这种技术称为**轮询**（polling）。如果设备执行操作的频率很高，轮询是合理的；但如果设备大部分时间空闲，轮询就会浪费 CPU 时间。有些驱动会根据当前的设备负载，在轮询和中断之间动态切换。

UART 驱动先把收到的数据复制到内核中的缓冲区，然后再复制到用户空间。这在数据速率较低时是合理的，但对于产生或消耗数据非常快的设备，这样的两次复制会显著降低性能。有些操作系统能够直接在用户空间缓冲区和设备硬件之间搬运数据，通常借助 DMA。

如第 1 章所述，在应用程序看来，控制台是一个普通文件，应用程序用 `read` 和 `write` 系统调用读取输入、写出输出。应用程序可能想控制设备的某些方面，而这些方面无法通过标准的文件系统调用表达（例如在控制台驱动中开启或关闭行缓冲）。Unix 操作系统为这类情况提供了 `ioctl` 系统调用。

计算机的某些用途要求对外部事件做出“实时”响应：保证在有限时间内做出响应。例如，在安全攸关的系统中，错过截止时间可能导致灾难。xv6 不适合实时场景。原因之一是，xv6 的调度器在决定下一个运行哪个进程时，不考虑实时截止时间；而且 xv6 中有很长的关中断内核代码路径，因此可能无法快速响应中断。实时操作系统不仅要解决这些问题，还必须以一种能分析最坏情况响应时间的方式来组织。

## 6.6 练习

1. 修改 `uart.c`，使其完全不使用中断。你可能还需要修改 `console.c`。

2. 为一块以太网卡添加驱动。
