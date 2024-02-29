# 直接内存访问

直接内存访问(DMA)是一种高级的硬件机制，它允许外围设备和内存之间直接传输它们的I/O数据，而不需要CPU的参与。使用这种机制可以大大提高与设备通信的效率。

## 交换数据

1.单变量访问：

```C

get_user(type val, type *address)

put_user(type val, type *address)

```

2.数组访问：

```C
unsigned long copy_to_user(void *to, const void *from, unsigned long n)

unsigned long copy_from_user(void *to, const void *from, unsigned long n)
```

3.内存映射I/O访问

获取I/O虚拟地址：

```C
void __iomem *devm_ioremap(struct device *dev, unsigned long offset, unsigned long size)

void devm_iounmap(struct device *dev, void __iomem *addr)
```

访问I/O虚拟地址：

```C
unsigned int ioread8(void __iomem *addr)
unsigned int ioread16(void __iomem *addr)
unsigned int ioread32(void __iomem *addr)

void iowrite8(u8 value, void __iomem *addr)
void iowrite16(u16 value, void __iomem *addr)
void iowrite32(u32 value, void __iomem *addr)
```


