# 第 10 章 文件系统

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/fs.html>。原书采用 CC BY 3.0 许可。

文件系统的作用是组织和存储数据。文件系统通常支持在用户和应用程序之间共享数据，并提供**持久性**（persistence），让数据在重启后仍然可用。

xv6 文件系统提供类 Unix 的文件、目录和路径名（见第 1 章），并把数据存放在磁盘上以实现持久化。文件系统要解决以下几个难题：

- 文件系统需要磁盘上的数据结构，来表示由命名目录和文件组成的树，记录存放每个文件内容的块是哪些，并记录磁盘上哪些区域是空闲的。
- 不同进程可能同时操作文件系统，所以文件系统代码必须相互协调，以维护不变式。
- 访问磁盘比访问内存慢好几个数量级，所以文件系统必须在内存中缓存常用的块。
- 文件系统必须支持**崩溃恢复**（crash recovery）。也就是说，如果发生崩溃（例如断电），文件系统在重启后仍必须能正确工作。风险在于，崩溃可能打断一系列更新，使磁盘上的数据结构处于不一致状态（例如某个块既被某个文件使用，又被标记为空闲）。

本章其余部分解释 xv6 如何应对前三个难题，崩溃恢复则在第 11 章中讨论。

## 10.1 概述

xv6 文件系统的实现分为七层，如图 10.1 所示。磁盘层在 virtio 硬盘上读写块；virtio 是 `qemu` 模拟出来的磁盘。缓冲区缓存层缓存磁盘块并同步对它们的访问，确保任何时刻只有一个内核进程能修改某个块中存储的数据。日志层让上层可以把对多个块的更新包装成一个**事务**（transaction），并保证在发生崩溃时这些块被原子地更新（即要么全部更新，要么全不更新）。inode 层提供单个文件，每个文件表示为一个 **inode**，它有唯一的 i 编号（i-number），以及若干存放文件数据的块。目录层把每个目录实现为一种特殊的 inode，其内容是一系列目录项，每个目录项包含一个文件名和一个 i 编号。路径名层提供层次化的路径名，如 `/usr/rtm/xv6/fs.c`，并通过递归查找来解析它们。文件描述符层用文件系统接口抽象了许多 Unix 资源（例如管道、设备、文件等），简化了应用程序员的工作。

![图 10.1：xv6 文件系统的各层](https://mit-pdos.github.io/xv6-riscv-book/fig/fslayer.svg)

*图 10.1：xv6 文件系统的各层。*

传统上，磁盘硬件把磁盘上的数据表示为一串编号的 512 字节**扇区**（sector）：扇区 0 是最前面的 512 字节，扇区 1 是接下来的 512 字节，依此类推。磁盘硬件只支持以整数个扇区为单位进行读写。因此，比如操作系统需要修改磁盘上的一个字节，它必须把包含该字节的整个扇区读入内存，修改这一个字节，再把整个扇区写回磁盘。正因如此，文件系统以一个或多个扇区为单位给文件分配磁盘空间。文件系统使用的分配粒度称为块大小（block size）。为了提高效率，大多数文件系统使用由多个扇区组成的块；xv6 的块大小是两个扇区（由 `fs.h` 中的 `BSIZE` 决定）。

xv6 把读入内存的块副本保存在 [`struct buf`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/buf.h#L1) 类型的对象中。这个结构中存储的数据有时与磁盘上的数据不同：它可能还没有从磁盘读进来（磁盘正在处理，但还没有返回块的内容），也可能已经被软件更新但还没有写回磁盘。

![图 10.2：xv6 文件系统的结构](https://mit-pdos.github.io/xv6-riscv-book/fig/fslayout.svg)

*图 10.2：xv6 文件系统的结构。*

文件系统必须规划好在磁盘上的什么位置存放 inode 和内容块。为此，xv6 把磁盘划分为几个区域，如图 10.2 所示。文件系统不使用块 0（块 0 通常存放引导代码，不过在 xv6 中它只是闲置不用）。块 1 称为**超级块**（superblock），它包含文件系统的元数据（以块为单位的文件系统大小、数据块数量、inode 数量以及日志中的块数）。从块 2 开始存放日志。日志之后是 inode，每个块中存放多个 inode。再之后是位图块，记录哪些数据块正在使用。其余的块都是数据块；每个数据块要么在位图块中被标记为空闲，要么存放某个文件或目录的内容。

xv6 文件系统最初由 xv6 之外的一个名为 `mkfs` 的程序创建。`mkfs` 写入超级块，为（空的）根目录初始化一个 inode，并把所有 inode 和数据块标记为空闲。

现在请阅读 [`kernel/buf.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/buf.h)、[`kernel/fs.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.h)、[`kernel/fs.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c)、[`kernel/sysfile.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c) 和 [`kernel/file.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c) 这几个文件。

## 10.2 缓冲区缓存层

**缓冲区缓存**（buffer cache）有两项任务：(1) 同步对磁盘块的访问，确保内存中每个块只有一份副本，并且任何时刻只有一个内核线程使用这份副本；(2) 缓存常用的块，这样就不需要从缓慢的磁盘重新读取。代码在 `bio.c` 中。

缓冲区缓存导出的主要接口是 `bread` 和 `bwrite`；前者获取一个 **buf**，其中包含某个块的副本，可以在内存中读取或修改；后者把修改过的缓冲区写到磁盘上对应的块。内核线程用完一个缓冲区后，必须调用 `brelse` 释放它。缓冲区缓存为每个缓冲区使用一个睡眠锁，确保每个缓冲区（因而每个磁盘块）任何时刻只被一个线程使用；`bread` 返回一个已加锁的缓冲区，`brelse` 释放这个锁。

缓冲区缓存用来存放磁盘块的缓冲区数量是固定的，这意味着如果文件系统请求一个不在缓存中的块，缓冲区缓存就必须回收一个当前存放着其他块的缓冲区。缓冲区缓存会回收最近最少使用的缓冲区来存放新块。这里的假设是：最近最少使用的缓冲区，也是近期最不可能再被使用的那个。

缓冲区缓存层与日志层之间有两处交互：1）为了实现事务，位于日志层之上的文件系统各层通过调用 [`log_write`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/log.c#L224) 来更新磁盘块，`log_write` 是 `bwrite` 的代理，只有日志层才调用 `bwrite`；2）在日志层提交事务之前，缓冲区缓存不能回收被该事务使用的缓冲区（见 11.2 节）。为了避免过早回收，日志层使用缓冲区缓存的函数 `bpin` 和 `bunpin` 在缓冲区缓存中固定（pin）和解除固定（unpin）一个缓冲区。

现在请阅读 [`kernel/bio.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c) 和 [`kernel/log.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/log.c) 这两个文件。

## 10.3 代码：缓冲区缓存

缓冲区缓存是一个由缓冲区组成的双向链表。函数 `binit` 由 [`main`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/main.c#L27) 调用，它用静态数组 [`buf`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L42-L51) 中的 `NBUF` 个缓冲区初始化这个链表。其他所有对缓冲区缓存的访问都通过 `bcache.head` 使用链表，而不是使用 `buf` 数组。

每个缓冲区有两个相关的状态字段。字段 `valid` 表示缓冲区中包含该块的副本。字段 `disk` 表示缓冲区的内容已经交给了磁盘，磁盘可能会修改这个缓冲区（例如把磁盘上的数据写入 `data`）。

[`bread`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L92) 调用 `bget` 为给定的块获取一个缓冲区[（bio.c:96）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L96)。如果需要从磁盘读取这个缓冲区，`bread` 会在返回缓冲区之前调用 `virtio_disk_rw` 完成读取。

[`bget`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L58) 扫描缓冲区链表，查找具有给定设备号和块号的缓冲区[（bio.c:64-72）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L64-L72)。如果存在这样的缓冲区，`bget` 就获取该缓冲区的睡眠锁，然后返回这个已加锁的缓冲区。

如果给定的块没有已缓存的缓冲区，`bget` 就必须创建一个，可能会复用一个存放其他块的缓冲区。它第二次扫描缓冲区链表，查找一个没有被使用的缓冲区（`b->refcnt = 0`）；任何这样的缓冲区都可以使用。`bget` 修改缓冲区的元数据，记录新的设备号和块号，并获取它的睡眠锁。注意，赋值 `b->valid = 0` 确保 `bread` 会从磁盘读取块数据，而不会错误地使用缓冲区之前的内容。

每个磁盘块至多只有一个缓存缓冲区，这一点很重要：它确保读者能看到写入的内容，而且文件系统要利用缓冲区上的锁来做同步。`bget` 通过一直持有 `bcache.lock` 来保证这个不变式：从第一个循环检查该块是否已缓存开始，直到第二个循环宣告该块现已缓存（通过设置 `dev`、`blockno` 和 `refcnt`）为止。这使得“检查块是否存在”与（如果不存在）“指定一个缓冲区来存放该块”这两步成为原子操作。

`bget` 在 `bcache.lock` 临界区之外获取缓冲区的睡眠锁是安全的，因为非零的 `b->refcnt` 能防止该缓冲区被复用于另一个磁盘块。睡眠锁保护对块的缓冲内容的读写，而 `bcache.lock` 保护的是“哪些块已被缓存”这一信息。

如果所有缓冲区都在使用中，说明同时执行文件系统调用的进程太多了；此时 `bget` 会 panic。更优雅的处理方式可能是睡眠，直到有缓冲区空闲出来，不过那样就有可能发生死锁。

一旦 `bread` 读取了磁盘（如果需要的话）并把缓冲区返回给调用者，调用者就独占这个缓冲区，可以读写其中的数据字节。如果调用者确实修改了缓冲区，它必须在释放缓冲区之前调用 `bwrite`，把修改后的数据写到磁盘。[`bwrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L107) 调用 `virtio_disk_rw` 与磁盘硬件交互。

调用者用完一个缓冲区后，必须调用 `brelse` 释放它。（`brelse` 这个名字是 b-release 的缩写，虽然晦涩，但值得记住：它起源于 Unix，在 BSD、Linux 和 Solaris 中也在使用。）[`brelse`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L117) 释放睡眠锁，并把缓冲区移到链表的最前面[（bio.c:128-133）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c#L128-L133)。移动缓冲区使得链表按照缓冲区最近被使用（即被释放）的时间排序：链表中的第一个缓冲区是最近使用的，最后一个是最近最少使用的。`bget` 中的两个循环利用了这一点：查找已有缓冲区的扫描在最坏情况下必须遍历整个链表，但当访问局部性良好时，先检查最近使用的缓冲区（从 `bcache.head` 开始，沿 `next` 指针前进）可以减少扫描时间。挑选可复用缓冲区的扫描则通过反向扫描（沿 `prev` 指针前进）来选出最近最少使用的缓冲区。

## 10.4 代码：块分配器

文件和目录的内容存储在磁盘块中，这些块必须从一个空闲池中分配。xv6 的块分配器在磁盘上维护一个空闲位图，每个块对应一位。某位为 0 表示对应的块空闲；为 1 表示该块正在使用。`mkfs` 在创建新文件系统时，会设置引导扇区、超级块、日志块、inode 块和位图块所对应的位。

块分配器提供两个函数：`balloc` 分配一个新的磁盘块，`bfree` 释放一个块。`balloc` 中位于[（fs.c:74）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L74)的循环会考察每一个块，从块 0 一直到 `sb.size`（文件系统中的块数）。它寻找位图位为 0 的块，即空闲块。如果 `balloc` 找到这样的块，就更新位图并返回该块。为了提高效率，这个循环被拆成两部分。外层循环读取每个位图块。内层循环检查单个位图块中全部 `BPB`（Bits-Per-Block，每块位数）个位。如果两个进程同时尝试分配块，可能会发生竞争，但缓冲区缓存一次只让一个进程使用某个位图块（见 10.2 节），从而避免了这种竞争。

[`bfree`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L94) 找到对应的位图块并清除对应的位。同样，`bread` 和 `brelse` 所隐含的独占使用避免了显式加锁的需要。

与本章其余部分描述的许多代码一样，`balloc` 和 `bfree` 必须在事务中调用。

## 10.5 inode 层

**inode** 这个术语有两种相关的含义。它可能指磁盘上的数据结构，其中包含文件的大小和数据块编号列表。也可能指内存中的 inode，它包含磁盘上 inode 的一份副本，以及内核中需要的额外信息。

磁盘上的 inode 紧密排列在磁盘上一块连续的区域中，这块区域称为 inode 块。每个 inode 大小相同，所以给定一个数字 n，很容易找到磁盘上的第 n 个 inode。实际上，这个数字 n 称为 inode 编号或 i 编号（i-number），在实现中就是用它来标识 inode 的。

磁盘上的 inode 由 [`struct dinode`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.h#L32) 定义。`type` 字段区分普通文件、目录和特殊文件（设备）。类型为 0 表示该磁盘 inode 是空闲的。`nlink` 字段统计引用这个 inode 的目录项数量，用来判断何时应该释放该磁盘 inode 及其数据块。`size` 字段记录文件内容的字节数。`addrs` 数组记录存放文件内容的磁盘块的块号。

内核把活跃 inode 的集合保存在内存中一个名为 `itable` 的表里；[`struct inode`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.h#L17) 是磁盘上 `struct` `dinode` 在内存中的副本。只有当存在指向某个 inode 的 C 指针时，内核才会把它保存在内存中。`ref` 字段统计指向该内存 inode 的 C 指针数量，如果引用计数降为 0，内核就会把这个 inode 从内存中丢弃。`iget` 和 `iput` 函数获取和释放指向 inode 的指针，并相应修改引用计数。指向 inode 的指针可能来自文件描述符、当前工作目录，以及 `kexec` 这类临时性的内核代码。

xv6 的 inode 代码中有四种锁或类似锁的机制。`itable.lock` 保护两个不变式：一个 inode 在 inode 表中至多出现一次；内存 inode 的 `ref` 字段等于指向该 inode 的内存指针数量。每个内存 inode 都有一个包含睡眠锁的 `lock` 字段，它保证对 inode 字段（如文件长度）以及对 inode 的文件或目录内容块的独占访问。inode 的 `ref` 如果大于 0，系统就会把该 inode 保留在表中，不会把这个表项复用于另一个 inode。最后，每个 inode 都有一个 `nlink` 字段（存在磁盘上，如果 inode 在内存中，内存里也有一份副本），统计引用该文件的目录项数量；如果一个 inode 的链接计数大于 0，xv6 就不会释放它。

`iget()` 返回的 `struct` `inode` 指针在对应的 `iput()` 调用之前保证有效：inode 不会被删除，指针所指的内存也不会被复用于另一个 inode。`iget()` 提供对 inode 的非独占访问，因此可以有许多指针指向同一个 inode。文件系统代码的许多部分都依赖 `iget()` 的这种行为，既用来长期持有对 inode 的引用（如打开的文件和当前目录），也用来在操作多个 inode 的代码（如路径名查找）中防止竞争，同时避免死锁。

`iget` 返回的 `struct` `inode` 可能没有任何有用的内容。为了确保它持有磁盘 inode 的副本，代码必须调用 `ilock`。`ilock` 会锁住这个 inode（使其他进程无法再 `ilock` 它），并在尚未读取的情况下从磁盘读入 inode。`iunlock` 释放 inode 上的锁。把获取 inode 指针与加锁分开，有助于在某些情况下避免死锁，例如在目录查找期间。多个进程可以持有 `iget` 返回的指向同一 inode 的 C 指针，但同一时刻只有一个进程能锁住这个 inode。

inode 表只保存被内核代码或数据结构以 C 指针持有的 inode。它的主要任务是同步多个进程的访问。inode 表恰好也缓存了常用的 inode，但缓存只是次要的；如果某个 inode 被频繁使用，缓冲区缓存很可能会把它留在内存中。修改了内存 inode 的代码会用 `iupdate` 把它写回磁盘。

## 10.6 代码：inode

为了分配一个新 inode（例如在创建文件时），xv6 调用 [`ialloc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L201)。`ialloc` 与 `balloc` 类似：它逐块遍历磁盘上的 inode 结构，查找被标记为空闲的那个。找到之后，它把新的 `type` 写入磁盘来占用这个 inode，然后在末尾调用 [`iget`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L215)，返回 inode 表中的一个表项。`ialloc` 能正确工作，依赖于同一时刻只有一个进程能持有对 `bp` 的引用：`ialloc` 可以确信不会有其他进程同时发现这个 inode 可用并试图占用它。

[`iget`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L249) 在 inode 表中查找具有所需设备号和 inode 编号的活跃表项（`ip->ref` `>` `0`）。如果找到，就返回对该 inode 的一个新引用[（fs.c:258-262）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L258-L262)。`iget` 在扫描时会记下第一个空槽的位置[（fs.c:263-264）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L263-L264)，在需要分配表项时使用它。

代码在读写 inode 的元数据或内容之前，必须先用 `ilock` 锁住它。[`ilock`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L295) 为此使用了一个睡眠锁。一旦 `ilock` 获得了对 inode 的独占访问，它会在需要时从磁盘（更可能是从缓冲区缓存）读取 inode。函数 [`iunlock`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L323) 释放睡眠锁，这可能会唤醒正在睡眠的进程。

[`iput`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L350) 通过递减引用计数来释放指向 inode 的 C 指针[（fs.c:373）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L373)。如果这是最后一个引用，该 inode 在 inode 表中的槽位就空闲了，可以复用于另一个 inode。

如果 `iput` 发现某个 inode 既没有 C 指针引用，也没有任何链接指向它（不出现在任何目录中）[（fs.c:357）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L357)，那么这个 inode 及其数据块就必须被释放。`iput` 调用 `itrunc` 把文件截断为 0 字节，释放数据块，然后调用 [`ifree`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L333) 把磁盘上该 inode 的类型设为 0（未分配）。

`iput` 在释放 inode 时的加锁协议值得仔细看一看。一个危险是，可能有一个并发线程正在 `ilock` 中等待使用这个 inode（例如读取文件或列出目录），而它并没有预料到这个 inode 已不再被分配。这种情况不会发生，因为如果一个内存 inode 没有链接指向它，且 `ip->ref` 为 1，系统调用就没有办法获得指向它的指针。那唯一的一个引用属于调用 `iput` 的线程。另一个主要危险是，一个并发的 `ialloc` 调用可能会选中 `iput` 正在释放的同一个 inode。这只可能在 `ifree` 写入磁盘、使该 inode 的类型变为 0 之后才发生。`iput` 最后才调用 `ifree`，此前它已经截断了 inode、释放了 inode 的睡眠锁，并放弃了自己的引用。所以当分配线程能看到这个 inode 空闲时，`iput` 已经用完它了，这个竞争是无害的。

`iput()` 可能会写磁盘。这意味着任何使用文件系统的系统调用都可能写磁盘，因为这个系统调用可能是最后一个持有该文件引用的调用。即使是 `read()` 这样看起来只读的调用，最终也可能调用 `iput()`。这又意味着，即使是只读的系统调用，只要使用了文件系统，也必须包装在事务中。

`iput()` 与崩溃之间有一个棘手的交互。当文件的链接计数降为 0 时，`iput()` 不会立即截断文件，因为某个进程可能仍在内存中持有对该 inode 的引用：一个进程成功打开了文件，可能仍在读写它。但是，如果在最后一个进程关闭该文件的文件描述符之前发生崩溃，那么这个文件在磁盘上会被标记为已分配，却没有任何目录项指向它。

文件系统用以下两种方式之一处理这种情况。简单的方案是：在重启后的恢复阶段，文件系统扫描整个文件系统，查找被标记为已分配、但没有目录项指向的文件。如果存在这样的文件，就可以释放它们。xv6 实现的就是这种方案。在启动期间，`fsinit` 调用 [`ireclaim`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L389)，后者检查磁盘上的每个 inode，查找被标记为已分配（类型非零）但链接计数为 0 的 inode；它们恰好就是那些已被取消链接、但在崩溃发生时仍处于打开状态的文件。对每个这样的孤儿 inode，`ireclaim` 调用 `iput` 来释放该 inode 及其数据块。

第二种方案不需要扫描文件系统。在这种方案中，如果某个文件的链接计数降为 0 而引用计数不为 0，文件系统就把它的 inode 编号记录在磁盘上（例如在超级块中）。如果文件系统在该文件的引用计数降为 0 时删除了它，就从磁盘上的列表中移除这个 inode 以更新列表。在恢复时，文件系统释放列表中的所有文件。

## 10.7 代码：inode 内容

![图 10.3：文件在磁盘上的表示](https://mit-pdos.github.io/xv6-riscv-book/fig/inode.svg)

*图 10.3：文件在磁盘上的表示。*

磁盘上的 inode 结构 `struct dinode` 包含一个大小和一个块号数组（见图 10.3）。inode 的数据存放在 `dinode` 的 `addrs` 数组所列出的块中。最前面的 `NDIRECT` 个数据块列在数组的前 `NDIRECT` 个表项中；这些块称为**直接块**（direct block）。接下来的 `NINDIRECT` 个数据块不是列在 inode 中，而是列在一个称为**间接块**（indirect block）的数据块中。`addrs` 数组的最后一个表项给出间接块的地址。因此，文件最前面的 12 kB（`NDIRECT` `x` `BSIZE`）字节可以从 inode 中列出的块加载，而接下来的 `256` kB（`NINDIRECT` `x` `BSIZE`）字节则只有在查阅间接块之后才能加载。对磁盘来说这是一种不错的表示，但对使用者来说较为复杂。函数 `bmap` 负责管理这种表示，使 `readi` 和 `writei` 这类更高层的例程（我们稍后会看到）不需要处理这种复杂性。`bmap` 返回 inode `ip` 的第 `bn` 个数据块的磁盘块号。如果 `ip` 还没有这个块，`bmap` 就分配一个。

函数 [`bmap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L421) 先处理简单的情况：最前面的 `NDIRECT` 个块列在 inode 本身中[（fs.c:426-434）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L426-L434)。接下来的 `NINDIRECT` 个块列在 `ip->addrs[NDIRECT]` 处的间接块中。`bmap` 读取间接块[（fs.c:445）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L445)，然后从块内的正确位置读出一个块号[（fs.c:446）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L446)。如果块号超过 `NDIRECT+NINDIRECT`，`bmap` 就会 panic；`writei` 中有检查来防止这种情况发生[（fs.c:551）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L551)。

`bmap` 按需分配块。`ip->addrs[]` 或间接块中的表项为 0，表示尚未分配块。`bmap` 遇到 0 时，会用按需分配的新块的块号替换它们[（fs.c:427-428）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L427-L428)[（fs.c:439-440）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L439-L440)。

`itrunc` 释放文件的块，并把 inode 的大小重置为 0。[`itrunc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L464) 先释放直接块[（fs.c:470-475）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L470-L475)，再释放间接块中列出的块[（fs.c:480-483）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L480-L483)，最后释放间接块本身[（fs.c:485-486）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L485-L486)。

`bmap` 让 `readi` 和 `writei` 能方便地访问 inode 的数据。[`readi`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L510) 首先确保偏移量和字节数没有超出文件末尾。从文件末尾之后开始的读取会返回错误[（fs.c:515-516）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L515-L516)，而从文件末尾开始或跨越文件末尾的读取会返回比请求更少的字节[（fs.c:517-518）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L517-L518)。主循环逐块处理文件，把数据从缓冲区复制到 [`dst`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L520-L532)。[`writei`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L544) 与 `readi` 相同，只有三处例外：从文件末尾开始或跨越文件末尾的写入会让文件增长，直到最大文件大小[（fs.c:551-552）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L551-L552)；循环把数据复制进缓冲区而不是复制出来[（fs.c:560）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L560)；如果写入扩展了文件，`writei` 必须更新文件大小[（fs.c:570-571）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L570-L571)。

函数 [`stati`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L496) 把 inode 的元数据复制到 `stat` 结构中，用户程序可以通过 `stat` 系统调用获取这个结构。

## 10.8 代码：目录层

目录在内部的实现与文件很相似。它的 inode 类型为 `T_DIR`，数据是一系列目录项。每个目录项是一个 [`struct dirent`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.h#L58)，包含一个名字和一个 inode 编号。名字最多 `DIRSIZ`（14）个字符；如果更短，就以一个 NULL（0）字节结尾。inode 编号为 0 的目录项是空闲的。

目录项包含的是文件的 i 编号而不是被命名文件的整个 inode，原因是要支持由 `link()` 系统调用创建的“链接”（link）。一个 inode 可以被多个目录引用，因而可以有多个路径名；inode 的每个名字（即每个目录项）称为一个链接。由于一个 inode 可能在多个目录中被命名，把 inode 存放在其中任何一个目录里都不方便。可能存在多个链接，这也是 inode 中设有 `nlink` 字段的原因。inode 与目录分开存放，还使得系统能合理地处理这种情况：某个进程持有指向一个文件的文件描述符时，该文件被取消链接（删除）。文件描述符指向的是 inode，而不是某个目录项，所以即使文件（实际上只是它的名字）被删除，文件描述符仍然可以使用。

函数 [`dirlookup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L592) 在目录中查找具有给定名字的目录项。如果找到，它返回指向对应 inode 的指针（未加锁），并把 `*poff` 设为该目录项在目录中的字节偏移量，以便调用者需要时修改它。如果 `dirlookup` 找到名字匹配的目录项，它会更新 `*poff`，并返回一个通过 `iget` 获得的未加锁 inode。`dirlookup` 正是 `iget` 返回未加锁 inode 的原因。调用者已经锁住了 `dp`，所以如果查找的是 `.`（当前目录的别名），在返回前试图锁住该 inode 就会再次锁 `dp`，导致死锁。（还有更复杂的死锁场景，涉及多个进程和 `..`（父目录的别名）；`.` 并不是唯一的问题。）调用者可以先解锁 `dp`，再锁住 `ip`，确保同一时刻只持有一把锁。

函数 [`dirlink`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L620) 把一个具有给定名字和 inode 编号的新目录项写入目录 `dp`。如果名字已经存在，`dirlink` 返回错误[（fs.c:626-630）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L626-L630)。主循环读取目录项，查找未分配的目录项。找到后就提前结束循环[（fs.c:632-637）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L632-L637)，此时 `off` 被设为该可用目录项的偏移量。否则，循环结束时 `off` 被设为 `dp->size`。无论哪种情况，`dirlink` 接下来都会在偏移量 [`off`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L642-L643) 处写入，向目录添加一个新目录项。

## 10.9 代码：路径名

路径名查找需要一连串的 `dirlookup` 调用，路径的每个组成部分调用一次。[`namei`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L731) 解析 `path` 并返回对应的 `inode`。函数 `nameiparent` 是一个变体：它在最后一个元素之前停下，返回父目录的 inode，并把最后一个元素复制到 `name` 中。两者都调用通用函数 `namex` 来完成实际工作。

[`namex`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L692) 首先决定路径解析从哪里开始。如果路径以斜杠开头，就从根目录开始解析；否则从当前目录开始[（fs.c:696-699）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L696-L699)。然后它用 `skipelem` 依次处理路径中的每个元素[（fs.c:701）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L701)。循环的每次迭代都必须在当前 inode `ip` 中查找 `name`。每次迭代先锁住 `ip`，并检查它是不是目录。如果不是，查找就失败[（fs.c:702-706）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L702-L706)。（必须锁住 `ip`，并不是因为 `ip->type` 会在背后被改变（它不会），而是因为在 `ilock` 运行之前，不能保证 `ip->type` 已从磁盘加载。）如果调用的是 `nameiparent`，并且这是路径的最后一个元素，那么按照 `nameiparent` 的定义，循环会提前结束；最后一个路径元素已经被复制到 `name` 中，所以 `namex` 只需返回未加锁的 [`ip`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L711-L715)。最后，循环用 `dirlookup` 查找路径元素，并通过设置 [`ip = next`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c#L716-L721) 为下一次迭代做准备。当循环处理完所有路径元素后，返回 `ip`。

`namex` 过程可能要花很长时间才能完成：对于路径名中经过的目录，它可能需要多次磁盘操作来读取 inode 和目录块（如果它们不在缓冲区缓存中）。xv6 经过精心设计，使得当某个内核线程调用的 `namex` 因磁盘 I/O 而阻塞时，另一个查找不同路径名的内核线程可以并发地继续执行。`namex` 分别锁住路径中的每个目录，因此对不同目录的查找可以并行进行。

这种并发带来了一些难题。例如，在一个内核线程查找路径名的同时，另一个内核线程可能正在通过取消链接一个目录来修改目录树。一个潜在的风险是，查找操作可能正在搜索一个已经被另一个内核线程删除的目录，而该目录的块已被复用于另一个目录或文件。

xv6 避免了这类竞争。例如，在 `namex` 中执行 `dirlookup` 时，查找线程持有该目录的锁，而 `dirlookup` 返回的是通过 `iget` 获得的 inode。`iget` 会增加该 inode 的引用计数。只有在从 `dirlookup` 拿到 inode 之后，`namex` 才释放目录上的锁。此时另一个线程可能会把该 inode 从目录中取消链接，但 xv6 还不会删除这个 inode，因为它的引用计数仍大于 0。

另一个风险是死锁。例如，查找 "." 时，`next` 与 `ip` 指向同一个 inode。在释放 `ip` 上的锁之前锁住 `next` 会导致死锁。为了避免这种死锁，`namex` 在获取 `next` 的锁之前先解锁目录。这里我们再次看到 `iget` 与 `ilock` 分离的重要性。

## 10.10 文件描述符层

Unix 接口一个很妙的地方是，Unix 中的大多数资源都表示为文件，包括控制台这样的设备、管道，当然还有真正的文件。文件描述符层就是实现这种统一性的那一层。

正如第 1 章所述，xv6 为每个进程提供它自己的打开文件表，即文件描述符表。每个打开的文件由一个 [`struct file`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.h#L1) 表示，它是对 inode 或管道的封装，外加一个 I/O 偏移量。每次调用 `open` 都会创建一个新的打开文件（一个新的 `struct` `file`）：如果多个进程各自独立地打开同一个文件，不同的实例会有不同的 I/O 偏移量。另一方面，同一个打开文件（同一个 `struct` `file`）可以在一个进程的文件表中出现多次，也可以出现在多个进程的文件表中。如果一个进程用 `open` 打开文件，然后用 `dup` 创建别名，或者通过 `fork` 与子进程共享它，就会出现这种情况。引用计数记录了对某个打开文件的引用数量。文件可以以只读、只写或读写方式打开，`readable` 和 `writable` 字段记录了这一点。

系统中所有打开的文件都保存在一个全局文件表 `ftable` 中。文件表提供以下函数：分配文件（`filealloc`）、创建重复引用（`filedup`）、释放引用（`fileclose`），以及读写数据（`fileread` 和 `filewrite`）。

前三个函数的形式我们已经很熟悉了。[`filealloc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L30) 在文件表中扫描一个未被引用的文件（`f->ref` `==` `0`）并返回一个新引用；[`filedup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L48) 增加引用计数；[`fileclose`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L60) 减少引用计数。当文件的引用计数降为 0 时，`fileclose` 根据类型释放底层的管道或 inode。

函数 `filestat`、`fileread` 和 `filewrite` 实现了对文件的 `fstat`、`read` 和 `write` 操作。[`filestat`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L88) 只允许用于 inode，它调用 `stati`。`fileread` 和 `filewrite` 检查打开模式是否允许该操作，然后把调用转给管道或 inode 的实现。如果文件表示的是 inode，`fileread` 和 `filewrite` 就把 I/O 偏移量用作操作的偏移量，然后推进它[（file.c:122-123）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L122-L123)[（file.c:162-163）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L162-L163)。管道没有偏移量的概念。回忆一下，inode 函数要求调用者负责加锁[（file.c:94-96）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L94-L96)[（file.c:121-124）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L121-L124)[（file.c:161-164）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.c#L161-L164)。inode 加锁带来一个方便的副作用：读写偏移量是原子更新的，因此多个进程同时写同一个文件时不会覆盖彼此的数据，不过它们写入的内容可能会交错在一起。

## 10.11 代码：系统调用

有了下层提供的这些函数，大多数系统调用的实现都很简单（见[（sysfile.c）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c)）。有几个调用值得仔细看一看。

函数 `sys_link` 和 `sys_unlink` 编辑目录，创建或删除对 inode 的引用。它们是展示事务威力的又一个好例子。[`sys_link`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L124) 首先获取它的参数，即两个字符串 `old` 和 [`new`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L129)。假设 `old` 存在、不是目录，并且链接数少于 `NLINK_MAX`[（sysfile.c:133-145）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L133-L145)，`sys_link` 就递增它的 `ip->nlink` 计数[（sysfile.c:151）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L151)。之所以有这个上限，是因为 `nlink` 在磁盘 inode 中是一个 16 位字段；如果不检查，创建过多链接会让计数溢出。然后 `sys_link` 调用 `nameiparent` 找到 [`new`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L155) 的父目录和最后一个路径元素，并创建一个指向 `old` 的 inode 的新目录项[（sysfile.c:165）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L165)。新的父目录必须存在，并且必须与已有 inode 位于同一设备上：inode 编号只在单个磁盘上具有唯一的含义。如果发生这类错误，`sys_link` 必须回头把 `ip->nlink` 减回去。

事务简化了实现，因为这需要更新多个磁盘块，但我们不必担心更新的顺序。它们要么全部成功，要么全都不生效。例如，如果没有事务，在创建链接之前更新 `ip->nlink`，会让文件系统暂时处于不安全的状态，而在这两步之间发生崩溃可能会造成严重破坏。有了事务，我们就不必担心这个问题。

`sys_link` 为已有的 inode 创建一个新名字。函数 [`create`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L259) 则为新 inode 创建一个新名字。它是三个创建文件的系统调用的通用版本：带 `O_CREATE` 标志的 `open` 创建一个新的普通文件，`mkdir` 创建一个新目录，`mkdev` 创建一个新的设备文件。和 `sys_link` 一样，`create` 首先调用 `nameiparent` 获取父目录的 inode。然后它调用 `dirlookup` 检查该名字是否已经存在[（sysfile.c:280）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L280)。如果名字已经存在，`create` 的行为取决于它是为哪个系统调用服务的：`open` 的语义与 `mkdir` 和 `mkdev` 不同。如果 `create` 是代表 `open` 被调用的（`type` `==` `T_FILE`），并且已存在的那个名字本身是一个普通文件，那么 `open` 把这视为成功，`create` 也同样如此[（sysfile.c:284）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L284)。否则就是错误[（sysfile.c:285-286）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L285-L286)。如果名字尚不存在，`create` 就用 [`ialloc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L289) 分配一个新 inode。如果新 inode 是目录，`create` 用 `.` 和 `..` 两个目录项初始化它。最后，在数据已经正确初始化之后，`create` 就可以把它链接到父目录中[（sysfile.c:306）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L306)。和 `sys_link` 一样，`create` 同时持有两个 inode 锁：`ip` 和 `dp`。这不会产生死锁，因为 inode `ip` 是新分配的：系统中不会有其他进程持有 `ip` 的锁并试图去锁 `dp`。

有了 `create`，就很容易实现 `sys_open`、`sys_mkdir` 和 `sys_mknod`。[`sys_open`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L329) 最复杂，因为创建新文件只是它所能做的事情中的一小部分。如果传给 `open` 的是 `O_CREATE` 标志，它就调用 [`create`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L344)。否则，它调用 [`namei`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L350)。`create` 返回的是已加锁的 inode，而 `namei` 不是，所以 `sys_open` 必须自己锁住 inode。这里正好可以方便地检查目录只能以读方式打开，不能以写方式打开。假设通过某种方式获得了 inode，`sys_open` 就分配一个文件和一个文件描述符[（sysfile.c:368）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L368)，然后填写这个文件[（sysfile.c:380-385）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c#L380-L385)。注意，其他进程无法访问这个部分初始化的文件，因为它只在当前进程的表中。

第 9 章在我们还没有文件系统之前就考察了管道的实现。函数 `sys_pipe` 通过提供一种创建管道对的方法，把那个实现与文件系统连接起来。它的参数是一个指针，指向可存放两个整数的空间，它会在那里记录两个新的文件描述符。然后它分配管道并安装这两个文件描述符。

## 10.12 现实世界

真实操作系统中的缓冲区缓存比 xv6 的复杂得多，但它服务于同样的两个目的：缓存和同步对磁盘的访问。xv6 的缓冲区缓存使用最近最少使用（LRU）淘汰策略；还可以实现许多更复杂的策略，每种策略都适合某些工作负载，而对另一些则不太合适。更高效的 LRU 缓存会去掉链表，改用哈希表做查找、用堆做 LRU 淘汰。现代缓冲区缓存通常与虚拟内存系统集成在一起，以支持内存映射文件。

xv6 使用的 inode 和目录的磁盘布局与早期 UNIX 类似；多年来这种方案一直沿用，生命力惊人。BSD 的 UFS/FFS 和 Linux 的 ext2/ext3 使用的基本上是相同的数据结构。文件系统布局中效率最低的部分是目录，每次查找都需要线性扫描它的所有磁盘块。当目录只有几个磁盘块时，这是合理的，但对于包含很多文件的目录来说代价很高。Microsoft Windows 的 NTFS、macOS 的 HFS 和 Solaris 的 ZFS 等文件系统（这里只举几个例子）把目录实现为磁盘上由块组成的平衡树。这样做很复杂，但能保证目录查找是对数时间的。

xv6 要求文件系统放在一个磁盘设备上，并且大小不能改变。随着大型数据库和多媒体文件把存储需求推得越来越高，操作系统正在开发各种方法来消除“一个文件系统只用一块磁盘”的瓶颈。基本方法是把多块磁盘组合成一个逻辑磁盘。RAID 这样的硬件方案仍然最流行，但目前的趋势是尽可能多地用软件来实现这部分逻辑。这些软件实现通常支持丰富的功能，比如通过动态添加或移除磁盘来扩大或缩小逻辑设备。当然，一个能动态扩大或缩小的存储层，需要一个同样能做到这一点的文件系统：xv6 使用的固定大小的 inode 块数组在这种环境下就不太好用。把磁盘管理与文件系统分开可能是最清晰的设计，但两者之间复杂的接口促使一些系统（如 Sun 的 ZFS）把它们合二为一。

xv6 的文件系统缺少现代文件系统的许多其他特性；例如，它不支持快照和增量备份。

现代 Unix 系统允许用访问磁盘存储的同一套系统调用来访问许多种资源：命名管道、网络连接、远程访问的网络文件系统，以及 `/proc` 这样的监控和控制接口。这些系统通常不像 xv6 那样在 `fileread` 和 `filewrite` 中使用 `if` 语句，而是给每个打开的文件一个函数指针表，每种操作对应一个函数指针，通过调用函数指针来调用该 inode 对此调用的实现。网络文件系统和用户级文件系统提供的函数会把这些调用转换为网络 RPC，并在返回之前等待响应。

## 10.13 练习

1. 为什么 `balloc` 中要 panic？xv6 能恢复吗？
2. 为什么 `ialloc` 中要 panic？xv6 能恢复吗？
3. 为什么 `filealloc` 在文件用完时不 panic？为什么这种情况更常见，因而值得处理？
4. 假设在 `sys_link` 调用 `iunlock(ip)` 和 `dirlink` 之间，`ip` 对应的文件被另一个进程取消了链接。链接还能被正确创建吗？为什么？
5. `create` 进行了四次要求必须成功的函数调用（一次 `ialloc`，三次 `dirlink`）。只要其中任何一次失败，`create` 就调用 `panic`。为什么这是可以接受的？为什么这四次调用都不可能失败？
6. `sys_chdir` 在 `iput(cp->cwd)` 之前调用 `iunlock(ip)`，而 `iput(cp->cwd)` 可能会尝试锁住 `cp->cwd`；然而，把 `iunlock(ip)` 推迟到 `iput` 之后也不会造成死锁。为什么？
7. 实现 `lseek` 系统调用。要支持 `lseek`，你还需要修改 `filewrite`：如果 `lseek` 把 `off` 设到了 `f->ip->size` 之后，就要用 0 填充文件中的空洞。
8. 为 `open` 添加 `O_TRUNC` 和 `O_APPEND`，使 shell 中的 `>` 和 `>>` 运算符能够工作。
9. 修改文件系统，使其支持符号链接。
10. 修改文件系统，使其支持命名管道。
11. 修改文件系统和虚拟内存系统，使其支持内存映射文件。

