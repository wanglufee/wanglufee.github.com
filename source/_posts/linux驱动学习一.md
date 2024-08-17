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

# linux内核模块程序结构
一般内核模块主要由以下几部分组成
- 模块加载函数： 当使用 `insmod` 或 `modprobe` 命令加载模块时，模块加载函数被调用，完成初始化工作。
- 模块卸载函数： 当使用 `rmmod` 命令卸载模块时，会调用模块卸载函数，完成卸载功能。
- 模块许可声明： 声明内核模块的许可权限。(大多数情况下license为 GPL v2)
- 模块参数(可选)： 模块加载时可传递给模块的值。
- 模块导出符号(可选)： 内核模块导出的符号，若导出则其他模块可以使用本模块中的变量或函数。
- 模块作者等信息声明(可选)
## 模块加载函数
```c
static int _ _init initialization_function(void)
{
    /* 初始化代码 */
}
module_init(initialization_function);
```
模块加载函数一般以 **__init** 声明，而模块加载函数以 **module_init(函数名)** 的形式被指定，它返回整形值，模块初始化成功时返回0。失败则返回相应的错误码。

还可以使用 **request_module(module_name)** 来灵活加载其他内核模块。
## 模块卸载函数
```c
static void _ _exit cleanup_function(void)
{
      /* 释放代码 */
}
module_exit(cleanup_function);
```
内核卸载函数一般以 **__exit** 声明，模块卸载函数在模块卸载时执行，不返回任何值，必须以 **module_exit(函数名)** 的形式来指定。若模块被编译进内核，则卸载函数会被省略。
## 模块参数
```c
static char *book_name = "dissecting Linux Device Driver";
module_param(book_name, charp, S_IRUGO);
static int book_num = 4000;
module_param(book_num, int, S_IRUGO);
```
可以使用 **modele_param(参数名,参数类型,参数读/写权限)** 为模块定义一个参数。 在加载模块时可以向模块传递参数 `insmod 模块名 参数名=参数值`。如果不传递，则使用定义的缺省值。 参数的类型可以为
**byte、short、ushort、int、uint、long、ulong、charp（字符指针）​、bool或invbool（布尔的反）**。

此外模块还可以拥有参数数组，可以通过 `module_param_array(数组名,数组类型,数组长,参数读/写权限)` 来定义。

模块被加载后在 /sys/module 目录下会出现以模块名命名的目录，如果包含可读写的参数时，目录下还将包含 parameters 目录，其中包含参数名的一系列文件节点。
## 导出符号
```c
EXPORT_SYMBOL(符号名);
EXPORT_SYMBOL_GPL(符号名);
```
模块可以使用上述两个宏导出符号，其他模块使用时只需要提前声明即可。 **EXPORT_SYMBOL_GPL**导出的符号只适用于包含GPL许可权的模块。
## 模块声明与描述
```c
MODULE_AUTHOR(author);
MODULE_DESCRIPTION(description);
MODULE_VERSION(version_string);
MODULE_DEVICE_TABLE(table_info);
MODULE_ALIAS(alternate_name);
```
上述宏分别用于声明模块的作者、描述、版本、设备表和别名。