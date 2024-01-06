# 设备树

## 为什么需要设备树

具体可以参考Linus在2011年写的一个邮件：[this whole ARM thing is a f*cking pain in the ass](https://lkml.org/lkml/2011/3/17/492)。

简单来说就是，ARM社区充斥着大量垃圾的、重复的描述板级信息的代码（arch/arm/mach-xxx），导致Linux内核十分的臃肿。因为Linux只是一个操作系统内核，它并不关心你开发板上有什么设备。于是ARM社区引入了PowerPC等架构已经采用的设备树。将这些描述硬件信息的内容从Linux内核中剥离出来，用一个单独的文件进行描述，这就是设备树文件.dts的由来。一个SOC可以制作很多开发板，这些开发板必然有一些通用的信息。将这些通用的信息提取出来作为一个通用的文件，其他的.dts文件直接引用通用文件即可，类似于C语言中的头文件。这个通用文件就是.dtsi文件。

一般情况下，.dtsi描述SOC信息，比如CPU架构，主频。.dts文件描述板级信息，比如开发板上有哪些IIC设备、SPI设备等。

对于嵌入式系统，在系统启动阶段，由bootloader通过bootm命令将设备树信息传递给内核，然后由内核来识别，并根据它展开出内核中的platform_device、i2c_client、spi_device等设备，这些设备用到的内存、IRQ等资源也会被传递给内核。

![device tree](../../images/kernel/device_tree.png)

## DTS、DTB和DTC的关系

.dts是设备树源码文件，DTC负责将.dts编译成.dtb。而.dtb就是负责传递给内核的二进制文件。

make dtbs会编译所有的dts文件，如果要编译指定的dtb，请使用make board_name.dtb。

在arch/arm/boot/dts/Makefile中，如果选中了某种SOC，则该SOC下的所有相关dtb文件都将被编译出来。

如果我们使用了I.MX6ULL新做了一个板子，只需要新建一个对应的.dts文件，并且将对应的.dtb文件添加到dtb-${CONFIG_SOC_IMX6ULL}下。

## DTS基本语法

设备树的每个节点按照以下规则命名：

```
node-name@unit-address{
    属性1 = …
    属性2 = …
    属性3 = …
    子节点…
}
```

node-name用于指定节点的名称，unit-address用于指定单元地址，其值要与节点reg属性的第一个地址一致。

1. "/"是root节点，一个.dts文件中只有一个root节点。多个文件中的"/"会被合并成一个根节点。
2. 子节点的描述信息用"{}"包含。
3. 设备节点的名称格式：label: node-name@unit-address。label被称为标签，可以使用&label来访问这个节点。

在根节点下有两个特殊的节点：aliases和chosen

- aliases：用于给节点定义别名
- chosen：虚拟节点，可以在chosen中设置bootargs，用于给内核传递参数

节点由一堆属性组成，节点是具体的设备，但是不同的设备有不同的属性，不过有一些是标准属性。

1.compatible属性

compatible属性的值是一个字符串列表，用于将设备和对应的驱动绑定起来。字符串列表用于表示设备所要使用的驱动程序。compatible属性是用来查找节点的方法之一。例如系统初始化platform总线上的设备时，根据设备节点”compatible”属性和驱动中of_match_table对应的值，匹配了就加载对应的驱动。compatible属性的格式如下：

```
"manufacturer, model"
```

manufacturer表示厂商，model表示对应驱动的名字。而根节点的compatible表示硬件设备名，SOC名。Linux内核会通过根节点的compatible属性查看是否支持此设备，如果支持设备就会启动内核。

compatible也可以有多个属性值，按照优先级查找。

2.model属性

model属性的值也是一个字符串，用来描述设备的制造厂商和型号。

3.status属性

status属性的值与设备状态有关，通过设置status属性可以禁用或者启用设备。

4.reg属性

reg属性的值一般是(address, length)对。用于描述设备资源在其父总线定义的地址空间内的地址。

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

## 内核操作设备树

内核提供了一系列函数来操作设备树中的节点和属性信息，这些函数统一以**of**开头。

节点操作函数

内核使用device_node结构体来描述一个节点
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
与查找节点相关的**of**函数有5个：

1.通过节点名字查找指定节点

```C
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
```

- from：开始查找的节点，NULL则表示从根节点开始查找
- name：要查找的节点名
- 返回值：找到的节点，NULL表示失败

2.通过device_type属性查找指定节点（X）

```C
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
```

-type：要查找的节点的device_type属性值

由于device_type已经被废弃，所以这个函数已经不用了。

3.通过device_type和compatible两个属性来查找指定节点

```C
struct device_node *of_find_compatible_node(struct device_node *from, const char *type, const char *compatible)
```

4.通过of_device_id匹配表来查找指定节点

```C
struct device_node *of_find_matching_node_and_match(struct device_node *from, const struct of_device_id *matches, const struct of_device_id **match)
```

5.通过路径来查找指定节点

```C
inline struct device_node *of_find_node_by_path(const char *path)
```

- path：带有全路径的节点名，可以使用节点的别名

推荐使用这个方法来查找节点。

节点的属性信息里保存了驱动所需要的内容，内核中使用结构体property表示属性。

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

内核提供了查找父子节点的**of**函数。

```C
struct device_node *of_get_parent(const struct device_node *node)

struct device_node *of_get_next_child(const struct device_node *node, struct device_node *prev)
```

同时内核也提供了提取属性值的**of**函数。

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

现在内核提供了内存映射相关的of函数，可以自动完成物理地址到虚拟地址的转换：

```C
void __iomem *of_iomap(struct device_node *np, int index)
```


