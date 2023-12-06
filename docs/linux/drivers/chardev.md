# 字符设备驱动程序

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

如果我们提前知道所需要的设备编号，那么使用*register_chrdev_region*就够了。但是在大部分情况下，不推荐这么做，而应该使用动态分配函数。

```C
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
```
dev用来保存你要申请的那个设备号变量， baseminor是次设备号的起始值，通常是0。

<q>
驱动程序应该始终使用*alloc_chrdev_region*而不是*register_chrdev_region* ————《Linux设备驱动程序*P50*》
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

内核使用*struct cdev*结构来表示字符设备，

## 内存映射

### 驱动层的操作

由于Linux有MMU模块，因此无法访问真实的物理地址，只能通过虚拟地址访问硬件外设。所以需要有一个函数可以在物理地址和虚拟地址之间进行转换。

关于如何获取硬件的物理地址，请参考[设备树](../kernel/others.md)。

    ioremap(phy_addr, size)

其中addr就是你要访问的直接物理地址，size是需要转化的大小，返回值就是虚拟地址，这个虚拟地址我们需要专门的数据类型去接收。

    static void __iomem *v_addr

**__iomem**是一个宏，主要就是在硬件寄存器的物理地址和程序中使用的虚拟地址之间进行转换。

有映射就一定有取消映射，用到的是这么一个函数。

    iounmap(v_addr)

这里的参数一定是你自己定义的用来接收虚拟地址的那个指针，注意不要和物理地址搞混了。

### I/O内存访问函数

使用ioremap函数将寄存器物理地址映射到虚拟地址后，Linux推荐使用内核自带的读写操作函数对映射后的内存进行读写。

- 读操作函数

```c
u8 readb(v_addr)
u16 readw(v_addr)
u32 readl(v_addr)
```

上面三个函数分别对应8位，16位，32位的读操作，addr是要读取的内存地址，返回值是读取到的数据

- 写操作函数

```c
void writeb(u8 value, v_addr)
void writew(u16 value, v_addr)
void writel(u32 value, v_addr)
```

同理，上面三个函数分别对应各自位的读操作函数。

拿到虚拟地址后，我们需要根据手册说明将其中的某几位进行配置，在配置之前先清除以前的配置是一个好习惯。

```C
val = readl(v_addr);  //从地址中读取值写入val
val &= ~(3 << 26);    //对bit26, 27位清零
val |= 3 << 26;       //对bit26, 27位置1
write(val, v_addr);   //写入虚拟地址
```

### 应用层的操作

由于Linux内核禁止用户态和内核态直接进行数据交互，我们只能通过内核提供的API函数进行。

```C
unsigned long copy_from_user(void *to, const void *from, unsigned long count)

unsigned long copy_to_user(void *to, const void *from, unsigned long count)
```

## 自动创建设备号

struct cdev是内核和设备间的借口，内核提供了以下三个函数用于操作cdev结构体。

```C
void cdev_init(struct cdev *cdev, struct file_operations *fops)
int cdev_add(struct cdev *cdev, dev_t num, unsigned int count)
void cdev_del(struct cdev *cdev)
```

scull中的注册

```C
cdev_init(&dev->cdev, &scull_fops);
dev->cdev.owner = THIS_MODULE;
dev->cdev.ops = &scull_fops;
ret = cdev_add(&dev->cdev, devno, 1);
```

原理与前面相同，初始化+注册。

