# 第 9 章 睡眠与唤醒

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/sleep.html>。原书采用 CC BY 3.0 许可。

调度和锁有助于让一个线程的行为对另一个线程不可见，但我们还需要一些抽象，帮助线程有意地相互配合。例如，xv6 中管道的读者可能需要等待写进程产生数据；父进程调用 `wait` 时可能需要等待子进程退出；读磁盘的进程需要等待磁盘硬件完成读取。xv6 内核在这些（以及许多其他）场合使用一种称为睡眠与唤醒的机制。睡眠让内核线程等待某个**条件**（condition）成立；另一个线程或中断处理程序可以使该条件成立（通常是修改某些变量），然后调用 wakeup，表示等待该条件的线程应当恢复运行。睡眠与唤醒常被称为**顺序协调**（sequence coordination）或**条件同步**（conditional synchronization）机制。

继续之前，请先阅读 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中的 `sleep_prepare()`、`sleep()` 和 `wakeup()` 函数，以及 [`kernel/pipe.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c) 整个文件。

## 9.1 概述

sleep/wakeup 接口如下：

```c
void sleep_prepare(void *chan)
void sleep()
void wakeup(void *chan)
```

进程睡眠是为了等待某个条件成立，参数 `chan`（**等待通道**，wait channel）用来标识所等待的条件。`sleep_prepare(chan)` 声明进程可能很快会睡眠，以等待所指定的等待通道；它会立即返回。当 `chan` 所标识的条件已经（或可能已经）成立时，调用 `wakeup(chan)`；它会唤醒所有用同一个 `chan` 调用了 `sleep_prepare(chan)` 之后又调用了 `sleep()` 的进程。如果在（必需的）前一次 `sleep_prepare(chan)` 调用之后已经调用过 `wakeup(chan)`，`sleep()` 会立即返回；否则，`sleep()` 把调用进程标记为 `SLEEPING`（而不是 `RUNNABLE`），并通过上下文切换到调度器来释放 CPU，让其他进程可以运行。`sleep_prepare()` 和 `sleep()` 由同一个进程调用（通常两者相隔不过几行代码）；`wakeup()` 则由另一个进程或中断处理程序调用。

这些函数把 `chan` 当作一个不透明的 64 位值；它们对它做的唯一事情就是比较是否相等。通常的做法是调用者把某个方便使用的对象的地址作为 `chan` 参数传入。内核程序员负责选定等待通道的约定，并负责让需要相互配合的 `sleep_prepare()` 和 `wakeup()` 调用传入同一个 `chan`。

内核代码睡眠是为了等待某个条件成立。例如，从管道读取数据的内核代码在管道缓冲区为空时会睡眠；这里的条件是管道缓冲区变为非空（因为另一个进程向管道写入了数据）。`sleep_prepare()` 和 `sleep` 并不知道如何检查条件：只有调用代码知道。通常的模式是：可能睡眠的进程先获取保护该条件的锁；然后检查条件；如果条件不成立，就调用 `sleep_prepare()`，释放锁，再调用 `sleep()`。之后使条件成立的代码会调用 `wakeup`。

下面是 xv6 内核管道代码睡眠方式的示意：

```c
piperead(pipe){
  acquire(&pipe->lock);
  while(there's no data in pipe->buffer){
    sleep_prepare(&pipe);
    release(&pipe->lock);
    sleep();
    acquire(&pipe->lock);
  }
  remove the data from the pipe;
  release(&pipe->lock);
}

pipewrite(pipe){
  acquire(&pipe->lock);
  append data to pipe->buffer;
  wakeup(&pipe);
  release(&pipe->lock);
}
```

这段代码用管道数据结构的地址作为等待通道。

`sleep_prepare(chan)` 必须在检查条件之前调用，或者在释放保护该条件的锁之前调用。这次调用记录下进程可能需要在所指定的 `chan` 上睡眠这一事实。如果另一个进程（或中断）在 `sleep_prepare(chan)` 和 `sleep()` 两次调用之间调用了 `wakeup(chan)`，`wakeup()` 会在这个即将睡眠的进程中设置一个标志，表示它实际上不应睡眠。如果 `sleep()` 看到这个标志，就直接返回而不睡眠。

为什么需要 `sleep_prepare(chan)`？在管道读者发现缓冲区为空、到它调用 `sleep` 之间，条件有可能变为成立，并且 `wakeup()` 已被调用。如果不做一些巧妙的处理，`wakeup()` 不会找到任何正在睡眠的进程，因而不起任何作用。随后该进程会调用 `sleep()`，尽管管道缓冲区里已经有数据可读；而如果（这完全可能）管道写者再也不写任何东西，读者就可能错误地永远睡下去。这种不希望出现的情况称为**丢失唤醒**（lost wake-up）。上一段描述的行为避免了这个问题。

在管道的例子中，和大多数睡眠代码一样，有一个锁保护条件，并防止在准备睡眠的进程检查条件和调用 `sleep_prepare()` 之间调用 `wakeup()`。这个锁常被称为**条件锁**（condition lock）。进程必须在调用 `sleep()` 之前释放这个锁（以便其他进程或中断处理程序能够修改条件并调用 `wakeup()`），并在 `sleep()` 返回后重新获取它。

## 9.2 代码：睡眠与唤醒

xv6 的 [`sleep_prepare`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L547)、[`sleep`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L562) 和 [`wakeup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L576) 实现了上面例子中使用的接口。基本思路是：`sleep` 把当前进程标记为 `SLEEPING`，然后调用 `sched` 释放 CPU；`wakeup` 查找在给定等待通道上睡眠的进程，并把它们标记为 `RUNNABLE`。

`sleep_prepare(chan)` 把等待通道记录在 `p->chan` 中，然后返回。如果调用 `sleep()` 时 `p->chan` 仍然非零，`sleep()` 就把调用进程的状态改为 `SLEEPING`，并调用 `sched()` 让其他进程运行。

对于另一个可能正在并发地准备在 `chan` 上睡眠的进程，`wakeup(chan)` 要处理两种情况。如果那个进程已经进行到把自己标记为 `SLEEPING` 这一步，`wakeup` 就会看到它处于该状态，并把它改为 `RUNNABLE`；随后它对 `sleep()` 的调用就会返回。如果那个进程只进行到调用了 `sleep_prepare(chan)`（而它的 `sleep()` 还没有进行到获取 `p->lock`），`wakeup(chan)` 会看到 `p->chan` 等于 `chan`，于是 `wakeup` 把 `p->chan` 设为零。这种情况下，当那个进程真正调用 `sleep` 时，它会看到零值并立即返回。这样做的目的是：如果 `wakeup()` 恰好在另一个进程即将调用 `sleep()` 时被调用，也不会丢失唤醒。

`sleep_prepare()` 的使用规则让我们不必担心即将睡眠的进程尚未调用 `sleep_prepare()` 的情况。要么该进程用一个锁把条件检查和 `sleep_prepare()` 包起来，这个锁会阻止对相关通道的任何 `wakeup()` 调用；要么该进程必须在检查条件之前调用 `sleep_prepare()`。

有时多个进程在同一个通道上睡眠；例如，有多个进程在读同一个管道。一次 `wakeup` 调用会把它们全部唤醒。其中一个会先运行，并（在管道的情况下）获取管道锁，读走所有在等待的数据。其他进程会发现，虽然被唤醒了，却没有数据可读。从它们的角度看，这次唤醒是“**虚假的**”（spurious），它们必须再次睡眠。因此，睡眠总是发生在一个重新检查条件的循环中，就像上面的 `piperead` 那样。

如果两处 sleep/wakeup 的使用碰巧选了同一个通道，也不会造成危害：它们会遇到虚假唤醒，但如上所述的循环可以容忍这个问题。sleep/wakeup 的魅力很大程度上在于它既轻量（不需要创建特殊的数据结构来充当等待通道），又提供了一层间接性（调用者不需要知道自己在和哪个具体进程交互）。

## 9.3 代码：管道

xv6 的管道是用睡眠来同步生产者和消费者的一个代码实例。我们在[第 1 章](https://mit-pdos.github.io/xv6-riscv-book/unix.html#CH:UNIX)中见过管道的接口：写入管道一端的字节被复制到内核中的缓冲区，然后可以从管道的另一端读出。后续章节会考察围绕管道的文件描述符支持，现在先来看 `pipewrite` 和 `piperead` 的实现。

每个管道用一个 `struct pipe` 表示，其中包含一个 `lock` 和一个 `data` 缓冲区。字段 `nread` 和 `nwrite` 分别统计从缓冲区读出和写入缓冲区的总字节数。缓冲区是环形的：在 `buf[PIPESIZE-1]` 之后写入的下一个字节是 `buf[0]`。而计数不会回绕。这一约定让实现能够区分满缓冲区（`nwrite` `==` `nread+PIPESIZE`）和空缓冲区（`nwrite` `==` `nread`），但这也意味着访问缓冲区时必须使用 `buf[nread` `%` `PIPESIZE]`，而不能直接用 `buf[nread]`（`nwrite` 同理）。

假设 `piperead` 和 `pipewrite` 的调用同时发生在两个不同的 CPU 上。[`pipewrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L77) 首先获取管道的锁，该锁保护计数、数据以及与之相关的不变量。接着 [`piperead`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L112) 也试图获取这个锁，但获取不到。它在 [`acquire`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/spinlock.c#L22) 中自旋，等待锁。在 `piperead` 等待期间，`pipewrite` 循环处理要写入的字节（`addr[0..n-1]`），依次把每个字节加入管道[（pipe.c:101）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L101)。在这个循环中，缓冲区可能会被填满[（pipe.c:88）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L88)。这种情况下，`pipewrite` 调用 `wakeup`，通知所有睡眠的读者缓冲区中有数据在等待，然后释放锁，并在通道 `&pi->nwrite` 上睡眠，等待读者从缓冲区中取走一些字节。

此时 `piperead` 获取到管道的锁并进入临界区：它发现 `pi->nread` `!=` [`pi->nwrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L119)（`pipewrite` 进入睡眠是因为 `pi->nwrite` `==` `pi->nread` `+` [`PIPESIZE`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L88)），于是继续执行到 `for` 循环，把数据从管道中复制出来[（pipe.c:129）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L129)，并把 `nread` 增加所复制的字节数。缓冲区中现在有这么多空间可供写入了，所以 `piperead` 在返回前调用 [`wakeup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L140) 唤醒所有睡眠的写者。`wakeup` 找到一个在 `&pi->nwrite` 上睡眠的进程（就是那个运行 `pipewrite`、但因缓冲区满而停下的进程）。`wakeup` 把这个进程标记为 `RUNNABLE`。

管道代码为读者和写者使用不同的等待通道（`&pi->nread` 和 `&pi->nwrite`）；当有大量读者和写者在等待同一个管道时，这可能会让系统更高效。管道代码在一个检查睡眠条件的循环中睡眠；如果有多个读者或写者，除了第一个被唤醒的进程，其余进程都会发现条件不成立，并再次睡眠。

## 9.4 代码：wait、exit 和 kill

请阅读 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中函数 `kwait()`、`kexit()` 和 `kkill()` 的代码；它们是相应系统调用的内部实现。

[第 1 章](https://mit-pdos.github.io/xv6-riscv-book/unix.html#CH:UNIX)中介绍过的子进程 `exit` 与父进程 `wait` 之间的交互，是一个有趣的睡眠实例。子进程死亡时，父进程可能已经在 `wait` 中睡眠，也可能在做别的事情；在后一种情况下，之后对 `wait` 的调用必须能察觉到子进程的死亡，这可能是子进程调用 `exit` 很久之后的事。xv6 记录子进程死亡、直到 `wait` 察觉的方式是：`exit` 把调用者置为 `ZOMBIE` 状态，它会一直停留在这个状态，直到父进程的 `wait` 注意到它，把子进程的状态改为 `UNUSED`，复制子进程的退出状态，并把子进程的进程 ID 返回给父进程。如果父进程先于子进程退出，父进程会把子进程交给 `init` 进程，而 `init` 永远在调用 `wait`；这样每个子进程都有一个父进程来为它善后。难点在于避免父进程 `wait` 与子进程 `exit` 同时发生、以及 `exit` 与 `exit` 同时发生时的竞争和死锁。

`kwait` 是 `wait` 的内核实现，它首先获取 [`wait_lock`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L370)，这个锁充当条件锁，帮助确保 `kwait` 不会错过正在退出的子进程发出的 `wakeup`。然后 `kwait` 扫描进程表。如果找到处于 `ZOMBIE` 状态的子进程，它就释放该子进程的资源及其 `proc` 结构体，把子进程的退出状态复制到传给 `wait` 的地址（如果该地址不为 0），并返回子进程的进程 ID。如果 `kwait` 找到了子进程但都还没有退出，它就睡眠，等待其中任意一个退出[（proc.c:414）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L414)，然后再次扫描。`kwait` 经常同时持有两个锁：`wait_lock` 和某个进程的 `pp->lock`；避免死锁的顺序是先 `wait_lock` 后 `pp->lock`。

[`kexit`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L326) 记录退出状态，释放部分资源，调用 `reparent` 把所有子进程交给 `init` 进程，唤醒可能正在 `wait` 中的父进程，把调用者标记为僵尸，并永久让出 CPU。`kexit` 在这一系列操作期间同时持有 `wait_lock` 和 `p->lock`。它持有 `wait_lock`，是因为这是 `wakeup(p->parent)` 的条件锁，可以防止在 `wait` 中的父进程丢失唤醒。`kexit` 在这一系列操作中也必须持有 `p->lock`，以防在 `wait` 中的父进程在子进程最终调用 `swtch` 之前就看到子进程处于 `ZOMBIE` 状态。`kexit` 以与 `kwait` 相同的顺序获取这两个锁，以避免死锁。

`kexit` 在把自己的状态设为 `ZOMBIE` 之前就唤醒父进程，这看起来可能不对，但其实是安全的：虽然 `wakeup` 可能让父进程开始运行，但父进程 `kwait` 中的循环在子进程的 `p->lock` 被 `scheduler` 释放之前无法检查该子进程，所以 `kwait` 要等到 `kexit` 把状态设为 [`ZOMBIE`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L358) 之后才能查看这个正在退出的进程。

`exit` 让进程终止自己，而 `kill` 系统调用[（proc.c:601）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L601)则让一个进程请求另一个进程终止。让 `kill` 直接销毁目标进程会过于复杂，因为目标进程可能正在另一个 CPU 上执行，也许正处在对内核数据结构进行一系列敏感更新的过程中。所以 `kkill` 做的事情很少：它只是设置目标进程的 `p->killed`，如果目标进程在睡眠，就唤醒它。目标进程最终会进入或离开内核，那时如果 `p->killed` 已被设置，`usertrap` 中的代码就会调用 `kexit`（它通过调用 [`killed`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L633) 来检查）。如果目标进程正在用户空间运行，它会在下一次进入内核时（通过发起系统调用，或者因为定时器或其他设备中断）发现自己已被杀死。

如果目标进程正在睡眠，`kkill` 会把它的状态设为 `RUNNABLE`，让它从 `sleep` 返回。这有潜在的危险，因为它所等待的条件可能并不成立。不过，xv6 对 `sleep` 的调用总是包在一个 `while` 循环中，`sleep` 返回后会重新检测条件。有些 `sleep` 调用还会在循环中检测 `p->killed`，如果它已被设置，就放弃当前的操作。只有在放弃操作是正确的情况下才会这样做。例如，管道读写代码[（pipe.c:84）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L84)在 killed 标志被设置时会返回；代码最终会返回到 trap，trap 会再次检查 `p->killed` 并退出。

有些 xv6 的 `sleep` 循环不检查 `p->killed`，因为这些代码处在一个应当原子执行的多步系统调用中间（也就是说，中途放弃就会出错）。virtio 驱动[（virtio\_disk.c:287）](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/virtio_disk.c#L287)就是一个例子：它不检查 `p->killed`，因为一次磁盘操作可能是一组写操作中的一个，而这组写操作必须全部完成，文件系统才能处于正确的状态。在等待磁盘 I/O 时被杀死的进程，要等到它完成当前系统调用、`usertrap` 看到 killed 标志后才会退出。

## 9.5 进程锁

与每个进程关联的锁（`p->lock`）是 xv6 中最复杂的锁。理解 `p->lock` 的一个简单方式是：读写 `struct proc` 的以下任一字段时都必须持有它：`p->state`、`p->chan`、`p->killed`、`p->xstate` 和 `p->pid`。这些字段可能被其他进程或其他 CPU 上的调度器线程使用，所以自然要用锁来保护。

不过，`p->lock` 的大多数用途是保护 xv6 进程数据结构和算法中更高层的不变量。下面是 `p->lock` 所做的全部事情：

- 它与 `p->state` 一起，防止为新进程分配 `proc[]` 槽位时发生竞争。
- 在进程创建或销毁期间，它把进程隐藏起来，不让外界看到。
- 它防止父进程的 `wait` 回收一个已把状态设为 `ZOMBIE` 但尚未让出 CPU 的进程。
- 它防止另一个 CPU 的调度器在一个让出 CPU 的进程把状态设为 `RUNNABLE` 之后、完成 `swtch` 之前，就决定运行它。
- 它确保只有一个 CPU 的调度器决定运行某个 `RUNNABLE` 进程。
- 它防止定时器中断导致进程在 `swtch` 中时让出 CPU。
- 它防止 `sleep` 和 `wakeup` 之间的竞争。
- 它防止 `kill` 的目标进程在 `kkill` 检查 `p->pid` 与设置 `p->killed` 之间退出，甚至被重新分配。
- 它使 `kkill` 对 `p->state` 的检查和写入成为原子操作。

`p->parent` 字段由全局锁 `wait_lock` 而不是 `p->lock` 保护。只有进程的父进程会修改 `p->parent`，不过这个字段既会被进程自己读取，也会被正在寻找自己子进程的其他进程读取。`wait_lock` 的作用是在 `wait` 睡眠等待任意子进程退出时充当条件锁。正在退出的子进程会一直持有 `wait_lock` 或 `p->lock`，直到它把状态设为 `ZOMBIE`、唤醒父进程并让出 CPU 之后。`wait_lock` 还会把父进程和子进程并发的 `exit` 串行化，从而保证 `init` 进程（它会继承该子进程）一定会从 `wait` 中被唤醒。`wait_lock` 是一个全局锁，而不是每个父进程各有一个的每进程锁，因为进程在获取它之前无法知道自己的父进程是谁。

## 9.6 现实世界

`sleep` 和 `wakeup` 是一种简单有效的同步方法，但同步方法还有很多，信号量（semaphore）（Dijkstra 1965）就是一例。所有这些方法面临的第一个难题，都是避免本章开头看到的“丢失唤醒”问题。最初的 Unix 内核的 `sleep` 只是禁用中断，这就足够了，因为 Unix 运行在单 CPU 系统上。Plan 9 的 `sleep` 使用一个回调函数，该函数在即将睡眠前、持有调度锁的情况下运行；它充当对睡眠条件的最后一刻检查，以避免丢失唤醒。Linux 内核的 `sleep` 使用一个显式的进程队列，称为**等待队列**（wait queue），而不是等待通道；该队列有自己的内部锁。

在 `wakeup` 中扫描整个进程集合效率不高。更好的办法是把 `sleep` 和 `wakeup` 中的 `chan` 换成一个数据结构，其中保存着在该结构上睡眠的进程列表，例如 Linux 的等待队列。Plan 9 的 `sleep` 和 `wakeup` 把这种结构称为**会合点**（rendezvous point）。许多线程库把同样的结构称为**条件变量**（condition variable）；在这种语境下，`sleep` 和 `wakeup` 操作被称为 `wait` 和 `signal`。所有这些机制都有相同的特点：睡眠条件由某种锁保护，而这个锁在睡眠过程中被原子地释放。

xv6 的 `wakeup` 会唤醒在某个等待通道上等待的所有进程。如果有多个这样的进程，它们都会尝试获取条件锁并重新检查条件；很多情况下只有一个进程能做有用的事（例如读走管道中等待的全部数据）。其余进程会发现条件已不再成立，于是重新睡眠；唤醒它们白白浪费了 CPU 时间。因此，大多数条件变量的设计都提供两个原语：`signal` 唤醒等待该条件变量的进程中的一个，`broadcast` 则唤醒全部进程。

强制杀死进程会带来一些问题。例如，被杀死的进程可能正深陷内核中睡眠，展开它的栈需要小心，因为调用栈上的每个函数可能都需要做一些清理工作。有些语言提供了异常机制来帮忙，但 C 语言没有。此外，还有其他事件可能导致睡眠的进程被唤醒，即使它所等待的事件还没有发生。例如，当一个 Unix 进程在睡眠时，另一个进程可能会向它发送一个 `signal`。这种情况下，该进程会从被中断的系统调用返回，返回值为 -1，错误码设为 EINTR。应用程序可以检查这些值并决定如何处理。xv6 不支持信号，所以不会出现这种复杂情况。

xv6 对 `kill` 的支持并不完全令人满意：有些睡眠循环可能应该检查 `p->killed`。一个相关的问题是，即使是检查 `p->killed` 的 `sleep` 循环，`sleep` 和 `kill` 之间也存在竞争：后者可能恰好在目标进程的循环检查完 `p->killed` 之后、调用 `sleep` 之前，设置 `p->killed` 并试图唤醒目标进程。如果发生这种情况，目标进程要等到它所等待的条件出现时才会注意到 `p->killed`。这可能要晚得多，甚至永远不会发生（例如目标进程在等待控制台输入，而用户一直不输入）。

## 9.7 练习

1. 在 xv6 中实现计数信号量。挑选 xv6 中几处使用 sleep 和 wakeup 的地方，把它们换成信号量。评价一下结果。
2. 你能否实现一种 sleep/wakeup 方案，只需要调用 `sleep()`（也许带额外参数），而不需要类似 `sleep_prepare()` 这样的调用？
3. 修复上面提到的 `kill` 与 `sleep` 之间的竞争，使得在目标进程的睡眠循环检查 `p->killed` 之后、调用 `sleep` 之前发生的 `kill`，能让目标进程放弃当前的系统调用。
4. 设计一个方案，让每个睡眠循环都检查 `p->killed`，这样，例如一个处在 virtio 驱动中的进程如果被另一个进程杀死，就能迅速从 while 循环中返回。
