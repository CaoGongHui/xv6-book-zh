# xv6 教材中文翻译

《xv6: a simple, Unix-like teaching operating system》（RISC-V 版）的简体中文翻译。原书作者 Russ Cox、Frans Kaashoek、Robert Morris，是 MIT 6.1810（原 6.S081）操作系统课程的配套教材。

- 原书在线版：<https://mit-pdos.github.io/xv6-riscv-book/>
- xv6 源码：<https://github.com/mit-pdos/xv6-riscv>
- 课程网站：<https://pdos.csail.mit.edu/6.1810/>

## 目录

- [前言与致谢](00-foreword.md)
- [第 1 章 操作系统接口](ch01-unix.md)
- [第 2 章 操作系统的组织](ch02-first.md)
- [第 3 章 页表](ch03-mem.md)
- [第 4 章 陷阱与系统调用](ch04-trap.md)
- [第 5 章 缺页异常](ch05-pgfault.md)
- [第 6 章 中断与设备驱动](ch06-interrupt.md)
- [第 7 章 锁](ch07-lock.md)
- [第 8 章 调度](ch08-sched.md)
- [第 9 章 睡眠与唤醒](ch09-sleep.md)
- [第 10 章 文件系统](ch10-fs.md)
- [第 11 章 日志](ch11-log.md)
- [第 12 章 再谈并发](ch12-lock2.md)
- [第 13 章 总结](ch13-sum.md)
- [参考文献](references.md)

## 翻译说明

- 依据 2026 年 11 月的在线版翻译，与 xv6-riscv 当前代码对应（例如系统调用 `pause`，旧版叫 `sleep`）。
- 代码片段保持原样；术语首次出现时附英文原文。
- 源码链接和插图直接引用原书网站和 GitHub。
- 发现错误欢迎提 Issue 或 PR。

## 许可

原书以 [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/) 许可发布，本译文同样以 CC BY 3.0 发布，版权归原作者所有。
