# 设备树

## 为什么需要设备树

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

.dts是设备树源码文件，DTC负责将.dts编译成.dtb。而.dtb就是负责传递给内核的二进制文件。

make dtbs会编译所有的dts文件，如果要编译指定的dtb，请使用make board_name.dtb。

在arch/arm/boot/dts/Makefile中，如果选中了某种SOC，则该SOC下的所有相关dtb文件都将被编译出来。

如果我们使用了I.MX6ULL新做了一个板子，只需要新建一个对应的.dts文件，并且将对应的.dtb文件添加到dtb-${CONFIG_SOC_IMX6ULL}下。

## DTS基本语法

设备树的每个节点按照以下规则命名：

```
label:node-name@unit-address{
    属性1 = ...
    属性2 = ...
    属性3 = ...
    子节点...
}
```

device tree的基本单元是node，这些node被组织成树状结构。除了root node，每个node都有一个parent node。一个device tree文件中只能有一个root node。每个node中都包含了property: value来描述该node的一些信息。

`label`用来指定标签名，方便引用。`node-name`用于指定节点的名称，`unit-address`用于指定地址，其值要与节点`reg`属性的第一个地址一致。

属性值标识了设备的特性，它的值可以是以下几种：

1. 可能为空，也就是没有值的定义。
2. 可能是一个u32、u64的数值，也可以是数组。
3. 可能是一个字符串，或者是string list。

在根节点下有两个特殊的节点：`aliases`和`chosen`

- aliases：用于给节点定义别名，方便对节点的引用。
- chosen：虚拟节点，可以在chosen中设置bootargs，由bootloader读取，传递给内核作为启动参数。

还有一个所有设备树文件必须要有的节点是`memory device node`，它定义了系统物理内存的layout。`device_type`属性定义了该node的设备类型，例如cpu、serial等。对于memory node，其`device_type`必须等于memory。`reg`属性定义了访问该device node的地址信息——起始地址和长度。

节点由一堆属性组成，节点是具体的设备，但是不同的设备有不同的属性，不过有一些是标准属性。

1.`compatible`属性

`compatible`属性的值是一个字符串列表，用于将设备和对应的驱动绑定起来。字符串列表用于表示设备所要使用的驱动程序。`compatible`属性是用来查找节点的方法之一。例如系统初始化platform总线上的设备时，根据设备节点`compatible`属性和驱动中`of_match_table`对应的值，匹配了就加载对应的驱动。`compatible`属性的格式如下：

```
"manufacturer, model"
```

manufacturer表示厂商，model表示对应驱动的名字。而根节点的`compatible`表示硬件设备名，SOC名。Linux内核会通过根节点的`compatible`属性查看是否支持此设备，如果支持设备就会启动内核。

`compatible`也可以有多个属性值，按照优先级查找。

2.`model`属性

`model`属性用来描述设备的生产厂商和型号。比如model="samsung, s3c24xx"——生产厂商是三星，SOC是s3c24xx。

3.`status`属性

`status`属性的值与设备状态有关，通过设置`status`属性可以禁用或者启用设备。

4.`reg`属性

`reg`属性的值一般是(address, length)对。用于描述设备资源在其父总线定义的地址空间内的地址。

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

## 内核操作设备树

内核提供了一系列函数来操作设备树中的节点和属性信息，这些函数统一以`of`开头。

节点操作函数

内核使用`device_node`结构体来描述一个节点
```C
struct device_node{
    const char *name;      //设备名称
    const char *type;      //设备类型
    cont char *full_name;  //设备的完整名称
    ...
    struct device_node *parent;  //父节点指针
    struct device_node *child;   //子节点指针
    struct device_node *sibling; //同级节点指针
    struct kobject kobj;         //kobject结构体是内核对象的一部分，用于跟踪此节点
    unsigned long _flags;        //表示节点的属性
    void *data;                  //指向任意数据的指针
};
```
与查找节点相关的`of`函数有5个：

1.通过节点名字查找指定节点

```C
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
```

- from：开始查找的节点，NULL则表示从根节点开始查找
- name：要查找的节点名
- 返回值：找到的节点，NULL表示失败

2.通过`device_type`属性查找指定节点（X）

```C
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
```

- type：要查找的节点的device_type属性值

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

- path：带有全路径的节点名，可以使用节点的别名

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

内核提供了查找父子节点的`of`函数。

```C
struct device_node *of_get_parent(const struct device_node *node)

struct device_node *of_get_next_child(const struct device_node *node, struct device_node *prev)
```

同时内核也提供了提取属性值的`of`函数。

1.查找指定的属性

```C
property *of_find_property(const struct device_node *np, const char *name, int *lenp)
```

- np：设备节点
- name：属性名字
- lenp：属性值的字节数

2.获取属性中元素的数量

```C
int of_property_count_elems_of_size(const struct device_node *np, const char *propname, int elem_size)
```

3.从属性中获取指定标号的u32类型数据

```C
int of_property_read_u32_index(const struct device_node *np, const char *propname, u32 index, u32 *out_value)
```

- propname：要读取的属性名
- index：要读取的值标号
- out_value：读取到的值

4.读取u8、u16、u32、u64类型的数组数据

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


