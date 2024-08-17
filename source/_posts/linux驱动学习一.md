---
title: linux驱动学习一
date: 2024-04-13 20:03:49
tags: linux驱动学习
categories: linux
---
# linux内核编译
> linux 内核配置

比较推荐的是通过命令 `make menuconfig` 命令来进行配置，它不依赖于QT或GTK+，且非常直观，运行此命令配置之后，会生成一个 .config 配置文件，记录哪些部分被编译进内核，哪些部分被编译为模块，或者不编译。

linux 配置系统由三部分组成
- Makefile: 分布在Linux内核源代码中，定义Linux内核的编译规则。
- Kconfig: 给用户提供配置选择的功能
- 配置工具: 包括配置命令解释器和配置用户界面

首先来看 Kconfig 文件

```bash
config TTY_PRINTK
       tristate "TTY driver to output user messages via printk"
       depends on EXPERT && TTY
       default n
       ---help---
        If you say Y here, the support for writing user messages (i.e.
        console messages) via printk is available.
        The feature is useful to inline user messages with kernel
        messages.
        In order to use this feature, you should output user messages
        to /dev/ttyprintk or redirect console to this TTY.
        If unsure, say N.
```
config 后跟的是配置项的符号， tristate 代表这个选项是一个三态选项，分别为 Y , N , M,其中 Y 表示编译进内核中， N 表示不进行编译， M 表示编译成内核模块，后面的字符则是在配置界面显示的内容。此处除了 tristate 还有 bool 等。 depends on 则表示这个模块依赖的模块有哪些，多个模块用 `&&` 来连接， default 表示默认值。 help 下跟的则是在配置中的帮助信息。

接下来看 Kbuild Makefile 文件
```bash
obj-$(CONfiG_ISDN) += isdn.o
obj-$(CONfiG_ISDN_PPP_BSDCOMP) += isdn_bsdcomp.o
```
此处 obj-$(xxx) 其中的 xxx 即为配置项符号， 命名的方式是在 Kconfig 的符号前加了 CONFIG_ 字段， 当其值为 y 时，会根据后面的 .o 文件去找对应的 .c 或 .s 文件，编译进内核中，当为 n 时，则不进行编译， 为 m 时，则寻找对应的文件，编译为模块的形式， 这里对应了用户配置时的选择。 

```bash
#
# Makefile for the linux ext2-filesystem routines.
#
obj-$(CONfiG_EXT2_FS) += ext2.o
ext2-y := balloc.o dir.o file.o fsync.o ialloc.o inode.o \
       ioctl.o namei.o super.o symlink.o
```
若一个模块由多个文件构成，则会去找对应的 .o 文件，最终链接生成最终的 .o 文件。此处若配置了 ext2.o ，则会去找 balloc.o  dir.o 等文件，最终生成 ext2.o 。
```bash
obj-$(CONfiG_EXT2_FS) += ext2/
```
目录层次，当配置为 y 或者 m 时，会迭代 ext2 目录。