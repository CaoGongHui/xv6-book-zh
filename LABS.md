# 实验与章节对照

把这个仓库放在 6.1810 实验仓库的 `book-zh/` 目录下，做实验时可以边读边写：

```bash
git clone git://g.csail.mit.edu/xv6-labs-2026
cd xv6-labs-2026
git clone https://github.com/CaoGongHui/xv6-book-zh.git book-zh
echo '/book-zh/' >> .git/info/exclude   # 让实验仓库忽略它，切换分支时也不受影响
```

`book-zh/` 是独立的 git 仓库，不会出现在实验仓库的 `git status` 和 `make zipball` 里。

每个实验在实验仓库里是一个分支，用 `git switch <分支名>` 切换（先提交当前实验的修改）。题目以[课程网站](https://pdos.csail.mit.edu/6.1810/2026/)为准，下面的阅读顺序是建议。

| 实验（分支） | 先读 | 相关源码 |
|---|---|---|
| util | [第 1 章 操作系统接口](ch01-unix.md) | `user/*.c`、`user/user.h`、`user/ulib.c` |
| syscall | [第 2 章 操作系统的组织](ch02-first.md)，[第 4 章](ch04-trap.md) 4.3–4.4 节 | `kernel/syscall.c`、`kernel/sysproc.c`、`user/usys.pl` |
| pgtbl | [第 3 章 页表](ch03-mem.md) | `kernel/vm.c`、`kernel/kalloc.c`、`kernel/memlayout.h` |
| traps | [第 4 章 陷阱与系统调用](ch04-trap.md) | `kernel/trap.c`、`kernel/trampoline.S` |
| cow | [第 5 章 缺页异常](ch05-pgfault.md) | `kernel/vm.c`（`uvmcopy`）、`kernel/trap.c` |
| net | [第 6 章 中断与设备驱动](ch06-interrupt.md) | `kernel/net.c`、`kernel/pci.c`，驱动写法可参考 `kernel/virtio_disk.c` |
| lock | [第 7 章 锁](ch07-lock.md)、[第 12 章 再谈并发](ch12-lock2.md) | `kernel/spinlock.c`、`kernel/kalloc.c`、`kernel/bio.c` |
| fs | [第 10 章 文件系统](ch10-fs.md)、[第 11 章 日志](ch11-log.md) | `kernel/fs.c`、`kernel/bio.c`、`kernel/log.c` |
| mmap（作业页面发布后才有分支） | [第 5 章](ch05-pgfault.md)、[第 10 章](ch10-fs.md) | `kernel/sysfile.c`、`kernel/vm.c` |

[第 8 章 调度](ch08-sched.md)和[第 9 章 睡眠与唤醒](ch09-sleep.md)没有单独的实验，但 traps、lock 实验都会用到，可以在做 traps 之前读。

## 常用命令（在 WSL 里实验仓库根目录执行）

```bash
make qemu                 # 编译并启动，Ctrl-a x 退出
make qemu-gdb             # 配合另一个终端里的 gdb 调试
./grade-lab-<实验名> <测试名>  # 只跑一个测试，例如 ./grade-lab-util sleep
make grade                # 跑当前实验的全部测试
```
