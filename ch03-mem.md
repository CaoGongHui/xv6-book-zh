# 第 3 章 页表

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/mem.html>。原书采用 CC BY 3.0 许可。

页表是操作系统为每个进程提供私有地址空间和内存最常用的机制。页表决定了内存地址的含义，以及物理内存的哪些部分可以访问。有了页表，xv6 就能隔离不同进程的地址空间，并把它们多路复用到同一块物理内存上。页表提供了一层间接，操作系统可以借此玩出很多有用的花样。xv6 用到了其中几种：在多个地址空间中映射同一块内存（trampoline 页），用一个未映射的页保护内核栈和用户栈，以及延迟分配用户堆内存。本章其余部分讲解 RISC-V 硬件提供的页表，以及 xv6 如何使用它们。

## 3.1 分页硬件

回顾一下，RISC-V 指令（无论用户指令还是内核指令）操作的都是虚拟地址。机器的 RAM，也就是物理内存，用物理地址来索引。RISC-V 的页表硬件把每个虚拟地址映射到一个物理地址，从而把这两种地址联系起来。

![图 3.1：把虚拟地址映射到物理地址的扁平页表的抽象视图](https://mit-pdos.github.io/xv6-riscv-book/fig/riscv_address.svg)

*图 3.1：把虚拟地址映射到物理地址的扁平页表的抽象视图。*

xv6 使用 RISC-V 的 Sv39 模式，即 64 位虚拟地址中只用低 39 位，高 25 位不用。在 Sv39 配置下，RISC-V 页表在逻辑上是一个由 2²⁷（134,217,728）个**页表项**（page table entry，PTE）组成的数组。每个 PTE 包含一个 44 位的物理页号（physical page number，PPN）和一些标志位。分页硬件翻译虚拟地址时，用 39 位中的高 27 位去索引页表，找到一个 PTE，然后构造一个 56 位的物理地址：高 44 位来自 PTE 中的 PPN，低 12 位从原虚拟地址复制过来。图 3.1 展示了这个过程，图中把页表在逻辑上看成一个简单的 PTE 数组（RISC-V 页表实际上是一棵树，完整情况见图 3.2）。页表让操作系统能以 4096（2¹²）字节的对齐块为粒度控制虚拟地址到物理地址的翻译。这样的块称为一个**页**（page）。

![图 3.2：RISC-V 地址翻译细节](https://mit-pdos.github.io/xv6-riscv-book/fig/riscv_pagetable.svg)

*图 3.2：RISC-V 地址翻译细节。*

RISC-V 的设计给虚拟地址和物理地址都留了扩展的余地。如果需要更大的虚拟地址空间，RISC-V 支持 Sv48 模式，使用 48 位虚拟地址（Waterman et al. 2024）。物理地址也有增长空间：PTE 格式中还留有位置，物理页号可以再增加 10 位。RISC-V 的设计者根据技术预测选择了这些地址长度。2⁴⁸ 字节是 262,144 GB，远大于如今任何应用可能用到的用户虚拟地址空间。2⁵⁶ 字节的物理地址空间是 65,536 TB，远多于目前任何计算机能装配的 RAM。

如图 3.2 所示，RISC-V CPU 的页表以三级树的形式存放在物理内存中。树根是一个 4096 字节的页表页，包含 512 个 PTE，这些 PTE 存放树中下一级页表页的物理地址。下一级的每个页又包含 512 个 PTE，指向树的最后一级。分页硬件用 27 位中的高 9 位在根页表页中选出一个 PTE，用中间 9 位在下一级的页表页中选出一个 PTE，再用低 9 位选出最终的 PTE。（在 Sv48 的 RISC-V 中，页表有四级，虚拟地址的第 39 到 47 位用于索引最顶层。）

如果翻译一个地址所需的三个 PTE 中有任何一个不存在，分页硬件就会引发一个**缺页异常**（page-fault exception），交给内核去处理这个缺页（见第 4 章和第 5 章）。

与图 3.1 的单级设计相比，图 3.2 的三级结构记录 PTE 的方式更节省内存。在大段虚拟地址都没有映射这种常见情况下，三级结构可以整个省掉一些页目录。例如，如果一个应用只用了从地址 0 开始的几个页，那么顶层页目录的第 1 到 511 项都是无效的，内核不必为这 511 个中间页目录分配页。而且，内核也不必为这 511 个中间页目录下面的底层页目录分配页。所以在这个例子中，三级设计省下了 511 个中间页目录页和 511×512 个底层页目录页。

虽然 CPU 在执行 load 或 store 指令时由硬件遍历这个三级结构，但三级结构有一个潜在的缺点：CPU 必须从内存中加载三个 PTE，才能把 load/store 指令中的虚拟地址翻译成物理地址。为了避免从物理内存加载 PTE 的开销，RISC-V CPU 会把页表项缓存在**快表**（Translation Look-aside Buffer，TLB）中。

每个 PTE 都包含一些标志位，告诉分页硬件相应的虚拟地址允许怎样使用。`PTE_V` 表示该 PTE 是否存在：如果没有设置，对该页的访问会引发缺页（即不允许访问）。`PTE_R` 控制指令是否可以读该页。`PTE_W` 控制指令是否可以写该页。`PTE_X` 控制 CPU 是否可以把该页的内容当作指令来执行。`PTE_U` 控制用户模式下的指令是否可以访问该页；如果没有设置 `PTE_U`，这个 PTE 只能在 supervisor 模式下使用。图 3.2 展示了这些标志位在 PTE 中的位置。这些标志以及所有其他与分页硬件相关的结构都定义在[（riscv.h）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h) 中。

要让 CPU 使用某个页表，内核必须把根页表页的物理地址写入 `satp` 寄存器。CPU 会用其 `satp` 指向的页表翻译之后所有指令产生的地址。每个 CPU 都有自己的 `satp`，这样不同的 CPU 可以运行不同的进程，每个进程都有由自己的页表描述的私有地址空间。

从内核的角度看，页表就是存放在内存中的数据，内核创建和修改页表的代码，与你在任何树形数据结构中可能看到的代码差不多。

关于本书所用术语的几点说明。**物理内存**（physical memory）指 RAM 中的存储单元。物理内存的每个字节都有一个地址，称为**物理地址**（physical address）。对地址解引用的指令（例如 load、store、跳转和函数调用）只使用虚拟地址，分页硬件把它们翻译成物理地址，再发给 RAM 硬件去读写存储。**地址空间**（address space）是在某个页表中有效的虚拟地址的集合；每个 xv6 进程都有一个独立的用户地址空间，xv6 内核也有自己的地址空间。**用户内存**（user memory）指进程的用户地址空间，加上页表允许该进程访问的物理内存。**虚拟内存**（virtual memory）指与管理页表、利用页表实现隔离等目标相关的思想和技术。

![图 3.3：左边是 xv6 的内核虚拟地址空间，右边是 xv6 期望看到的 RISC-V 物理地址空间](https://mit-pdos.github.io/xv6-riscv-book/fig/xv6_layout.svg)

*图 3.3：左边是 xv6 的内核虚拟地址空间。RWX 指 PTE 的读、写和执行权限。右边是 xv6 期望看到的 RISC-V 物理地址空间。*

## 3.2 内核地址空间

xv6 启动时会创建一个页表，描述内核的地址空间。内核设置自己地址空间的布局，使自己能在可预知的虚拟地址上访问物理内存和各种硬件资源。图 3.3 展示了这个布局如何把内核虚拟地址映射到物理地址。文件 [`kernel/memlayout.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h) 声明了 xv6 内核内存布局的各个常量。

QEMU 模拟的计算机包含从物理地址 `0x80000000` 开始、至少延续到 `0x88000000` 的 RAM（物理内存），xv6 把后者称为 `PHYSTOP`。QEMU 模拟的机器还包含磁盘接口等 I/O 设备。QEMU 把设备接口以**内存映射**（memory-mapped）控制寄存器的形式暴露给软件，这些寄存器位于物理地址空间中 `0x80000000` 以下。内核通过读写这些特殊的物理地址与设备交互；这些读写是和设备硬件通信，而不是访问 RAM。第 4 章会讲解 xv6 如何与设备交互。

内核把所有物理 RAM 和设备寄存器都映射到与物理地址相等的虚拟地址上。这称为“直接映射”（direct mapping），它让内核只需 load 或 store 虚拟地址 x，就能读写物理地址 x。内核代码本身在虚拟地址空间和物理内存中都位于 `KERNBASE=0x80000000`。[`kfork`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L259) 为子进程分配用户内存时，分配器返回的是那块内存的物理地址；`kfork` 把父进程的用户内存复制给子进程时，直接把这个地址当作虚拟地址使用。

有几个内核虚拟地址不是直接映射的：

- trampoline 页。它被映射在虚拟地址空间的顶端；用户页表也有同样的映射。第 4 章会讨论 trampoline 页的作用，不过这里已经能看到页表的一个有趣用法：一个物理页（存放 trampoline 代码）在内核虚拟地址空间中被映射了两次，一次在虚拟地址空间顶端，一次是直接映射。

- 内核栈页。每个进程都有自己的内核栈，它被映射在内核的高虚拟地址处，这样 xv6 可以在它下面留一个未映射的**保护页**（guard page）。保护页的 PTE 是无效的（即没有设置 `PTE_V`），因此如果内核栈溢出，很可能会引发缺页，内核随即 panic。没有保护页的话，溢出的栈会覆盖内核的其他内存，导致错误的行为。相比之下，panic 崩溃更可取。

虽然内核通过高地址映射使用这些栈，但内核也能通过直接映射的地址访问它们。另一种设计是只用直接映射，在直接映射的地址上使用栈。但那样的话，要设置保护页，就得取消一些原本指向物理内存的虚拟地址的映射，而这些物理内存之后就很难再利用了。

内核用 `PTE_R` 和 `PTE_X` 权限映射 trampoline 页和内核代码（text）页，不加 `PTE_W`。其他页用 `PTE_R` 和 `PTE_W` 权限映射，不加 `PTE_X`。保护页的映射是无效的。限制这些权限的目的是帮助发现以意外方式访问页面的内核 bug，例如内核代码不小心想覆写内核指令。

内核只创建一个内核页表，所有 CPU 在内核中执行时都使用它。xv6 在最初创建内核页表之后就不再修改它。

## 3.3 代码：创建地址空间

继续之前，请先阅读 [`kernel/vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c)，一直读到 `mappages()` 的末尾。

xv6 中操作地址空间和页表的代码大部分在 [`vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c) 中。核心数据结构是 `pagetable_t`，它实际上是指向 RISC-V 根页表页的指针；一个 `pagetable_t` 可以是内核页表，也可以是某个进程的页表。核心函数是 `walk` 和 `mappages`：`walk` 为一个虚拟地址找到对应的 PTE，`mappages` 为新的映射填写 PTE。以 `kvm` 开头的函数操作内核页表；以 `uvm` 开头的函数操作用户页表；其他函数两者都用。`copyout` 和 `copyin` 在作为系统调用参数提供的用户虚拟地址与内核之间复制数据；它们放在 `vm.c` 中，是因为它们需要显式地翻译这些地址，才能找到对应的物理内存。

在启动过程的早期，`main` 调用 [`kvminit`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L67)，后者用 [`kvmmake`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L22) 创建内核页表。这个调用发生在 xv6 启用 RISC-V 分页之前，所以地址直接指向物理内存。`kvmmake` 首先分配一页物理内存来存放根页表页，然后调用 `kvmmap` 填入内核需要的翻译。这些翻译包括内核的指令和数据、直到 `PHYSTOP` 的物理内存，以及实际上是设备的内存区间。[`proc_mapstacks`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L33) 为每个进程分配一个内核栈。它调用 `kvmmap`，把每个栈映射到由 `KSTACK` 生成的虚拟地址上，`KSTACK` 为无效的栈保护页留出了空间。

[`kvmmap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L59) 调用 [`mappages`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L147)，后者在页表中为一段虚拟地址区间建立到相应物理地址区间的映射。它对区间内的每个虚拟地址按页间隔逐一处理。对每个要映射的虚拟地址，`mappages` 调用 `walk` 找到该地址对应 PTE 的地址，然后初始化这个 PTE，填入相应的物理页号、所需的权限（`PTE_W`、`PTE_X` 和/或 `PTE_R`），以及表示 PTE 有效的 `PTE_V`[（vm.c:168）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L168)。

[`walk`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L99) 模仿 RISC-V 分页硬件为虚拟地址查找 PTE 的过程（见图 3.2）。`walk` 一次下降一级页表树，用每一级对应的 9 位虚拟地址去索引相应的页目录页。在每一级，它找到的要么是下一级页目录页的 PTE，要么是最终页的 PTE[（vm.c:105）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L105)。如果第一级或第二级页目录页中的某个 PTE 无效，说明所需的目录页还没有分配；如果设置了 `alloc` 参数，`walk` 就分配一个新的页表页，并把它的物理地址放进这个 PTE。`walk` 返回树中最底层 PTE 的地址[（vm.c:115）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L115)。

上面的代码依赖于物理内存被直接映射到内核虚拟地址空间。例如，`walk` 逐级下降时，从 PTE 中取出下一级页表的（物理）地址[（vm.c:107）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L107)，然后把这个地址当作虚拟地址，去取下一级的 PTE[（vm.c:105）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L105)。

在每个 CPU 上，`main` 调用 [`kvminithart`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L75) 安装内核页表，把根页表页的物理地址放进该 CPU 的 `satp` 寄存器。此后 CPU 就用内核页表翻译地址。由于内核页表是直接映射的，地址在这次切换前后指向 RAM 中的同一位置，所以内核能继续正确执行。

每个 RISC-V CPU 都把页表项缓存在**快表**（Translation Look-aside Buffer，TLB）中，xv6 修改页表时，必须通知 CPU 让相应的 TLB 缓存项失效。否则，之后某个时刻 TLB 可能会用到一个旧的缓存映射，它指向的物理页在此期间可能已经分配给了另一个进程，结果一个进程就可能乱写另一个进程的内存。RISC-V 有一条 `sfence.vma` 指令，用于刷新当前 CPU 的 TLB。xv6 在 `kvminithart` 中重新加载 `satp` 寄存器之后执行 `sfence.vma`，在 `uservec` 和 `userret` 的 trampoline 代码中也会执行它。

在修改 `satp` 之前也需要执行 `sfence.vma`，以等待所有未完成的 load 和 store 完成。这次等待确保之前对页表的更新已经完成，并确保之前的 load 和 store 使用的是旧页表而不是新页表。

## 3.4 物理内存分配

内核必须在运行时为页表、用户内存、内核栈和管道缓冲区分配和释放物理内存。

xv6 把内核末尾到 `PHYSTOP` 之间的物理内存用于运行时分配。它每次分配和释放整个 4096 字节的页。它把空闲页本身串成一个链表，以此记录哪些页是空闲的。分配就是从链表中取下一个页；释放就是把释放的页加入链表。

请阅读 [`kernel/kalloc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c)。

## 3.5 代码：物理内存分配器

分配器位于 [`kalloc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c) 中。分配器的数据结构是一个由可供分配的物理内存页组成的**空闲链表**（free list）。每个空闲页的“next”指针存放在一个 [`struct run`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c#L17) 中。分配器把每个空闲页的 `run` 结构存放在该空闲页自身里面，因为页空闲时那里没有存放别的东西。空闲链表由一个自旋锁保护[（kalloc.c:21-24）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c#L21-L24)。链表和锁被包在一个结构体里，以表明这个锁保护的是该结构体中的字段。现在先忽略锁以及对 `acquire` 和 `release` 的调用；第 7 章会详细讨论锁。

函数 `main` 调用 `kinit` 初始化分配器[（kalloc.c:27）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c#L27)。`kinit` 把空闲链表初始化为包含内核末尾到 `PHYSTOP` 之间的每一页物理 RAM。xv6 本应通过解析硬件提供的配置信息来确定可用的物理内存有多少，但它直接假设机器有 128 MB 的 RAM。`kinit` 调用 `freerange`，后者对每一页调用 `kfree`，把内存加入空闲链表。PTE 只能指向按 4096 字节边界对齐（即 4096 的倍数）的物理地址，所以 `freerange` 用 `PGROUNDUP` 确保只释放对齐的物理地址。分配器一开始没有任何内存；这些对 `kfree` 的调用给了它一些可以管理的内存。

分配器有时把地址当作整数，以便对它们做算术运算（例如在 `freerange` 中遍历所有页），有时又把地址当作指针来读写内存（例如操作存放在每个页中的 `run` 结构）；地址的这种双重用途，是分配器代码中充满 C 类型转换的主要原因。

函数 [`kfree`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c#L47) 首先把要释放的内存的每个字节都设为 1。这样，释放后还在使用这块内存的代码（使用“悬空引用”）读到的就是垃圾数据，而不是原来的有效内容；希望这能让这类代码更快地出错。然后 `kfree` 把这一页插到空闲链表的头部：它把 `pa` 转换成指向 `struct` `run` 的指针，把空闲链表原来的头记录在 `r->next` 中，再把空闲链表设为 `r`。`kalloc` 取下并返回空闲链表中的第一个元素。

## 3.6 进程地址空间

每个进程都有自己的页表，xv6 在进程之间切换时也会切换页表。图 3.4 比图 2.3 更详细地展示了一个进程的地址空间。进程的用户地址空间从 0 开始，原则上一直到 `MAXVA`（`0x4000000000`）结束[（riscv.h:417）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h#L417)，不过实际上只有很小一部分映射到了物理内存。

进程的地址空间由以下几部分组成：存放程序代码（text）的页（xv6 用 `PTE_R`、`PTE_X` 和 `PTE_U` 权限映射），存放程序预初始化数据的页，一个栈页，以及若干堆页。xv6 用 `PTE_R`、`PTE_W` 和 `PTE_U` 权限映射数据、栈和堆。

在用户地址空间中使用权限，是加固用户进程的常用技术。如果代码段用 `PTE_W` 映射，进程就可能不小心修改自己的程序；例如，一个编程错误可能导致程序向空指针写数据，修改了地址 0 处的指令，然后程序继续运行，也许会造成更大的混乱。为了立即发现这类错误，xv6 映射代码段时不加 `PTE_W`；如果程序不小心试图向地址 0 写数据，硬件会拒绝执行这次 store 并引发缺页（见第 4 章）。随后内核会杀死该进程，并打印一条有用的信息，方便开发者追查问题。

同样，映射数据时不加 `PTE_X`，用户程序就不会不小心跳转到程序数据中的某个地址并从那里开始执行。

在现实世界中，通过仔细设置权限来加固进程，还有助于抵御安全攻击。攻击者可能会给某个程序（例如 Web 服务器）喂入精心构造的输入，触发程序中的 bug，希望把这个 bug 变成一次漏洞利用（One, n.d.）。仔细设置权限，加上其他技术（例如随机化用户地址空间的布局），能让这类攻击更难得逞。

栈只有一页，图中展示的是由 `exec` 系统调用创建的初始内容。包含命令行参数的字符串，以及指向它们的指针数组，位于栈的最顶端。紧挨着它们下方的是一些值，让程序能从 `main` 开始执行，就好像刚刚调用了函数 `main(argc`, `argv)` 一样。

为了检测用户栈溢出已分配的栈内存，xv6 在栈的正下方放置了一个不可访问的保护页，做法是清除它的 `PTE_U` 标志。如果用户栈溢出，进程试图使用栈下方的地址，硬件就会产生缺页异常，因为保护页对用户模式下运行的程序不可访问。现实中的操作系统也可能在用户栈溢出时自动为它分配更多内存。

这里可以看到页表的几个很好的用例。第一，不同进程的页表把用户地址翻译到物理内存的不同页上，因此每个进程都有私有的用户内存。第二，每个进程看到的内存是从 0 开始的连续虚拟地址，而进程的物理内存可以是不连续的。第三，内核在用户地址空间的顶端映射了一个存放 trampoline 代码的页（不加 `PTE_U`），于是同一个物理页出现在所有地址空间中，但只有内核能使用它。

![图 3.4：一个进程的用户地址空间及其初始栈](https://mit-pdos.github.io/xv6-riscv-book/fig/processlayout.svg)

*图 3.4：一个进程的用户地址空间及其初始栈。*

## 3.7 代码：exec

请阅读 [`kernel/exec.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c)，以及 [`kernel/vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c) 中从 `uvmcreate()` 开始的部分。

`exec` 是一个系统调用，它用从文件中读出的数据替换进程的用户地址空间，这个文件称为二进制文件或可执行文件。二进制文件通常是编译器和链接器的输出，包含机器指令和程序数据。[`kexec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L28) 是 `exec` 在内核中的内部实现，它用 [`namei`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L42) 打开名为 `path` 的二进制文件，`namei` 会在第 10 章讲解。然后它读取 ELF 头。xv6 的二进制文件采用广泛使用的 **ELF 格式**（ELF format），定义在[（elf.h）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/elf.h) 中。一个 ELF 二进制文件由一个 ELF 头 [`struct elfhdr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/elf.h#L6) 和其后的一系列程序段头 [`struct proghdr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/elf.h#L25) 组成。每个 `proghdr` 描述应用中必须加载到内存的一个段；xv6 程序有两个程序段头：一个用于指令，一个用于数据。

第一步是快速检查该文件是否可能包含一个 ELF 二进制。ELF 二进制以四字节的“魔数”`0x7F`、`'E'`、`'L'`、`'F'` 开头，即 [`ELF_MAGIC`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/elf.h#L3)。如果 ELF 头的魔数正确，`kexec` 就假定这个二进制文件格式良好。

`kexec` 用 [`proc_pagetable`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L56) 分配一个没有用户映射的新页表，用 [`uvmalloc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L72) 为每个 ELF 段分配内存，再用 [`loadseg`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L76) 把每个段加载到内存中。`loadseg` 用 `walkaddr` 找到已分配内存的物理地址，ELF 段的每一页都要写到那里，并用 `readi` 从文件中读取数据。

`/init` 是第一个用 `kexec` 创建的用户程序，它的程序段头如下：

```
# objdump -p user/_init

user/_init:     file format elf64-little

Program Header:
0x70000003 off    0x0000000000006bb0 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**0
         filesz 0x000000000000004a memsz 0x0000000000000000 flags r--
    LOAD off    0x0000000000001000 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**12
         filesz 0x0000000000001000 memsz 0x0000000000001000 flags r-x
    LOAD off    0x0000000000002000 vaddr 0x0000000000001000
                                       paddr 0x0000000000001000 align 2**12
         filesz 0x0000000000000010 memsz 0x0000000000000030 flags rw-
   STACK off    0x0000000000000000 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
```

可以看到，代码段应当从文件偏移 0x1000 处的内容加载到内存的虚拟地址 0x0 处（没有写权限）。还可以看到，数据应当加载到地址 0x1000 处，这正好是一个页边界，并且没有执行权限。

程序段头的 `filesz` 可能小于 `memsz`，这表示两者之间的差额应当填零（用于 C 全局变量），而不是从文件中读取。对 `/init` 来说，数据段的 `filesz` 是 0x10 字节，`memsz` 是 0x30 字节，因此 `uvmalloc` 分配足够容纳 0x30 字节的物理内存，但只从文件 `/init` 中读取 0x10 字节。

接下来 `kexec` 分配并初始化用户栈。它只分配一个栈页。`kexec` 把参数字符串逐个复制到栈顶，并把指向它们的指针记录在 `ustack` 中。它在将要传给 `main` 的 `argv` 列表末尾放一个空指针。`argc` 和 `argv` 的值通过系统调用的返回路径传给 `main`：`argc` 通过系统调用的返回值传递，放在 `a0` 中；`argv` 通过进程 trapframe 中的 `a1` 项传递。

`kexec` 在栈页正下方放一个不可访问的页，这样试图使用超过一页栈的程序就会出错。这个不可访问的页也让 `kexec` 能处理过大的参数：在这种情况下，`kexec` 用来把参数复制到栈上的 [`copyout`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L345) 函数会发现目标页不可访问，并返回 -1。

在准备新内存映像的过程中，如果 `kexec` 检测到错误（例如无效的程序段），就跳转到标签 `bad`，释放新映像并返回 -1。`kexec` 必须等到确定系统调用会成功之后才能释放旧映像：如果旧映像没了，系统调用就无法向它返回 -1。`kexec` 中唯一的错误情况都发生在创建映像的过程中。一旦映像创建完成，`kexec` 就可以提交新页表[（exec.c:134）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L134)并释放旧页表[（exec.c:138）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c#L138)。

`exec` 系统调用把 ELF 文件中的字节加载到内存中 ELF 文件所指定的地址上。用户或进程可以在 ELF 文件中填入任何他们想要的地址。因此 `exec` 是有风险的，因为 ELF 文件中的地址可能有意或无意地指向内核。对一个不够谨慎的内核来说，后果可能从崩溃到恶意破坏内核的隔离机制（即安全漏洞利用）不等。xv6 做了一系列检查来避免这些风险。例如，`if(ph.vaddr + ph.memsz < ph.vaddr)` 检查两者之和是否溢出 64 位整数。危险在于，用户可以构造一个 ELF 二进制，其 `ph.vaddr` 指向用户选定的地址，而 `ph.memsz` 足够大，使两者之和溢出为 0x1000，看起来就像一个合法的值。在旧版本的 xv6 中，用户地址空间也包含内核（只是在用户模式下不可读写），用户可以选一个对应内核内存的地址，从而把 ELF 二进制中的数据复制进内核。在 RISC-V 版本的 xv6 中这不可能发生，因为内核有自己独立的页表；`loadseg` 加载到的是进程的页表，而不是内核的页表。

内核开发者很容易漏掉某个关键检查，现实中的内核长期以来一直有遗漏检查的问题，用户程序可以利用这些缺失的检查获得内核特权。xv6 很可能没有完整地验证用户态提供给内核的数据，恶意用户程序也许能借此绕过 xv6 的隔离。

## 3.8 现实世界

和大多数操作系统一样，xv6 用分页硬件实现内存保护和映射。大多数操作系统通过把分页和缺页异常结合起来，对分页的使用远比 xv6 复杂，我们会在第 4 章讨论这一点。

xv6 之所以简单，一是因为内核在虚拟地址和物理地址之间使用直接映射，二是因为它假定物理 RAM 位于地址 0x80000000，也就是内核期望被加载的位置。这在 QEMU 上可行，但在真实硬件上并不是个好主意；真实硬件把 RAM 和设备放在不可预知的物理地址上，所以（例如）0x80000000 处可能根本没有 RAM，而 xv6 却期望把内核存放在那里。更严肃的内核设计会利用页表，把任意的硬件物理内存布局变成可预知的内核虚拟地址布局。

RISC-V 支持物理地址级别的保护，但 xv6 没有使用这个功能。

在内存很多的机器上，使用 RISC-V 对“超级页”（super page）的支持可能是合理的。物理内存小的时候，小页更合适，能以细粒度进行分配和换出到磁盘。例如，如果一个程序只用了 8 KB 内存，给它一整个 4 MB 的物理内存超级页就是浪费。在 RAM 很多的机器上，大页更合适，还可能减少操作页表的开销。

为了避免在切换页表时刷新整个 TLB，RISC-V CPU 可能支持地址空间标识符（address space identifier，ASID）（Waterman et al. 2024）。这样内核就可以只刷新某个特定地址空间的 TLB 项。xv6 没有使用这个功能。

xv6 内核缺少一个能为小对象提供内存的类 `malloc` 分配器，这使内核无法使用需要动态分配的复杂数据结构。更精细的内核可能会分配多种不同大小的小块内存，而不是（像 xv6 那样）只分配 4096 字节的块；真正的内核分配器既要处理大的分配，也要处理小的分配。

内存分配是一个经久不衰的热门话题，基本问题是高效利用有限的内存，以及为未知的未来请求做好准备（Knuth 1997）。如今人们更关心速度，而不是空间效率。

## 3.9 练习

1. 解析 RISC-V 的设备树，找出计算机拥有的物理内存数量。

2. 函数 `copyin` 和 `copyinstr` 用软件遍历用户页表。设置内核页表，让内核也映射用户程序，使 `copyin` 和 `copyinstr` 可以用 `memcpy` 把系统调用参数复制到内核空间，依靠硬件完成页表遍历。

3. 修改 xv6，让内核使用超级页。

4. Unix 的 `exec` 实现传统上会对 shell 脚本做特殊处理。如果要执行的文件以文本 `#!` 开头，那么第一行会被当作用来解释该文件的程序。例如，如果调用 `exec` 运行 `myprog` `arg1`，而 `myprog` 的第一行是 `#!/interp`，那么 `exec` 会以命令行 `/interp` `myprog` `arg1` 运行 `/interp`。在 xv6 中实现对这一约定的支持。

5. 为内核实现地址空间布局随机化。
