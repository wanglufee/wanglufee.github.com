---
title: linux驱动学习二
date: 2024-04-17 15:41:05
tags: linux驱动学习
categories: linux
---
# linux文件系统与设备文件
linux下一切皆文件，而驱动同样也需要和文件系统交互，这章来简单看下文件系统与设备文件。
## linux文件操作
### 创建
```c
int creat(const char *filename, mode_t mode);
```
其中参数 mode 指定新建文件的权限，与 umask 一起决定文件的最终权限。 而 umask 可以通过以下函数来改变。
```c
int umask(int newmask);
```
该函数将 umask 设置为 newmask 返回旧的 umask。
### 打开
```c
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```
其中 pathname 是要打开的文件名，包含路径，默认是当前目录下， flags 文件打开的标志。 如果标志选择创建文件，那么需要指定 mode 参数。
### 读写
```c
int read(int fd, const void *buf, size_t length);
int write(int fd, const void *buf, size_t length);
```
其中 buf 为指向缓冲区的指针， length 为缓冲区大小(以字节为单位)。read 函数返回实际读到的字节数，write 返回实际写入的字节数。
### 定位
```c
int lseek(int fd, offset_t offset, int whence);
```
lseek 将文件读写指针相对 whence 移动 offset 字节。成功时返回文件指针相对文件头的位置。whence 可选 `SEEK_SET` 相对文件头， `SEEK_CUR` 相对文件读写指针的当前位置， `SEEK_END` 相对文件末尾。
### 关闭
```c
int close(int fd);
```
关闭文件只需要调用close函数，传入文件描述符即可。
## c库文件操作
c 库文件操作独立于具体的操作系统平台。
### 创建和打开
```c
fiLE *fopen(const char *path, const char *mode);
```
用于打开指定文件，mode 为模式。 mode 模式中的 `b` 用于区分二进制文件和文本文件，在DOS,Windows中由区分，linux中无区分。
### 读写
C库函数支持以字符、字符串等为单位，支持按照某种格式进行文件的读写
```c
int fgetc(fiLE *stream);
int fputc(int c, fiLE *stream);
char *fgets(char *s, int n, fiLE *stream);
int fputs(const char *s, fiLE *stream);
int fprintf(fiLE *stream, const char *format, ...);
int fscanf (fiLE *stream, const char *format, ...);
size_t fread(void *ptr, size_t size, size_t n, fiLE *stream);
size_t fwrite (const void *ptr, size_t size, size_t n, fiLE *stream);
```
c 库函数还提供了读写过程中的定位能力
```c
int fgetpos(fiLE *stream, fpos_t *pos);
int fsetpos(fiLE *stream, const fpos_t *pos);
int fseek(fiLE *stream, long offset, int whence);
```
### 关闭
```c
int fclose (fiLE *stream);
```
关闭依然为 close。
## linux文件系统
