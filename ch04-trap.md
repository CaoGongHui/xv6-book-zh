# 第 4 章 陷阱与系统调用

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/trap.html>。原书采用 CC BY 3.0 许可。

有三类事件会让 CPU 暂停普通的指令执行，强制把控制权转交给处理该事件的特殊内核代码。第一种情况是系统调用：用户程序执行 `ecall` 指令，请求内核为它做某件事。第二种情况是**异常**（exception）：某条指令（用户指令或内核指令）做了非法的事，例如从无效的虚拟地址加载数据。第三种情况是设备**中断**（interrupt）：设备发出信号表示需要处理，例如磁盘硬件完成了一次读写请求。

本书用**陷阱**（trap）作为这几种情况的统称。通常，陷阱发生时正在执行的代码之后还需要恢复执行，而且不需要知道发生过什么特别的事。也就是说，我们往往希望陷阱是透明的；这对设备中断尤其重要，因为被中断的代码通常没有预料到它。陷阱会强制把控制权转入内核；内核保存寄存器和其他状态，以便之后恢复执行；内核执行相应的处理代码（例如某个系统调用的实现或设备驱动）；内核恢复保存的状态并从陷阱返回；原来的代码从中断处继续执行。

xv6 在内核中处理所有陷阱；陷阱不会交给用户代码处理。对系统调用来说，在内核中处理陷阱是很自然的。对中断来说这也合理，因为隔离要求只有内核才能使用设备，而且内核能够让多个进程共享设备。对异常来说同样合理，因为内核也许能够处理来自用户空间的异常（例子见第 [5](ch05-pgfault.md) 章），或者通过杀死出错的程序来应对。

xv6 的陷阱处理分四个阶段：RISC-V CPU 执行的硬件动作，为内核 C 代码做准备的几条汇编指令，决定如何处理该陷阱的一个 C 函数，以及系统调用或设备驱动的服务例程。虽然三类陷阱之间的共性意味着内核可以用一条代码路径处理所有陷阱，但实践证明，为两种不同情况分别编写代码更方便：来自用户空间的陷阱，和来自内核空间的陷阱。处理陷阱的内核代码（汇编或 C）通常称为**处理程序**（handler）；处理程序最开始的那些指令通常用汇编（而不是 C）编写，有时称为**向量**（vector）。

继续阅读之前，请先读 [`kernel/trampoline.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S)，以及 [`kernel/trap.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c) 中的 `usertrap()` 和 `prepare_return()`。

## 4.1 RISC-V 陷阱机制

每个 RISC-V CPU 都有一组硬件控制寄存器。内核写这些寄存器来告诉 CPU 如何处理陷阱，也可以读它们来了解已经发生的陷阱。完整的说明见 RISC-V 文档（[Waterman et al. 2024](https://mit-pdos.github.io/xv6-riscv-book/references.html#ref-riscv:priv)）。[`riscv.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h) 包含 xv6 用到的定义。下面概述最重要的几个寄存器：

- `stvec`：内核把陷阱处理代码的地址写在这里；RISC-V 跳转到 `stvec` 中的地址来处理陷阱。

- `sepc`：陷阱发生时，RISC-V 把程序计数器保存在这里（因为随后 `pc` 会被 `stvec` 的值覆盖）。`sret`（从陷阱返回）指令把 `sepc` 复制到 `pc`。内核可以写 `sepc` 来控制 `sret` 返回到哪里。

- `scause`：RISC-V 在这里放一个数字，描述陷阱的原因。

- `sscratch`：内核陷阱处理代码借助 `sscratch`，避免在保存用户寄存器之前就把它们覆盖掉。

- `sstatus`：`sstatus` 中的 SIE 位控制是否启用设备中断。如果内核清除 SIE，RISC-V 会推迟设备中断，直到内核重新设置 SIE。SPP 位表示陷阱来自用户模式还是监管者模式，并控制 `sret` 返回到哪种模式。

上述寄存器只能在监管者模式下（也就是由内核）访问；CPU 不允许用户代码读写它们。

多核芯片上的每个 CPU 都有自己的一组这些寄存器，任一时刻可能有不止一个 CPU 在处理陷阱。

强制进入陷阱时，RISC-V 硬件会执行以下操作：

1. 如果陷阱是设备中断，并且 `sstatus` 的 SIE 位是清除状态，则不执行下面任何步骤。

2. 清除 `sstatus` 中的 SIE 位，以禁用中断。

3. 把 `pc` 复制到 `sepc`。

4. 把当前模式（用户或监管者）保存到 `sstatus` 的 SPP 位。

5. 把 `scause` 设置为表示陷阱原因的数字。

6. 把模式设置为监管者模式。

7. 把 `stvec` 复制到 `pc`。

8. 从新的 `pc` 处开始执行。

CPU 不会切换到内核页表，不会切换到内核中的栈，除了 `pc` 之外也不保存任何寄存器。这些工作必须由内核软件完成。CPU 在陷阱期间只做最少工作的一个原因是给软件留出灵活性；例如，有些操作系统在某些情况下省略页表切换，以提高陷阱的性能。

值得思考一下，上面列出的步骤中有没有可以省略的，也许能让陷阱更快。虽然有些情况下更简单的步骤序列也能工作，但一般来说，省略其中许多步骤是危险的。例如，假设 CPU 不切换程序计数器。那么来自用户空间的陷阱可能会切换到监管者模式，却仍在执行用户指令。这些用户指令可能破坏用户/内核隔离，例如修改 `satp` 寄存器，让它指向一个可以访问全部物理内存的页表。因此，CPU 切换到由内核指定的指令地址（也就是 `stvec`）是很重要的。

## 4.2 来自用户空间的陷阱

![图 4.1：来自用户代码的陷阱的处理流程概览](https://mit-pdos.github.io/xv6-riscv-book/fig/trap.svg)

*图 4.1：来自用户代码的陷阱的处理流程概览。*

xv6 根据陷阱发生在内核中还是用户代码中，采用不同的处理方式。这里讲来自用户代码的陷阱；第 4.5 节介绍来自内核代码的陷阱。

在用户空间执行时，如果用户程序发起系统调用（`ecall` 指令），或做了非法的事，或者有设备发出中断，都可能发生陷阱。如图 4.1 所示，来自用户空间的陷阱的大致路径是先 [`uservec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22)，再 [`usertrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L38)；当内核准备好返回时，`usertrap` 返回到 [`userret`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L101)，由它执行 `sret` 回到用户空间。

xv6 陷阱处理设计上的一个主要约束是：RISC-V 硬件在强制进入陷阱时不会切换页表。这意味着 `stvec` 中的陷阱处理程序地址必须在用户页表中有有效映射，因为陷阱处理代码开始执行时生效的就是用户页表。此外，xv6 的陷阱处理代码需要切换到内核页表；为了在切换之后还能继续执行，内核页表中也必须有 `stvec` 所指处理程序的映射。

xv6 用一个**蹦床**（trampoline）页来满足这些要求。这个页包含 `uservec`，也就是 `stvec` 指向的 xv6 陷阱处理代码。蹦床页映射在每个进程页表的虚拟地址 `0x3ffffff000`（称为 `TRAMPOLINE`）处，这是虚拟地址空间的最后一页，因此位于程序自己使用的内存之上。蹦床页在内核页表中也映射在同一个虚拟地址。参见图 2.3 和图 3.3。因为蹦床页映射在用户页表中，陷阱可以在监管者模式下从那里开始执行。因为蹦床页在内核地址空间中映射在同一地址，陷阱处理程序切换到内核页表之后可以继续执行。

`uservec` 陷阱处理程序的代码在 [`trampoline.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22) 中。`uservec` 开始执行时，全部 32 个寄存器中都是被中断的用户代码的值。这 32 个值需要保存到内存中的某个位置，以便之后内核在返回用户空间之前恢复它们。写内存需要用一个寄存器存放目标地址，但此时没有可用的通用寄存器！幸好 RISC-V 提供了 `sscratch` 寄存器来帮忙。`uservec` 开头的 `csrw` 指令把 `a0` 保存到 `sscratch`。这样 `uservec` 就有一个寄存器（`a0`）可以用了。

`uservec` 的下一个任务是保存 32 个用户寄存器。内核为每个进程分配一页内存，用来存放 `trapframe` 结构，其中（除了其他内容外）有保存 32 个用户寄存器的空间[（proc.h:40）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L40)。因为 `satp` 仍然指向用户页表，`uservec` 需要 trapframe 映射在用户地址空间中。xv6 把每个进程的 trapframe 映射在该进程用户页表的虚拟地址 `TRAPFRAME`（`0x3fffffe000`）处，也就是 `TRAMPOLINE` 下面一页。每个进程的 `p->trapframe` 存放的是该进程 trapframe 的内核虚拟地址。

`uservec` 把寄存器 `a0` 设置为地址 `TRAPFRAME`，把所有用户寄存器保存到那里。然后它从 `sscratch` 中取回用户的 `a0`，也保存到 trapframe 中。

内核之前已经在 trapframe 中初始化了一些对 `uservec` 有用的值：当前进程内核栈的地址、当前 CPU 的 hartid、`usertrap` 函数的地址，以及内核页表的地址。`uservec` 取出这些值，把 `satp` 切换到内核页表，然后跳转到 C 函数 `usertrap`。

`usertrap` 的任务是判断陷阱的原因，处理它，然后返回[（trap.c:38）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L38)。它首先修改 `stvec`，使得在内核中发生的陷阱由 `kernelvec` 而不是 `uservec` 处理。它保存 `sepc` 寄存器（保存下来的用户程序计数器），供之后返回用户空间时使用。如果陷阱是系统调用，`usertrap` 调用 `syscall` 处理；如果是设备中断，调用 `devintr`；如果是缺页异常，调用 `vmfault`；否则就是一个异常（例如使用了无效地址），内核会杀死出错的进程。系统调用路径会把保存的用户程序计数器加 4，因为对于系统调用，RISC-V 让程序指针仍指向 `ecall` 指令，而用户代码需要从下一条指令继续执行。`usertrap` 会检查进程是否已被杀死，或者是否应该让出 CPU（如果这次陷阱是时钟中断）。

返回用户空间的第一步是调用 [`prepare_return`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L101)。这个函数设置 RISC-V 控制寄存器，为将来来自用户空间的陷阱做准备：把 `stvec` 设置为 `uservec`，并准备好 `uservec` 依赖的 trapframe 字段。`prepare_return` 把 `sepc` 设置为之前保存的用户程序计数器。最后，`usertrap` 返回到蹦床页中的 `userret`[（trampoline.S:101）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L101)，并通过 `a0` 传回指向用户页表的指针。

`userret` 把 `satp` 切换到进程的用户页表。回忆一下，用户页表映射了蹦床页和 `TRAPFRAME`，但没有映射内核的其他任何东西。蹦床页在用户页表和内核页表中映射在同一个虚拟地址，这使得 `userret` 在修改 `satp` 之后还能继续执行。从这时起，`userret` 能使用的数据只有寄存器内容和 trapframe 的内容。`userret` 把 `TRAPFRAME` 地址加载到 `a0`，通过 `a0` 从 trapframe 中恢复保存的用户寄存器，恢复保存的用户 `a0`，然后执行 `sret` 返回用户空间。

`uservec` 和 `userret` 用汇编语言编写，因为用 C 代码很难保存或恢复所有寄存器，也很难在切换页表后继续正常运行。

## 4.3 代码：发起系统调用

用户程序通过调用库函数来发起系统调用。例如，shell 用下面这个函数调用显示提示符（在 `user/sh.c` 中）：

```c
  write(2, "$ ", 2);
```

下面是对应的库函数，在 `user/usys.S` 中：

```
write:
 li a7, SYS_write
 ecall
 ret
```

C 编译器为这个函数调用生成的代码把三个参数加载到寄存器 `a0`、`a1` 和 `a2` 中。然后 `write()` 函数把系统调用号 `SYS_write`（16）加载到 `a7`。内核会查看这些寄存器，找出要执行哪个系统调用以及参数是什么。`ecall` 指令从用户空间陷入内核，依次执行 `uservec`、`usertrap`，然后是 `syscall`。

此时请阅读 [`kernel/syscall.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c)、[`kernel/sysfile.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c) 中的 `sys_write()`，以及 [`kernel/vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c) 中的 `copyout()`、`copyin()` 和 `copyinstr()`。

[`syscall`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L137) 从 trapframe 中保存的 `a7` 取出系统调用号，用它作为下标查找 [`syscalls`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L109) 表。在我们的例子中，`a7` 的值是 [`SYS_write`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.h#L17)，于是调用系统调用的实现函数 `sys_write`。

`sys_write` 返回后，`syscall` 把它的返回值记录在 `p->trapframe->a0` 中。这样原来用户空间对 `write()` 的调用就会返回这个值，因为 RISC-V 上的 C 调用约定把返回值放在 `a0` 中。按照惯例，系统调用返回负数表示出错，返回零或正数表示成功。

## 4.4 代码：系统调用参数

系统调用参数最初在用户寄存器中，随后由内核陷阱代码移到 trapframe 中。内核函数 `argint`、`argaddr` 和 `argfd` 从 trapframe 中取出第 *n* 个系统调用参数，分别作为整数、指针或文件描述符。

有些系统调用以指针作为参数，内核必须用这些指针来读写用户内存。例如，`write` 系统调用向内核传入一个用户空间指针，指向要写入的数据。这类指针带来两个难题。第一，用户程序可能有 bug 或是恶意的，可能向内核传入无效指针，或者传入一个意在诱骗内核访问内核内存而非用户内存的指针。第二，xv6 内核页表的映射与用户页表的映射不同，所以内核不能用普通指令从用户提供的地址加载或向其存储数据。

内核实现了一些函数，可以安全地在用户提供的地址和内核之间传输数据。`fetchstr` 就是一个例子[（syscall.c:26）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L26)。`exec` 等文件系统调用用 `fetchstr` 从用户空间取出字符串形式的文件名参数。`fetchstr` 调用 `copyinstr` 完成实际工作。

[`copyinstr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L412) 从用户页表 `pagetable` 中的虚拟地址 `srcva` 最多复制 `max` 个字节到 `dst`。由于 `pagetable` **不是**当前页表，`copyinstr` 用 `walkaddr`（它会调用 `walk`）在 `pagetable` 中查找 `srcva`，得到物理地址 `pa0`。内核页表把全部物理 RAM 映射在与其物理地址相等的虚拟地址上。因此 `copyinstr` 可以直接从 `pa0` 把字符串字节复制到 `dst`。[`walkaddr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L122) 会检查用户提供的虚拟地址是否属于进程的用户地址空间，这样程序就无法诱骗内核读取其他内存。类似的函数 `copyout` 把数据从内核复制到用户提供的地址。

## 4.5 来自内核空间的陷阱

请阅读 [`kernel/kernelvec.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S)，以及 [`kernel/trap.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c) 中的 `kerneltrap()`。

xv6 处理来自内核代码的陷阱的方式与处理来自用户代码的陷阱不同。进入内核时，`usertrap` 把 `stvec` 指向 [`kernelvec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S#L12) 处的汇编代码。由于 `kernelvec` 只在 xv6 已经处于内核中时才会执行，它可以依赖 `satp` 已设置为内核页表，并且栈指针指向一个有效的内核栈。`kernelvec` 把所有调用者保存（caller-saved）寄存器压入当前栈，之后再从栈中恢复它们，使被中断的内核代码可以不受干扰地继续执行。

`kernelvec` 把寄存器保存在被中断内核线程的栈上，这是合理的，因为这些寄存器值属于那个线程。如果陷阱导致切换到另一个线程，这一点尤其重要：在这种情况下，陷阱实际上会从新线程的栈返回，而被中断线程保存的寄存器则安全地留在它自己的栈上。

`kernelvec` 保存寄存器后跳转到 [`kerneltrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L137)。`kerneltrap` 只准备处理一种陷阱：设备中断。它调用 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188) 来处理。如果陷阱不是设备中断，那它一定是异常，例如内核代码试图使用无效指针。这只可能是内核代码的 bug 造成的。内核在这种情况下没有办法恢复，所以它调用 `panic()`，打印错误信息后停机。

如果 `kerneltrap` 是因时钟中断被调用的，并且正在运行的是某个进程的内核线程（而不是调度器线程），`kerneltrap` 会调用 `yield`，让其他线程有机会运行。之后某个时刻，其中某个线程会让出 CPU，让我们的线程及其 `kerneltrap` 继续执行。第 [8](ch08-sched.md) 章解释 `yield` 中发生了什么。

`kerneltrap` 的工作完成后，需要返回到被陷阱打断的代码。由于 `yield` 可能已经改动了 `sepc` 和 `sstatus` 中的先前模式，`kerneltrap` 在开始时会保存它们。此时它恢复这些控制寄存器，然后返回到 [`kernelvec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S#L38)。`kernelvec` 从栈中弹出保存的寄存器，执行 `sret`，`sret` 把 `sepc` 复制到 `pc`，恢复被中断的内核代码。

当某个 CPU 从用户空间进入内核时，xv6 把该 CPU 的 `stvec` 设置为 `kernelvec`；这一点可以在 [`usertrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L47) 中看到。但存在一个时间窗口：内核已经开始执行，而 `stvec` 仍然设置为 `uservec`。在这个窗口内绝不能发生设备中断。幸好 RISC-V 在开始进入陷阱时总会禁用中断，而 `usertrap` 直到设置好 `stvec` 之后才会重新启用中断。

## 4.6 现实世界

如果把内核内存映射到每个进程的用户页表中（并清除 `PTE_U`），就不再需要蹦床页。这样从用户空间陷入内核时也不必切换页表。这又使内核中的系统调用实现可以利用当前进程用户内存已被映射这一点，让内核代码直接解引用用户指针。许多操作系统采用了这些思路来提高效率。xv6 没有这样做，是为了减少内核因无意中使用用户指针而产生安全漏洞的可能，也为了省去确保用户虚拟地址和内核虚拟地址不重叠所需的一些复杂性。

## 4.7 练习

1. `trampoline.S` 和 `kernelvec.S` 中的部分或全部代码能否用 C 而不是汇编编写？

2. 有没有办法去掉每个用户地址空间中特殊的 `TRAPFRAME` 页映射？例如，能否修改 `uservec`，让它直接把 32 个用户寄存器压入内核栈，或者保存在 `proc` 结构中？

3. 能否修改 xv6，去掉特殊的 `TRAMPOLINE` 页映射？