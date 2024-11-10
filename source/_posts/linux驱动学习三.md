---
title: linux驱动学习三
date: 2024-04-28 21:58:14
tags: linux驱动学习
categories: linux
---
# 字符设备驱动
## 字符设备驱动结构
在 linux 内核中使用 cdev 结构体描述一个**字符设备**，结构体如下
```c
struct cdev {
  struct kobject kobj;             /* 内嵌的kobject对象 */
  struct module *owner;            /* 所属模块*/
  struct file_operations *ops;     /* 文件操作结构体*/
  struct list_head list;
  dev_t dev;                       /* 设备号*/
  unsigned int count;
};
```
其中 dev_t 成员定义了设备号，为32位，其中12位为主设备号，20位为次设备号，使用如下宏从 dev_t 获得主设备号和次设备号
```c
MAJOR(dev_t dev)
MINOR(dev_t dev)
```
而使用 MKDEV 宏则可以通过主设备号和次设备号生成 dev_t 成员。
```c
MKDEV(int major, int minor)
```
cdev 结构体的另一个重要成员 file_operations 定义了字符设备驱动提供给虚拟文件系统的接口函数。

linux 提供了一组函数用于操作 cdev 结构体
```c
void cdev_init(struct cdev *, struct file_operations *);
struct cdev *cdev_alloc(void);
void cdev_put(struct cdev *p);
int cdev_add(struct cdev *, dev_t, unsigned);
void cdev_del(struct cdev *);
```
cdev_init 函数用于初始化 cdev 的成员，并与 file_operations 建立连接。
```c
void cdev_init(struct cdev *cdev, struct file_operations *fops)
{
    memset(cdev, 0, sizeof *cdev); 
    INIT_LIST_HEAD(&cdev->list);
    kobject_init(&cdev->kobj, &ktype_cdev_default);
    cdev->ops = fops; /* 将传入的文件操作结构体指针赋值给cdev的ops*/
}
```
cdev_alloc 函数用于动态申请一个 cdev 内存
```c
struct cdev *cdev_alloc(void)
{
        struct cdev *p = kzalloc(sizeof(struct cdev), GFP_KERNEL);
        if (p) {
            INIT_LIST_HEAD(&p->list);
            kobject_init(&p->kobj, &ktype_cdev_dynamic);
        }
        return p;
}
```
cdev_add 函数和 cdev_del 函数分别向系统添加和删除一个cdev，完成字符设备的注册和注销。
## 分配和释放设备号
在调用 cdev_add 向系统注册字符设备之前，需要向系统申请设备号，通过调用 register_chrdev_region 或 alloc_chrdev_region 函数向系统申请设备号。
```c
int register_chrdev_region(dev_t from, unsigned count, const char *name);
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,const char *name);
```
register_chrdev_region 函数用于已知起始设备的设备号的情况，alloc_chrdev_region 用于设备号未知，动态向系统申请未被占用的设备号，函数调用成功后，将设备号放入第一个 dev 参数中。

相应的在调用 cdev_del 函数将字符设备从系统中销毁之后，应调用 unregister_chrdev_region 释放原先申请的设备号。
```c
void unregister_chrdev_region(dev_t from, unsigned count);
```
## file_operations结构体
file_operations 结构体中的成员函数是字符设备驱动设计的主体内容。这些函数实际会在**应用程序**调用 linux 的 open , write , read , close 等函数时最终被内核调用。
```c
struct file_operations {
  struct module *owner;
  loff_t (*llseek) (struct file *, loff_t, int);
  ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
  ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
  ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
  ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
  int (*iterate) (struct file *, struct dir_context *);
  unsigned int (*poll) (struct file *, struct poll_table_struct *);
  long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
  long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
  int (*mmap) (struct file *, struct vm_area_struct *);
  int (*open) (struct inode *, struct file *);
  int (*flush) (struct file *, fl_owner_t id);
  int (*release) (struct inode *, struct file *);
  int (*fsync) (struct file *, loff_t, loff_t, int datasync);
  int (*aio_fsync) (struct kiocb *, int datasync);
  int (*fasync) (int, struct file *, int);
  int (*lock) (struct file *, int, struct file_lock *);
  ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
  unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long,unsigned long, unsigned long);
  int (*check_flags)(int);
  int (*flock) (struct file *, int, struct file_lock *);
  ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *,  size_t, unsigned int);
  ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *,  size_t, unsigned int);
  int (*setlease)(struct file *, long, struct file_lock **);
  long (*fallocate)(struct file *file, int mode, loff_t offset,loff_t len);
  int (*show_fdinfo)(struct seq_file *m, struct file *f);
};
```
- llseek 函数用来修改一个文件的当前读写位置，并将新位置返回，在出错时，这个函数返回一个负值。
- read 函数用来从设备中读取数据，成功时函数返回读取的字节数，出错时返回一个负值。
- write 函数向设备发送数据，成功时该函数返回写入的字节数。如果此函数未被实现，当用户进行 write 系统调用时，将得到-EINVAL返回值。
- unlocked_ioctl 提供设备相关控制命令的实现（既不是读操作，也不是写操作）​，当调用成功时，返回给调用程序一个非负值。
- mmap 函数将设备内存映射到进程的虚拟地址空间中，如果设备驱动未实现此函数，用户进行 mmap 系统调用时将获得-ENODEV返回值。
- open 函数在用户调用 linux API 函数打开文件时被调用，如果驱动未实现这个函数，那么设备的打开操作永远成功。
- poll 函数一般用于询问设备是否可被非阻塞地立即读写。
## linux字符设备驱动的组成
### 字符设备驱动模块加载与卸载函数
在字符设备驱动模块加载函数中应该实现设备号申请与 cdev 注册，而在卸载函数中应实现设备号释放和 cdev 注销。
linux 在习惯上会定义一个设备结构体，包含设备设计的 cdev，私有数据以及锁等信息。
```c
/* 设备结构体
struct xxx_dev_t {
    struct cdev cdev;
  ...
} xxx_dev;
/* 设备驱动模块加载函数
static int _ _init xxx_init(void)
{
  ...
  cdev_init(&xxx_dev.cdev, &xxx_fops);         /* 初始化cdev */
  xxx_dev.cdev.owner = THIS_MODULE;
  /* 获取字符设备号*/
  if (xxx_major) {
      register_chrdev_region(xxx_dev_no, 1, DEV_NAME);
  } else {
      alloc_chrdev_region(&xxx_dev_no, 0, 1, DEV_NAME);
  }

  ret = cdev_add(&xxx_dev.cdev, xxx_dev_no, 1);  /* 注册设备*/
  ...
}
/* 设备驱动模块卸载函数*/
static void _ _exit xxx_exit(void)
{
  unregister_chrdev_region(xxx_dev_no, 1);      /* 释放占用的设备号*/
  cdev_del(&xxx_dev.cdev);                      /* 注销设备*/
 ...
}
```
### 字符设备驱动的file_operations结构体中的成员函数
file_operations 结构体中的函数是虚拟文件系统的接口，是用户空间对 linux 进行系统调用的最终落实。
```c
/* 读设备*/
ssize_t xxx_read(struct file *filp, char __user *buf, size_t count,loff_t*f_pos)   // filp 是文件结构体指针， buf 是用户空间地址，count 是要读的字节数， f_pos 是读的位置相对于文件开头的偏移
{
   ...
   copy_to_user(buf, ..., ...);   // 由于用户空间不宜直接访问内核空间，因此使用这个函数完成用户空间到内核空间的复制
   ...
}
/* 写设备*/
ssize_t xxx_write(struct file *filp, const char __user *buf, size_t count,loff_t *f_pos)
{
   ...
   copy_from_user(..., buf, ...);
   ...
}
/* ioctl函数 */
long xxx_ioctl(struct file *filp, unsigned int cmd,unsigned long arg)  // 
{
  ...
  switch (cmd) {
  case XXX_CMD1:
        ...
        break;
  case XXX_CMD2:
        ...
        break;
  default:
        /* 不能支持的命令 */
        return  - ENOTTY;
  }
  return 0;
}
```
其中用于内核空间和用户空间复制的函数原型为:
```c
unsigned long copy_from_user(void *to, const void _ _user *from, unsigned long count);
unsigned long copy_to_user(void _ _user *to, const void *from, unsigned long count);
```
其返回值为不能复制的字节数，如果复制成功则返回0。

如果需要复制的类型是简单类型则可以使用 put_user() 和 get_user()
```c
int val;                         /* 内核空间整型变量
...
get_user(val, (int *) arg);      /* 用户→内核，arg是用户空间的地址 */
...
put_user(val, (int *) arg);      /* 内核→用户，arg是用户空间的地址 */
```
内核空间在访问用户空间之前需要检查合法性。通过 access_ok(type,addr,size) 进行判断，以确认传入的缓存区确实属于用户空间。

检查合法性非常重要，在 copy_from_user() , copy_to_user() , get_user() , put_user() 函数中均进行了合法性检查。

在字符设备驱动中需要定义一个 file_operations 实例，将设备驱动函数赋值给 file_operations 成员。
```c
struct file_operations xxx_fops = {
    .owner = THIS_MODULE,
    .read = xxx_read,
    .write = xxx_write,
    .unlocked_ioctl= xxx_ioctl,
    ...
};
```
其中 xxx_fops 在 cdev_init(&xxx_dev.cdev,&xxx_fops) 的语句中与 cdev 建立联系。

## 总结
字符设备通常会使用 cdev 结构体来表示，其中包含了设备号以及设备对应的文件操作结构体，其中文件操作结构体中的函数是我们要实现的重点，这个结构体通过 cdev_init() 函数来与 cdev 联系起来，linux 中则习惯于使用一个结构体来包含 cdev ，以及设备需要使用到的私有数据和锁等。