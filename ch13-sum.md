# 第 13 章 总结

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/sum.html>。原书采用 CC BY 3.0 许可。

本书通过逐行研读一个操作系统 xv6，介绍了操作系统的主要思想。有些代码行体现了这些主要思想的精髓（例如上下文切换、用户/内核边界、锁等），每一行都很重要；另一些代码行则展示了如何实现某个具体的操作系统思想，而这些地方很容易换一种方式来做（例如更好的调度算法、表示文件的更好的磁盘数据结构、允许并发事务的更好的日志等）。所有这些思想都是在一个特定的、非常成功的系统调用接口，即 Unix 接口的背景下阐述的，但这些思想同样适用于其他操作系统的设计。
