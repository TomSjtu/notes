# 设备树

## 设备树的由来

起因是在<arch/arm/mach-xxx\>目录下充斥着大量重复的board specific的代码，每次Linux内核merge window期间，ARM的代码变化占整个arch目录的一半以上，导致内核十分的臃肿。经过社区讨论后决定：

1. ARM的核心代码仍然保存在<arch/arm\>目录下
2. ARM SOC周边外设模块的驱动代码保存在drivers目录下
3. ARM SOC board specific的代码被移除，由Device Tree机制来负责传递硬件资源信息。

本质上，Device Tree改变了原来用hardcode方式将硬件配置信息嵌入到内核代码的方法。对于嵌入式系统，在系统启动阶段，由bootloader通过bootm命令将设备树信息传递给内核，然后由内核来识别，并根据它展开出内核中的platform_device、i2c_client、spi_device等设备，这些设备用到的内存、IRQ等资源也会被传递给内核。

设备树文件.dts用来描述硬件信息。一个SOC可以制作很多开发板，将这些开发板的通用信息提取出来，变为一个单独的.dtsi文件。其他的.dts可以直接引用通用文件，就像C语言中的头文件。

一般情况下，.dtsi描述SOC信息，比如CPU架构，主频。.dts文件描述板级信息，比如开发板上有哪些IIC设备、SPI设备等。

![device tree](../../images/kernel/device_tree.png)

开发板的设备树文件一般位于/boot/dtb/[SOC_NAME]/[BOARD_NAME.dtb]。

如果要查看开发板中的设备树结构可以使用：

```SHELL
ls /sys/firmware/devicetree/base
```

或者

```SHELL
ls /proc/device-tree
```

## DTS、DTB和DTC的关系

dts是设备树源码文件，包含的头文件为dtsi。dts文件是人类可以看懂的，但是uboot和linux无法识别，因此需要dtc将dts文件编译成dtb文件。dtb文件是uboot和linux可以识别的二进制文件。在Linux源码目录下/scripts/dtc目录包含了dtc工具的源码。

dtc的使用方法是：

```SHELL
dtc -I dts -O dtb -o [output].dtb [input].dts
```

反过来可以生成dts文件：

```SHELL
dtc -I dtb -O dts -o [output].dts [input].dtb
```

make dtbs会编译所有的dts文件，如果要编译指定的dtb，请使用make board_name.dtb。

## DTS基本语法

设备树的语法非常简单，就是由一系列的node和property组成。这些node被组织成树状结构，除了root node，每个node都有一个parent node。每个node都包含一系列的property，用来描述该node的一些信息。node还可以嵌套child node。

```
label:node-name@unit-address{
    属性1 = ...
    属性2 = ...
    属性3 = ...
    子节点...
}
```

- label：用来指定一个唯一的标签，方便引用。
- node-name：用来指定节点的名称。
- unit-address：用来指定地址，和此节点的reg属性的开始地址必须一致。

> 注意：如果node中没有reg属性，则节点名字中不能有unit-address。unit-address的具体格式和设备挂在哪个bus相关。例如对于CPU，其unit-address就是从0开始编址。而具体的设备，例如以太网控制器，其unit-address就是寄存器地址。

属性值标识了设备的特性，它的值可以是以下几种：

1. 可能为空，也就是没有值的定义。
2. 可能是一个u32、u64的数值，也可以是数组。
3. 可能是一个字符串，或者是string list。

### 特殊节点

别名节点`aliases`：用来给device-node定义别名，因为每次写一长串路径比较麻烦。

`memory`节点：用来描述系统物理内存的layout。`device_type`属性定义了该node的设备类型，例如cpu、serial等。对于memory node，其`device_type`必须等于memory。`reg`属性定义了访问该device node的地址信息——起始地址和长度。

`chosen`节点：用来定义启动参数，其父节点必须是根节点。内核的一些启动参数可以通过`chosen`节点下的`bootargs`属性来设置，它可以被bootloader读取。

### 属性

节点由一堆属性组成，节点是具体的设备，但是不同的设备有不同的属性，不过有一些是标准属性。

1.`compatible`属性

`compatible`属性用来表示device和driver的适配：

```
compatible = "rockchip, rk3399";
```

表示厂商和SOC名。在驱动文件中有一个of匹配表，用来匹配设备节点和驱动节点。如果设备节点的compatible属性值和of匹配表中任何一个值相等，那么就表示该设备可以使用这个驱动。

```C
static const struct of_device_id rockchip_rk3399_match[] = {
    { .compatible = "rockchip,rk3399" },
};
```

`compatible`也可以有多个属性值，按照优先级的顺序进行匹配。

2.`model`属性

`model`属性用来表示设备的型号。

3.`status`属性

`status`属性的值与设备状态有关，通过设置`status`属性可以禁用或者启用设备。

4.`reg`属性

`reg`属性的值一般是以(address, length)对的形式出现。用于描述设备资源在其父总线定义的地址空间内的地址。

```
reg = <0x4000e000 0x400>  //起始地址+大小
```

5.`#address-cells`和`#size-cells`属性

如果一个`device node`的sub node有寻址需求（即需要定义`reg`属性），那么这两个属性就必须要定义，用于描述sub node的`reg`属性的信息。

```
#address-cells: 决定了子节点reg属性的地址信息所占用的字长
#size-cells：决定了子节点reg属性中长度信息所占的字长
```

6.`ranges`属性

`ranges`属性的值按照(child-bus-address, parent-bus-address, lenght)格式编写。`ranges`属性用来指定某个设备的地址范围或者IO范围，这是对设备进行寻址的重要信息。操作系统通过`ranges`属性获知哪些内存区域或者IO端口是被硬件设备所占用的。

- child-bus-address：子总线地址空间的物理地址，由父节点的#address-cells确定此物理地址占用的字长
- parent-bus-address：父总线地址空间的物理地址，同样由父节点的#address-cells确定此物理地址占用的字长
- length：子地址空间的长度，由父节点的#size-cells确定此地址长度占用的字长

如果`ranges`属性为空，则说明子地址空间和父地址空间相同，不需要进行转换。

**Arm体系此属性设置为空**。

7.`intc`属性

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

## 内核的of函数

内核提供了一系列函数来操作设备树中的节点和属性信息，这些函数统一以`of`开头，定义在<include/linux/of.h\>中。

内核使用`device_node`结构体来描述一个节点
```C
struct device_node {
	const char *name;
	phandle phandle;
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
	struct	kobject kobj;
#endif
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};
```
与查找节点相关的`of`函数有5个：

1.通过节点名字查找指定节点

```C
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
```

> from：开始查找的节点，NULL则表示从根节点开始查找

> name：要查找的节点名

> 返回值：找到的节点，NULL表示失败

2.通过`device_type`属性查找指定节点（X）

```C
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
```

> type：要查找的节点的device_type属性值

由于`device_type`已经被废弃，所以这个函数已经不用了。

3.通过`device_type`和`compatible`两个属性来查找指定节点

```C
struct device_node *of_find_compatible_node(struct device_node *from, const char *type, const char *compatible)
```

4.通过`of_device_id`匹配表来查找指定节点

```C
struct device_node *of_find_matching_node_and_match(struct device_node *from, const struct of_device_id *matches, const struct of_device_id **match)
```

5.通过路径来查找指定节点

```C
inline struct device_node *of_find_node_by_path(const char *path)
```

> path：带有全路径的节点名，可以使用节点的别名

推荐使用这个方法来查找节点。

节点的属性信息里保存了驱动所需要的内容，内核中使用结构体`property`表示属性。

```C
struct property{
    char *name;    //属性名
    int length;    //属性值的长度
    void *value;   //指向属性值的指针
    struct property *next;      //指向下一个属性的指针
    unsigned long _flags;       //表示属性的类别
    unsigned int unique_id;     //标识设备的唯一属性
    struct bin_attribute attr;  //表示属性的一些元数据
};
```

通过亲缘关系来查找节点：

```C
struct device_node *of_get_parent(const struct device_node *node)

struct device_node *of_get_next_child(const struct device_node *node, struct device_node *prev)

struct device_node *of_get_child_by_name(const struct device_node *node, const char *name)
```

同时内核也提供了提取属性值的`of`函数。

1.查找指定的属性

```C
property *of_find_property(const struct device_node *np, const char *name, int *lenp)
```

> np：设备节点

> name：属性名字

> lenp：属性值的字节数

2.获取属性中元素的数量

```C
int of_property_count_elems_of_size(const struct device_node *np, const char *propname, int elem_size)
```

3.从属性中获取指定标号的u32类型数据

```C
int of_property_read_u32_index(const struct device_node *np, const char *propname, u32 index, u32 *out_value)
```

> propname：要读取的属性名

> index：要读取的值标号

> out_value：读取到的值

4.读取u8、u16、u32、u64数组类型的数据

```C
int of_property_read_u8_array(const struct device_node *np, const char *propname, u8 *out_values, size_t sz)
```

将函数名中的u8替换成其他数据类型即可。

5.读取u8、u16、u32、u64类型属性值

```C
int of_property_read_u8(const struct device_node *np, const char *propname, u8 *out_value)
```

6.读取属性中字符串的值

```C
int of_property_read_string(struct device_node *np, const char *propname, const char **out_string)
```

这个函数使用比较繁琐，建议使用以下函数：

```C
int of_property_read_string_index(const struct device_node *np,const char *propname, int index,const char **out_string)
```

相比前面的函数增加了参数index，它用于指定读取属性值中第几个字符串，index从零开始计数。 第一个函数只能得到属性值所在地址，也就是第一个字符串的地址，其他字符串需要我们手动修改移动地址，非常麻烦，推荐使用第二个函数。

现在内核提供了内存映射相关的`of`函数，可以自动完成物理地址到虚拟地址的转换：

```C
void __iomem *of_iomap(struct device_node *np, int index)
```

```
gpio1 {
	gpio-ranges = <&pinctrl 1 0 4>;
};