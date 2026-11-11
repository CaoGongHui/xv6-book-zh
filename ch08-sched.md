# 第 8 章 调度

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/sched.html>。原书采用 CC BY 3.0 许可。

任何操作系统运行的进程数都很可能多于计算机的 CPU 数，因此需要一套方案让这些进程分时共享 CPU。理想情况下，这种共享对用户进程是透明的。常见做法是把进程**多路复用**（multiplexing）到硬件 CPU 上，让每个进程都以为自己拥有一个独立的虚拟 CPU。本章讲解 xv6 如何实现这种多路复用。

阅读本章之前，请先阅读 [`kernel/proc.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h)、[`kernel/swtch.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S)，以及 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中的 `yield()`、`sched()` 和 `scheduler()`。

## 8.1 多路复用

xv6 在两种情况下把每个 CPU 从一个进程切换到另一个进程，以此实现多路复用。第一，当进程发起一个会阻塞（必须等待）的系统调用时，例如 `read` 或 `wait`，xv6 会进行切换。第二，xv6 会周期性地强制切换，以应对那些长时间计算而不阻塞的进程。前者称为**自愿切换**（voluntary switch），后者称为**非自愿切换**（involuntary switch）。

实现多路复用有几个难点。第一，如何从一个进程切换到另一个进程？基本思路是保存和恢复 CPU 寄存器，但这无法用 C 语言表达，所以实现起来比较棘手。第二，如何以对用户进程透明的方式强制切换？xv6 采用标准做法：由硬件定时器的中断驱动上下文切换。第三，所有 CPU 都在同一组进程之间切换，所以需要一套加锁方案，避免出现两个 CPU 同时决定运行同一个进程之类的错误。第四，进程退出时必须释放它的内存和其他资源，但进程自己无法完成全部释放工作。第五，多核机器上的每个 CPU 都必须记住自己正在执行哪个进程，这样系统调用才能作用于正确进程的内核状态。

## 8.2 上下文切换概述

“**上下文切换**”（context switch）一词指的是这样一组步骤：CPU 停止执行一个内核线程（通常以后还会恢复），转而恢复执行另一个内核线程。这种切换是多路复用的核心。xv6 并不直接从一个进程的内核线程切换到另一个进程的内核线程；而是由内核线程通过上下文切换到该 CPU 的“**调度器线程**”（scheduler thread）来让出 CPU，再由调度器线程挑选另一个进程的内核线程来运行，并通过上下文切换转到那个线程。

![图 8.1：从一个用户进程切换到另一个用户进程](https://mit-pdos.github.io/xv6-riscv-book/fig/switch.svg)

*图 8.1：从一个用户进程切换到另一个用户进程。这个例子中，xv6 运行在单个 CPU 上（因此只有一个调度器线程）。*

从更大的范围看，从一个用户进程切换到另一个用户进程所涉及的步骤如图 8.1 所示：从旧进程的用户空间陷入（系统调用或中断）到它的内核线程，上下文切换到当前 CPU 的调度器线程，上下文切换到新进程的内核线程，最后从陷阱返回到用户级进程。

## 8.3 代码：上下文切换

`kernel/swtch.S` 中的函数 `swtch()` 是线程上下文切换的核心：它保存被切出线程的 CPU 寄存器，并恢复被切入线程先前保存的寄存器。这样做之所以足够，根本原因在于线程的状态由内存中的数据（例如它的栈）加上它的 CPU 寄存器组成。线程的内存不需要保存和恢复，因为不同线程把数据放在 RAM 的不同区域；但 CPU 只有一组寄存器，所以必须在线程之间切换（保存和恢复）寄存器。

每个线程的 `struct proc` 中包含一个 `struct context`，线程不运行时，它保存的寄存器就放在这里。CPU 调度器线程的 `struct context` 位于该 CPU 的 `struct cpu` 中。当线程 X 想切换到线程 Y 时，线程 X 调用 `swtch(&X’s context, &Y’s context)`。`swtch()` 把当前 CPU 寄存器保存到 X 的 context 中，然后把 Y 的 context 内容加载到 CPU 寄存器中，然后返回。

下面是 `swtch` 的简化版本：

```
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        ...
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ...
        ld s11, 104(a1)
        
        ret
```

`a0` 存放第一个函数参数，`a1` 存放第二个；在这里就是两个 `struct context` 指针。`16(a0)` 指的是 `a0` 所指内存中偏移 16 字节处；对照 [`kernel/proc.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L2) 中 `struct context` 的定义，这就是名为 `s0` 的结构体字段。

`swtch` 的 `ret` 返回到哪里？它返回到 `ra` 寄存器所指向的指令。在线程 X 调用 `swtch()` 切换到 Y 的例子中，执行 `ret` 时，`ra` 刚刚从 Y 的 `struct context` 中加载出来。而 Y 的 `struct context` 中的 `ra`，最初是 Y 过去让出 CPU 时调用 `swtch` 保存下来的。所以 `ret` 返回到 Y 调用 `swtch()` 之后的那条指令；也就是说，X 对 `swtch()` 的调用返回时，就像是从 Y 当初对 `swtch()` 的调用中返回一样。而 `sp` 会是 Y 的栈指针，因为 `swtch` 从 Y 的 `struct context` 中加载了 `sp`；因此返回后，Y 会在它自己的栈上执行。`swtch()` 不需要直接保存或恢复程序计数器，保存和恢复 `ra` 就够了。

[`swtch`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S#L9) 保存**被调用者保存寄存器**（callee-saved register）（`ra,sp,s0..s11`），但不保存**调用者保存寄存器**（caller-saved register）。RISC-V 调用约定要求：如果代码需要在函数调用前后保留某个调用者保存寄存器的值，编译器必须生成指令，在函数调用前把该寄存器保存到栈上，并在函数返回时从栈上恢复。所以 `swtch` 可以依赖调用它的函数已经保存了调用者保存寄存器（要么如此，要么调用函数并不需要这些寄存器中的值）。

## 8.4 代码：调度

上一节考察了 `swtch` 的内部实现；现在把 `swtch` 当作已知，来看看如何从一个进程的内核线程经由调度器切换到另一个进程。调度器以每个 CPU 一个特殊线程的形式存在，每个线程都运行 `scheduler` 函数。这个函数负责选择下一个要运行的进程。每个 CPU 都有自己的调度器线程，因为在任一时刻可能有多个 CPU 在寻找可运行的东西。进程切换总是经过调度器线程，而不是直接从一个进程切换到另一个进程，这是为了避免某些情况下没有可供调度器执行的栈（例如旧进程已经退出，或者当前没有其他进程想运行）。

想让出 CPU 的进程必须获取自己的进程锁 `p->lock`，释放它持有的其他所有锁，更新自己的状态（`p->state`），然后调用 `sched`。在 [`yield`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L501)、`sleep` 和 `kexit` 中都能看到这个步骤序列。`sched` 调用 `swtch`，把当前上下文保存到 `p->context` 中，并切换到 `cpu->context` 中的调度器上下文。`swtch` 在调度器的栈上返回，就好像 `scheduler` 的 `swtch` 返回了一样[（proc.c:453）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L453)。

[`scheduler`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L429) 运行一个循环：找到一个要运行的进程，`swtch()` 到它那里，最终该进程会 `swtch()` 回到调度器，调度器继续它的循环。调度器遍历进程表，寻找可运行的进程，即 `p->state` `==` `RUNNABLE` 的进程。一旦找到，它就设置每 CPU 的当前进程变量 `c->proc`，把该进程标记为 `RUNNING`，然后调用 `swtch` 开始运行它[（proc.c:448-453）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L448-L453)。在过去的某个时刻，目标进程一定调用过 `swtch()`；调度器对 `swtch()` 的调用实际上是让那次更早的调用返回。图 8.2 展示了这种模式。

![图 8.2：swtch() 的源或目标总有一个是调度器线程](https://mit-pdos.github.io/xv6-riscv-book/fig/switch.svg)

*图 8.2：`swtch()` 的源或目标总有一个是调度器线程，并且相关的 `p->lock` 始终处于持有状态。*

xv6 在调用 `swtch` 期间一直持有 `p->lock`：`swtch` 的调用者获取锁，而锁在 `swtch` 返回后由目标线程释放。这种安排并不常见：更普遍的做法是由获取锁的线程自己释放锁。xv6 的上下文切换打破这个惯例，是因为 `p->state` 和 `p->context` 必须一起原子地更新。例如，如果在调用 `swtch` 之前就释放了 `p->lock`，另一个 CPU *c* 可能因为该进程状态为 `RUNNABLE` 而决定运行它。CPU *c* 会调用 `swtch`，从 `p->context` 恢复寄存器，而此时原来的 CPU 还在往 `p->context` 中保存寄存器。结果是该进程会在 CPU *c* 上以只保存了一部分的寄存器恢复运行，而且两个 CPU 会使用同一个栈，导致一片混乱。一旦 `yield` 开始把正在运行的进程状态改为 `RUNNABLE`，就必须一直持有 `p->lock`，直到该进程保存完所有寄存器、调度器在它自己的栈上运行为止。最早的正确释放点是在 `scheduler`（运行在自己的栈上）清除 `c->proc` 之后。同样，一旦 `scheduler` 开始把一个 `RUNNABLE` 进程转为 `RUNNING`，就必须等到该进程的内核线程完全运行起来（在 `swtch` 之后，例如在 `yield` 中）才能释放锁。

有一种情况下，调度器对 `swtch` 的调用不会落到 `sched` 中。`allocproc` 把新进程上下文中的 `ra` 寄存器设为 [`forkret`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L513)，这样它的第一次 `swtch` 就会“返回”到该函数的开头。`forkret` 的作用是释放 `p->lock`，并设置返回用户空间所需的一些控制寄存器和 trapframe 字段。最后，`forkret` 模拟系统调用返回用户空间的正常路径。

## 8.5 代码：mycpu 和 myproc

xv6 经常需要一个指向当前进程 `proc` 结构体的指针。在单处理器上，可以用一个全局变量指向当前的 `proc`。这在多核机器上行不通，因为每个 CPU 执行的是不同的进程。解决这个问题的办法是利用每个 CPU 都有自己的一组寄存器这一事实。

当某个 CPU 在内核中执行时，xv6 保证该 CPU 的 `tp` 寄存器始终存放该 CPU 的 hartid。RISC-V 为它的 CPU 编号，给每个 CPU 一个唯一的 **hartid**。[`mycpu`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L74) 用 `tp` 作为下标索引 `cpu` 结构体数组，返回当前 CPU 对应的那个结构体。[`struct cpu`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L22) 中保存着指向该 CPU 上当前运行进程（如果有的话）的 `proc` 结构体指针、该 CPU 调度器线程保存的寄存器，以及管理中断禁用所需的自旋锁嵌套计数。

保证 CPU 的 `tp` 存放该 CPU 的 hartid 有点复杂，因为用户代码可以随意修改 `tp`。`start` 在 CPU 启动序列的早期、仍处于机器模式时设置 `tp` 寄存器[（start.c:48）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c#L48)。在准备返回用户空间时，`prepare_return` 把 `tp` 保存到 trampoline 页中，以防用户代码修改它。最后，`uservec` 在从用户空间进入内核时恢复保存的 `tp`[（trampoline.S:78）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L78)。编译器保证在内核代码中绝不修改 `tp`。如果 xv6 能在需要时向 RISC-V 硬件询问当前的 hartid 会更方便，但 RISC-V 只允许在机器模式下这样做，在监管者模式下不行。

`cpuid` 和 `mycpu` 的返回值是脆弱的：如果定时器中断导致线程让出 CPU，之后又在另一个 CPU 上恢复执行，那么先前返回的值就不再正确。为了避免这个问题，xv6 要求代码在调用 `cpuid()` 或 `mycpu()` 之前禁用中断，用完返回值之后才重新开启中断。

函数 [`myproc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L83) 返回当前 CPU 上正在运行的进程的 `struct proc` 指针。`myproc` 禁用中断，调用 `mycpu`，从 `struct cpu` 中取出当前进程指针（`c->proc`），然后开启中断。即使中断处于开启状态，`myproc` 的返回值也可以安全使用：如果定时器中断把调用进程移到了另一个 CPU 上，它的 `struct proc` 指针仍然不变。

## 8.6 现实世界

xv6 调度器实现了一个简单的调度策略：依次运行每个进程。这种策略称为**轮转**（round robin）。真实的操作系统实现了更复杂的策略，例如允许进程有优先级。其思想是，调度器会优先选择可运行的高优先级进程，而不是可运行的低优先级进程。这些策略可能变得很复杂，因为常常存在相互冲突的目标：例如，操作系统可能还希望保证公平性和高吞吐量。

## 8.7 练习

1. 修改 xv6，使其在从一个进程的内核线程切换到另一个进程的内核线程时只进行一次上下文切换，而不是经由调度器线程切换。让出 CPU 的线程需要自己选择下一个线程并调用 `swtch`。难点在于：防止多个 CPU 意外执行同一个线程；把加锁做对；以及避免死锁。
