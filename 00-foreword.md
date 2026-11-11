# 前言与致谢

> 译自《xv6: a simple, Unix-like teaching operating system》（Russ Cox、Frans Kaashoek、Robert Morris），原文：<https://mit-pdos.github.io/xv6-riscv-book/index.html>。原书采用 CC BY 3.0 许可。

作者：Russ Cox、Frans Kaashoek、Robert Morris

这是一份为操作系统课程准备的草稿。它通过研究一个名为 xv6 的示例内核，讲解操作系统的主要概念。xv6 以 Dennis Ritchie 和 Ken Thompson 的 Unix 第 6 版（v6）（Ritchie and Thompson 1974）为蓝本。xv6 大体沿用 v6 的结构和风格，但用 ANSI C（Kernighan 1988）实现，运行在多核 RISC-V（Patterson and Waterman 2017）上。

阅读本书时应同时阅读 xv6 的源码，这种方法受到 John Lions 的《UNIX 第 6 版注释》（Lions 2000）的启发；书中有指向 <https://github.com/mit-pdos/xv6-riscv> 上源码的超链接。更多关于 v6 和 xv6 的在线资源（包括若干基于 xv6 的实验作业），请参见 <https://pdos.csail.mit.edu/6.1810>。

我们在 MIT 的操作系统课程 6.828 和 6.1810 中使用了本书。感谢这些课程的教师、助教和学生，他们都直接或间接地为 xv6 做出了贡献。特别感谢 Adam Belay、Austin Clements 和 Nickolai Zeldovich。最后，感谢通过电子邮件向我们报告书中错误或提出改进建议的各位：Abutalib Aghayev, Sebastian Boehm, brandb97, Anton Burtsev, Raphael Carvalho, Tej Chajed,Brendan Davidson, Rasit Eskicioglu, Color Fuzzy, Wojciech Gac, Giuseppe, Tao Guo, Haibo Hao, Naoki Hayama, Chris Henderson, Robert Hilderman, Eden Hochbaum, Wolfgang Keller, Paweł Kraszewski, Henry Laih, Jin Li, Austin Liew, lyazj@github.com, Pavan Maddamsetti, Jacek Masiulaniec, Michael McConville, m3hm00d, Mes0903, miguelgvieira, Mark Morrissey, Muhammed Mourad, Harry Pan, Harry Porter, pr3pony, Siyuan Qian, Zhefeng Qiao, Askar Safin, Salman Shah, Huang Sha, Vikram Shenoy, Adeodato Simó, Ruslan Savchenko, Pawel Szczurko, Warren Toomey, tyfkda, tzerbib, unicornx, Vanush Vaswani, Chen Wang, Xi Wang, and Zou Chang Wei, Sam Whitlock, Qiongsi Wu, LucyShawYang, ykf1114@gmail.com, and Meng Zhou

如果你发现错误或有改进建议，请发邮件给 Frans Kaashoek 和 Robert Morris（kaashoek,rtm@csail.mit.edu）。
