# 其他杂项

## 设备树

### 为什么需要设备树

具体可以参考Linus在2011年写的一个邮件：[this whole ARM thing is a f*cking pain in the ass](https://lkml.org/lkml/2011/3/17/492)。

简单来说就是，ARM社区充斥着大量垃圾的、重复的描述板级信息的代码（arch/arm/mach-xxx），导致Linux内核十分的臃肿。因为Linux只是一个操作系统内核，它并不关心你开发板上有什么设备。于是ARM社区引入了PowerPC等架构已经采用的设备树。将这些描述硬件信息的内容从Linux内核中剥离出来，用一个单独的文件进行描述，这就是设备树文件.dts的由来。一个SOC可以制作很多开发板，这些开发板必然有一些通用的信息。将这些通用的信息提取出来作为一个通用的文件，其他的.dts文件直接引用通用文件即可，类似于C语言中的头文件。这个通用文件就是.dtsi文件。

一般情况下，.dtsi描述SOC信息，比如CPU架构，主频。.dts文件描述板级信息，比如开发板上有哪些IIC设备、SPI设备等。

对于嵌入式系统，在系统启动阶段，由bootloader通过bootm命令将设备树信息传递给内核，然后由内核来识别，并根据它展开出内核中的platform_device、i2c_client、spi_device等设备，这些设备用到的内存、IRQ等资源也会被传递给内核。


### DTS、DTB和DTC的关系

.dts是设备树源码文件，DTC负责将.dts编译成.dtb。而.dtb就是负责传递给内核的二进制文件。

make dtbs会编译所有的dts文件，如果要编译指定的dtb，请使用make board_name.dtb。

在arch/arm/boot/dts/Makefile中，如果选中了某种SOC，则该SOC下的所有相关dtb文件都将被编译出来。

如果我们使用了I.MX6ULL新做了一个板子，只需要新建一个对应的.dts文件，并且将对应的.dtb文件添加到dtb-${CONFIG_SOC_IMX6ULL}下。

### DTS基本语法

1. "/"是root节点，一个.dts文件中只有一个root节点。多个文件中的"/"会被合并成一个根节点。
2. 子节点的描述信息用"{}"包含。
3. 设备节点的名称格式：label: node-name@unit-address。可以使用&label来访问这个节点。

在根节点下有两个特殊的节点：aliases和chosen

- aliases：用于给节点定义别名
- chosen：虚拟节点，可以在chosen中设置bootargs，用于给内核传递参数

节点由一堆属性组成，节点是具体的设备，但是不同的设备有不同的属性，不过有一些是标准属性。

1.compatible属性

compatible属性的值是一个字符串列表，用于将设备和对应的驱动绑定起来。字符串列表用于表示设备所要使用的驱动程序。compatible属性的格式如下：

```
"manufacturer, model"
```

manufacturer表示厂商，model表示对应驱动的名字。而根节点的compatible表示硬件设备名，SOC名。Linux内核会通过根节点的compatible属性查看是否支持此设备，如果支持设备就会启动内核。

compatible也可以有多个属性值，按照优先级查找。

2.model属性

model属性的值也是一个字符串，用来描述开发板的名字或者设备模块信息。

3.status属性

status属性的值与设备状态有关。

| 值      | 描述 |
| :--- | :--- |
| "okay"      | 表示设备是可操作的       |
| "disabled"   | 表示设备当前不可操作，但是未来可以变为可操作的        |
| "fail"  | 表示当前设备不可操作，但是未来

4.reg属性

reg属性的值一般是(address, length)对。用于描述设备地址信息。

```
reg = <0x4000e000 0x400>  //起始地址+大小
```

5.#address-cells和#size-cells属性

用于描述子节点的地址信息。

```
#address-cells: 决定了子节点reg属性的地址信息所占用的字长
#size-cells：决定了子节点reg属性中长度信息所占的字长
```

6.ranges属性

ranges属性的值按照(child-bus-address, parent-bus-address, lenght)格式编写。ranges属性用来指定某个设备的地址范围或者IO范围，这是对设备进行寻址的重要信息。操作系统通过ranges属性获知哪些内存区域或者IO端口是被硬件设备所占用的。

- child-bus-address：子总线地址空间的物理地址，由父节点的#address-cells确定此物理地址占用的字长
- parent-bus-address：父总线地址空间的物理地址，同样由父节点的#address-cells确定此物理地址占用的字长
- length：子地址空间的长度，由父节点的#size-cells确定此地址长度占用的字长

如果ranges属性为空，则说明子地址空间和父地址空间相同，不需要进行转换。

**Arm体系此属性设置为空**。

7.intc属性
用于表示中断控制器的相关信息。

- interrupt-controller：
- interrupt-cells：
- interrupt-parent：
- interrupts：

根节点必须要有的属性有：

```
#address-cells:子节点reg属性中，用多少个u32整数来描述地址
#size-cells：子节点reg属性中，用多少个u32整数来描述大小
compatible：指定板子兼容的平台
model：板子名称
```

### 内核操作设备树

内核提供了一系列函数来操作设备树中的节点和属性信息，这些函数统一以**of**开头。

1.节点操作函数
内核使用device_node结构体来描述一个节点
```C
struct device_node{
    const char *name;
    const char *type;
    cont char *full_name;
    ...
    struct device_node *parent;
    struct device_node *child;
    struct device_node *sibling;
    struct kobject kobj;
    unsigned long _flags;
    void *data;
};
```

节点的属性信息里保存了驱动所需要的内容，内核中使用结构体property表示属性。
```C
struct property{
    char *name;
    int length;
    void *value;
    struct property *next;
    unsigned long _flags;
    unsigned int unique_id;
    struct bin_attribute attr;
};
```
同时内核也提供了提取属性值的Of函数。




驱动通过查找节点，然后对设备树进行操作




