# 字符设备驱动程序

字符设备驱动框架如图所示：

![Alt text](../../images/kernel/chrdev.png)

在创建一个字符设备的时候，首先应该向内核申请一个设备号。拿到设备号之后，需要手动实现file_operation结构体中的函数指针，并保存到cdev结构体中。然后使用`cdev_add()`函数注册cdev。

注销设备时需要释放内核中的cdev，并且归还申请的设备号。


## 快速参考

```C
#include <linux/types.h>

dev_t devID

int MAJOR(dev_t dev)

int MINOR(dev_t dev)

dev_t MKDEV(unsigned int major, unsigned int minor)
```

```C
#include <linux/fs.h>

int register_chrdev_region(dev_t first, unsigned int count, char *name)

int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name)

void unregister_chrdev_region(dev_t first, unsigned int count)

int register_chrdev(unsigned int major, const char *name, struct file_operations *fops)

int unregister_chrdev(unsigned int major, const char *name)

struct file_operations

struct file

struct inode
```

```C
#include <linux/cdev.h>

struct cdev *cdev_alloc(void)

void cdev_init(struct cdev *dev, dev_t num, unsigned int count)

int cdev_add(struct cdev *cdev, dev_t num, unsigned int count)

void cdev_del(struct cdev *dev)
```

```C
#include <linux/kernel.h>

container_of(pointer, type, field)

#include <asm/uaccess.h>

unsigned long copy_from_user(void *to, const void *from, unsigned long count)

unsigned long copy_to_user(void *to, const void *from, unsigned long count)
```

## 设备号初始化

### 设备号的注册与卸载(不推荐的做法)

- 注册
```C
int register_chrdev(unsigned int major, const char *name, struct file_operations *fops)
```

这个方法在注册时需要手动指定主设备号，因此你必须事先知道哪个主设备号没有被占用，这在实际开发中难以推广，并且该函数还一次性占用了主设备号下的全部次设备号。

- 卸载
```C
int unregister_chrdev(unsigned int major, const char *name)
```

### 设备号的注册与卸载(推荐的做法)

- 注册

```C
int register_chrdev_region(dev_t first, unsigned count, const char *name)
```

first是要分配的设备编号的起始值，count是所请求的连续设备编号的个数，name是和该编号范围关联的设备名称，它将出现在/proc/devices和sysfs中。

如果我们提前知道所需要的设备编号，那么使用`register_chrdev_region()`就够了。但是在大部分情况下，不推荐这么做，而应该使用动态分配函数：

```C
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
```
dev用来保存你要申请的那个设备号变量， baseminor是次设备号的起始值，通常是0。

<q>
驱动程序应该始终使用`alloc_chrdev_region`而不是`register_chrdev_region` ————《Linux设备驱动程序*P50*》
</q>

- 卸载

上面两种方法注册的设备统一用以下函数卸载。

```C
void unregister_chrdev_region(dev_t first, unsigned count)
```

该函数通常在清除函数中调用。

这里我们给出一个完整的示例，假设我们需要注册一个名字为"test"的设备。

```C

int major;
int minor;
dev_t devid;

if(major){
    devid = MKDEV(major, 0);
    register_chrdev_region(devid, 1, "test");
}else {
    alloc_chrdev_region(&devid, 0, 1, "test");
    major = MAJOR(devid);
    minor = MINOR(devid);
}
```

对设备号操作时，使用到了三个宏定义，**MKDEV**, **MAJOR**, **MINOR**。当给定主设备号时，使用MKDEV来构建完整的devID，次设备号则一般选择0。

实际开发中，major可以用一个宏定义取代，默认取0，即走else分支选择“动态分配”。用户可以选择使用默认值即动态分配的方式，或者自定义特定的设备号——只要在编译前修改宏定义为设备号，或者在insmod时指定即可。

## 字符设备的注册

内核使用**struct cdev**结构来表示字符设备，当我们自己实现file_operations结构体中的函数之后，我们需要使用`cdev_init()`函数来将文件操作指针与字符设备相关联。

```C
void cdev_init(struct cdev *cdev, const struct file_operations *fops)
```

`cdev_add()`函数用于向内核添加一个新的字符设备，`cdev_del()`函数用来删除。

```C
int cdev_add(struct cdev *p, dev_t dev, unsigned count)
void cdev_del(struct cdev *p)
```

## 内存映射

### 驱动层的操作

由于Linux有MMU模块，因此无法访问真实的物理地址，只能通过虚拟地址访问硬件外设。所以需要有一个函数可以在物理地址和虚拟地址之间进行转换。

关于如何获取硬件的物理地址，请参考[设备树](./dts.md)。

地址映射函数如下：
```C
void __iomem *ioremap(phys_addr_t paddr, unsigned long size)
```

其中addr就是你要访问的直接物理地址，size是需要转化的大小，返回值就是虚拟地址，这个虚拟地址我们需要专门的数据类型去接收。

取消地址映射函数：

```C
void iounmap(void *addr)
```

### I/O内存访问函数

使用ioremap函数将寄存器物理地址映射到虚拟地址后，Linux推荐使用内核自带的读写操作函数对映射后的内存进行读写。

```C
unsigned int ioread8(void __iomem *addr)
unsigned int iorea16(void __iomem *addr)
unsigned int ioread32(void __iomem *addr)

void iowrite8(u8 b, void __iomem *addr)
void iowrite16(u16 b, void __iomem *addr)
void iowrite32(u32 b, void __iomem *addr)
```

对于读I/O而言，输入参数为__iomem类型的指针，指向被映射后的地址，返回值为读取到的数据；对于写I/O而言，输入参数第一个为要写入的数据，第二个为要写入的地址，返回值为空。

下面一段代码演示了地址映射的相关操作：

```C
unsigned long pa_dr = 0x20A8000 + 0x00;
unsigned int __iomem *va_dr;
unsigned int val;
va_dr = ioremap(pa_dr, 4);
val = ioread32(va_dr);
val &= ~(0x01 << 19);
iowrite32(val, va_dr);
```

### 应用层的操作

由于Linux内核禁止用户态和内核态直接进行数据交互，我们只能通过内核提供的API函数进行。

```C
unsigned long copy_from_user(void *to, const void *from, unsigned long count)

unsigned long copy_to_user(void *to, const void *from, unsigned long count)
```


