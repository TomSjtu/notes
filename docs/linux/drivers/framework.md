# 驱动框架

> 驱动框架下各类子系统的介绍。

## Pinctrl子系统

通过寄存器映射来操作外设的编程方式，在驱动开发中是不建议的，因为每当芯片的寄存器发生变化，那么底层的驱动代码就得推翻重写。在驱动开发中，有没有办法可以不涉及到具体的寄存器操作呢？对于有些外设，是具备抽象条件的，Pintrl子系统就是其中之一。

Pinctrl：Pin Controller，引脚控制：

- 引脚枚举与命名
- 引脚复用：一个引脚可以用作GPIO，也可以用作I2C或UART
- 引脚配置：比如上拉、下拉

Pinctrl驱动由芯片厂家BSP工程师提供，一般的驱动工程师只需要在设备树里描述即可：

- 指明使用哪些引脚
- 复用为什么功能
- 配置为什么状态

IOMUX， IO复用器


### Pinctrl主要数据结构



pinctrl_desc:

```C
struct pinctrl_pin_desc {
    unsigned number;
    const char *name;
    void *drv_data;
};
```

