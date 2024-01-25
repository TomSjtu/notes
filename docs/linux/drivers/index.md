# 设备模型

由于Linux支持世界上几乎所有的、不同功能的硬件设备，导致Linux内核中有一半的代码都是设备驱动。随着硬件的快速迭代，设备驱动的代码也在快速增长。

为了降低设备的多样性带来的驱动开发的复杂度，Linux提出了<font color="green">设备模型</font>（device model）的概念，该模型将设备和驱动分层，把我们编写的驱动代码分成了两块：设备与驱动。设备负责提供硬件资源而驱动负责去使用设备提供的硬件资源。二者由总线关联起来。

设备模型通过几个数据结构来反映当前系统中总线、设备以及驱动工作的情况：

- 总线(bus)：总线是CPU和设备之间信息交互的通道，所有的设备都应连接到总线上，无论是CPU内部总线还是虚拟总线。

- 类（class）：面向对象思想，将相同功能的设备，归结到一种class统一管理。

- 设备（device）：挂载在总线的物理设备。

- 驱动（driver）：硬件设备的驱动程序，负责初始化设备以及实现该设备的一些接口函数。

> platform bus是内核中的一种虚拟总线类型，它不是物理上存在的总线，而是一种抽象的总线。它允许开发者以一种标准的方式来描述和管理那些不通过传统物理总线连接的设备。

Linux内核使用<font color="green">sysfs文件系统</font>将内核的设备驱动导出到用户空间，用户可以通过访问/sys目录下的文件，来查看甚至控制内核的一些驱动设备。

/sys文件目录记录了各个设备之间的关系。其中，/sys/bus目录下的每个子目录都是已经注册的总线类型。每个总线类型下还有两个文件夹——devices和drivers；devices是该总线类型下的所有设备，以符号链接的形式指向真正的设备（/sys/devices/）。而drivers是所有注册在这个总线类型上的驱动。

![Linux设备模型](../../images/kernel/linux_device_model01.png)

/sys/devices目录下是全局的设备结构体系，包含了所有注册在各类总线上的物理设备。所有的物理设备以总线拓扑的结构来显示。

/sys/class目录下是包含所有注册在内核中的设备类型，按照设备的功能进行分类。比如鼠标的功能是作为人机交互的输入，于是被归类到/sys/class/input目录下。

那么“总线-设备-驱动”是如何配合工作的呢？

![总线-设备-驱动](../../images/kernel/linux_device_model02.png)

在总线上挂载了两个链表，分别管理设备模型和驱动模型。当我们向系统注册一个设备时，便会在设备的链表中插入新的设备。在插入的同时总线会执行`match()`方法对新插入的设备/驱动进行配对。若配对成功则调用`probe()`方法获取设备资源，在移除设备/驱动时，调用`remove()`方法。

## kobject、ktype和kset

kobject是Linux设备模型的基础，是一种抽象的、统一的对大量硬件设备的描述。它主要提供以下功能：

1. 通过parent指针，将所有kobject以树状结构的形式组合起来。
2. 使用引用计数kref，来记录kobject被引用的次数，在计数为0时释放它。
3. 和sysfs虚拟文件系统配合，将每一个kobject的特性，以文件的形式开放给用户空间查询。

设备驱动模型的基本元素有三个：

- kobject：sysfs中的一个目录，表示基本驱动对象。
- kset：一个特殊的kobject，用来管理类似的kobject。
- ktype：目录下kobject属性文件操作的接口。

kobject的数据结构如下：

```C
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```
> name：kobject的名称，同时也是sysfs中的目录名称。当kobject添加到内核时，需要根据name注册到sysfs中。

> entry：用于将kobject加入到链表中。

> parent：指向父kobject的指针，在sysfs中根据层次结构显示为目录结构。

> kset：该kobject属于的kset。若该kobject未指定parent，则会把kset作为parent。

> ktype：该kobject属于的kobj_type，每个kobject必须有一个ktype。

> sd：该kobject在sysfs中的对应目录项。

> kref：原子引用计数。

> state_initialized：指示该kobject是否已经初始化。

> state_in_sysfs：指示该kobject是否已在sysfs中建立目录。

> state_add_uevent_sent/state_remove_uevent_sent：记录是否已向用户空间发送add uevent。

> uevent_suppress：如果该字段为1，则表示忽略所有上报的uevent事件。

kobject核心机制：

内嵌在别的数据结构（比如device_driver）中，当kobject中的引用计数归零时，释放kobject所占用的内存空间。同时通过ktype中的`release()`回调函数，释放内嵌数据结构的内存空间。每一个内嵌kobject的数据结构都需要自己实现ktype中的回调函数。

ktype的数据结构如下：

```C
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	const struct attribute_group **default_groups;
	/*省略其他成员*/	
};
```

> release：当kobject引用计数归零时调用该析构函数，负责释放kobject的内存。

> sysfs_ops：sysfs文件系统读写时的特性。

> default_groups：定义了kobject的属性，由struct attritube和struct bin_attribute构成。

ktype的存在是为了描述一族kobject所具有的普遍特性。这样就不需要每个kobject定义自己的特性，而是在ktype中统一定义。

kset的数据结构如下：

```C
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
};
```

> list/list_lock：用于保存该kset下所有kobject对象。

> kobj：该kset自己的kobject。

> uevent_ops：uevent是用户空间的缩写，提供了与用户空间热插拔进行通信的机制。当任何kobject需要上报uevent时，都要调用所属的kset中uevent_ops中的函数。uevent的概念稍后说明。

kset是kobject对象的集合体。它与ktype的区别在于：具有相同ktype的kobject可以被分组到不同的kset。

## sysfs

sysfs文件系统是一个处于内存中的虚拟文件系统，它提供了kobject对象的层次结构视图。用户查询系统中各种设备的拓扑结构，就像查询文件目录一样简单。还可以通过导出文件的方式，将内核变量提供给用户读取或者写入。

sysfs的核心是把kobject对象与目录项（directory entries）关联起来，这点是通过kobject结构体中的dentry字段实现的。这种关联将kobject对象映射到该目录上。通过这种方式，我们可以轻松地查看挂载于/sys目录下的整个文件系统视图。向sysfs中添加kobject，可以使用`kobject_add()`和`kobject_create_and_add()`函数。

在/sys目录下挂载了至少11个目录：block, bus, class, dev, devices, firmware, fs, hypervisor, kernel, module, power。其中最重要的目录是devices，该目录将设备模型导出到用户空间。其目录机构就是系统中实际的设备拓扑结构。其他目录中的许多数据都是将devices目录下的数据加以转换加工所得。

我们已经知道kobject可以被映射为某个文件目录，仅有如此还不够。因为这样的sysfs仅仅只是一颗树，但没有提供实际数据的文件。为了能够读写这颗树，内核提供了`attribute`和`bin_attribute`两种属性。

在ktype中用`struct attribute_group`描述这两个属性：

```C
struct attribute_group {
	const char *name;
	struct attribute **attrs;
	struct bin_attribute **bin_attrs;
};

/*attribute和bin_attribute的定义如下*/

struct attribute {
	const char *name;
	umode_t mode;
};

struct bin_attribute {
	struct attribute attr;
	size_t size;
	void *private;
	struct address_space *(*f_mapping)(void);
	ssize_t (*read)(struct file *, struct kobject *, struct bin_attribute *,
			char *, loff_t, size_t);
	ssize_t (*write)(struct file *, struct kobject *, struct bin_attribute *,
			 char *, loff_t, size_t);
	int (*mmap)(struct file *, struct kobject *, struct bin_attribute *attr,
		    struct vm_area_struct *vma);
};
```

`struct attribute`为普通的attribute，使用该attribute生成的sysfs文件，只能用字符串的形式读写。而`struct bin_attribute`在`struct attribute`的基础上，增加了`read()`、`write()`等函数，因此它所生成的sysfs文件可以用任何方式读写。 

## uevent

uevent是kobject功能的一部分，用于在kobject状态发生改变时，比如添加、移除，通知用户空间。用户空间收到讯息后，做出相应的处理。

该机制通常是用来支持热插拔（hotplug）设备的，例如当U盘插入后，USB相关的驱动会动态创建用于表示该U盘的device结构，并告知用户空间为该U盘动态创建/dev/目录下的设备节点。

uevent机制比较简单，当设备模型中任何设备有事件需要上报时，都会触发uevent提供的接口。uevent模块可以通过两个途径把事件上报到用户空间：一种是通过kmod模块，另一种是通过netlink通信机制。

uevent的代码主要位于kobject.h和kobject_uevent.c两个文件。

`kobject_action`定义了uevent的类型：

```C
enum kobject_action {
	KOBJ_ADD,
	KOBJ_REMOVE,
	KOBJ_CHANGE,
	KOBJ_MOVE,
	KOBJ_ONLINE,
	KOBJ_OFFLINE,
	KOBJ_BIND,
	KOBJ_UNBIND,
};
```

> ADD/REMOVE：kobject的添加/移除事件。

> CHANGE：kobject状态或者内容发生改变。

> MOVE：kobject更改名称或者更改了parent。

> ONLINE/OFFLINE：kobject的上线/下线事件

> BIND/UNBIND：kobject的绑定/解绑事件

`kobj_uevent_env`定义了事件上报时的环境变量：

```C
struct kobj_uevent_env {
	char *argv[3];
	char *envp[UEVENT_NUM_ENVP];
	int envp_idx;
	char buf[UEVENT_BUFFER_SIZE];
	int buflen;
};
```

> argv：指针数组，可以保存命令行参数，最大为3个。

> envp：指针数组，用于保存每个环境变量的地址。

> envp_idx：访问envp数组的索引。

> buf：保存uevent消息的缓冲区

> buflen：存储缓冲区的大小

`kset_uevent_ops`定义了kset的uevent接口操作：

```C
struct kset_uevent_ops {
	int (* const filter)(struct kset *kset, struct kobject *kobj);
	const char *(* const name)(struct kset *kset, struct kobject *kobj);
	int (* const uevent)(struct kset *kset, struct kobject *kobj, struct kobj_uevent_env *env);
};
```

> filter：当kobject需要上报uevent时，它所属的kset可以通过此接口过滤掉不希望上报的uevent。

> name：用于获取kset中kobject的uevent名称，这个名称通常与uevent中的ACTION字段相对应。

> uevent：当一个kobject需要上报uevent时，uevent函数会被调用，它可以为uevent添加环境变量。

uevent的一些操作API：

```C
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp[]);
int kobject_synth_uevent(struct kobject *kobj, const char *buf, size_t count);
int add_uevent_var(struct kobj_uevent_env *env, const char *format, ...);
```

> kobject_uevent_env：以envp为环境变量，上报一个指定action的uevent。

> kobject_synth_uevent：允许发送一个合成的uevent。 

> add_uevent_var：以格式化字符串的形式，向uevent添加新的环境变量。

## device和device_driver

`device`和`device_driver`是Linux驱动开发的基本概念。驱动开发，其实就是开发指定的软件（driver）以及驱动指定的设备（device）。内核为此定义了两种数据结构，分别是`struct device`和`struct device_driver`。在<include/linux/device.h\>中可以找到这两个结构体的定义，由于比较复杂，就不在这里列举了。

Linux设备模型框架体系下开发，主要包括两个步骤：

1. 分配一个`struct device`类型的变量，填充信息，然后将其注册到内核。

2. 分配一个`struct device_driver`类型的变量，填充信息，然后将其注册到内核。

内核会在合适的时机，调用`struct device_driver`中的各类回调函数，从而触发后者终结设备驱动的执行。而所有的驱动程序逻辑，其实都是由这些回调函数来实现的。

一般情况下，Linux驱动开发很少直接操作上面两个结构体，因为内核又封装了一层，比如`platform_device`，封装后的接口更为简单易用。`device`和`device_driver`必须挂在在同一个bus之下，名称也必须一样，内核才能完成匹配操作。

如果存在相同名称的`device`和`device_driver`，内核就会执行`device_driver`中的`probe()`回调函数，该函数是所有`driver`的入口函数，用来执行诸如硬件设备初始化、字符设备注册、文件操作ops注册等动作（对应`remove()`函数）。

## bus

总线是连接处理器和设备之间的通道。为了方便设备模型的实现，系统中的每个设备都需要连接在一个总线上，这个总线可以是内部总线、虚拟总线或者是平台总线。

总线驱动则负责实现总线的各种行为，其管理着两个链表，分别是添加到该总线的设备链表以及注册到该总线的驱动链表。当你向总线添加（移除）一个设备（驱动）时，便会在对应的列表上添加新的节点， 同时对挂载在该总线的驱动以及设备进行匹配，在匹配过程中会忽略掉那些已经有驱动匹配的设备。

![总线模型](../../images/kernel/bus_model.jpg)

内核用`struct bus_type`结构体抽象出总线：

```C
struct bus_type {
	const char *name;
	const char *dev_name;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	int (*remove)(struct device *dev);

	struct subsys_private *p;
};
```

我们发现`bus_type`结构体中的大部分成员都与`device`有关，说明它主要负责设备的注册和注销等操作。

> name：该bus的名称，在sysfs中以目录形式存在，比如platform bus表现为/sys/bus/platform。

> dev_name：注册到bus的设备名称。

> match：当属于该bus的device或者device_driver添加到内核时，调用该函数。

> uevent：当属于该bus的device，发生添加、移除或者其他动作时，调用该函数。

> probe、remove：当属于该bus的device，发生初始化和移除时，调用该函数。

> p：保存了bus模块的一些私有数据。


bus模块的主要功能是：

- bus的注册和注销
- 本bus下有device或者device_driver注册到内核时的处理
- 本bus下有device或者device_driver从内核注销时的处理
- device_driver的probe
- 管理bus下所有的device和device_driver

内核提供了`bus_register()`函数来注册总线，`bus_unregister()`函数来注销总线。

```C
int bus_register(struct bus_type *bus);

void bus_unregister(struct bus_type *bus);
```

当我们成功注册总线时，会在/sys/bus/目录下创建一个新目录，目录名为我们新注册的总线名。bus目录中包含了当前系统中已经注册了的所有总线，例如i2c，spi，platform等。

## class

在本文的开头我们提到，class是用来统一管理相同功能的设备，这样可以避免每个设备驱动实现重复的功能。

```C
struct class {
	const char		*name;
	struct module		*owner;

	const struct attribute_group	**class_groups;
	const struct attribute_group	**dev_groups;
	struct kobject			*dev_kobj;

	int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
	char *(*devnode)(struct device *dev, umode_t *mode);

	void (*class_release)(struct class *class);
	void (*dev_release)(struct device *dev);

	int (*shutdown_pre)(struct device *dev);

	const struct kobj_ns_type_operations *ns_type;
	const void *(*namespace)(struct device *dev);

	void (*get_ownership)(struct device *dev, kuid_t *uid, kgid_t *gid);

	const struct dev_pm_ops *pm;

	struct subsys_private *p;
};
```

> name：class的名称，体现在/sys/class目录下。

> class_groups：class属性。

> dev_groups：dev属性。

> dev_uevent：设备的uevent回调函数。

> class_release/dev_release：release回调函数。

> pm：电源管理的回调函数。

> p：私有数据。

`struct class_interface`定义了当前class下有设备添加或者移除时，可以调用的回调函数：

```C
struct class_interface {
	struct list_head	node;
	struct class		*class;

	int (*add_dev)		(struct device *, struct class_interface *);
	void (*remove_dev)	(struct device *, struct class_interface *);
};
```

class的注册/注销函数如下：

```C
int __must_check __class_register(struct class *class, struct lock_class_key *key);
void class_unregister(struct class *class);
```

class的概念比较抽象，后续，在各类子系统中，我们能看到许多class的用例。

## 平台设备驱动

对于I2C、SPI、USB这些常见的设备来说，Linux内核都会创建与之相对应的驱动总线。但是有些结构简单的设备，比如led、rtc时钟、蜂鸣器等，内核就不会自己创建驱动总线。为了使这部分设备的驱动开发也能遵循设备驱动模型，Linux内核引入了虚拟的总线——平台总线（platform bus）。平台总线用于管理和挂载那些没有相应物理总线的设备，这些设备被称为平台设备，对应的设备驱动被称为平台驱动。平台设备对于Linux驱动工程师是非常重要的，因为大多数的驱动代码，实际就是为了驱动平台设备。

Platform架构图如下所示：

![platform架构](../../images/kernel/platform.gif)

从图片中我们可以看到：

- platform bus：继承自bus模块，用于挂载platform设备。
- platform device：继承自device模块，用于描述platform设备。
- platform drvier：继承自device_driver模块，用于驱动platform设备。

### 平台总线

在Linux平台设备驱动模型中，总线是最重要的一环，负责匹配设备和驱动。它维护着两个链表，里面记录着各个已经注册的平台设备和平台驱动。每当有新的设备或者是驱动加入到总线时，便会调用`platform_match()`函数对新增的设备或驱动进行配对。内核使用`struct bus_type platform_bus_type`来描述平台总线，该总线在内核初始化的时候注册：

```C
sturct bus_type platform_bus_type{
	.name = "platform",
	.dev_groups = platform_dev_groups,
	.match = platform_match,
	.uevent = platform_uevent,
	.pm = &platform_dev_pm_ops,
};
```

对`platform_bus_typ`e的初始化来说，`match`函数指针最为重要，它指向的函数负责实现平台总线和平台设备的匹配过程。对于每个驱动总线，都必须实例化该函数指针。

### 平台设备

内核使用`platform_device`结构体来描述平台设备：

```C
 struct platform_device {
     const char *name;    //设备名称，匹配时会比较驱动的名字
     int id;              //内核允许存在多个
     struct device dev;   //继承的device结构体
     u32 num_resources;   //记录资源的数目
     struct resource *resource;    //平台设备提供给驱动的资源
     const struct platform_device_id *id_entry;    
 };
```

平台设备的工作是为驱动程序提供设备信息,设备信息包括硬件信息和软件信息两部分。

1. 硬件信息：驱动程序需要使用到什么寄存器，占用哪些中断号、内存资源、IO口等等

2. 软件信息：以太网卡设备中的MAC地址、I2C设备中的设备地址、SPI设备的片选信号线等等

对于硬件信息，使用结构体`struct resource`来保存设备所提供的资源，比如设备使用的中断编号，寄存器物理地址等，结构体原型如下：

```C
struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
};
```

> name： 指定资源的名字，可以设置为NULL

> start、end： 指定资源的起始地址以及结束地址

> flags： 用于指定该资源的类型，在Linux中，资源包括I/O、Memory、Register、IRQ、DMA、Bus等多种类型，最常见的有以下几种：

| 资源宏定义 | 描述 |
| ---  | --- |
| IORESOURCE_IO | 用于IO地址空间，对应于IO端口映射方式 |
| IORESOURCE_MEM | 用于外设的可直接寻址的地址空间 |
| IORESOURCE_IRQ | 用于指定该设备使用某个中断 |
| IORESOURCE_DMA | 用于指定使用的DMA通道 |

设备驱动程序的主要目的是操作设备的寄存器。不同架构的计算机提供不同的操作接口，主要有IO端口映射和IO內存映射两种方式。对应于IO端口映射方式，只能通过专门的接口函数（如inb、outb）才能访问；采用IO内存映射的方式，可以像访问内存一样，去读写寄存器。在嵌入式中，基本上没有IO地址空间，所以通常使用`IORESOURCE_MEM`。

在资源的起始地址和结束地址中，对于IORESOURCE_IO或者是IORESOURCE_MEM，他们表示要使用的内存的起始位置以及结束位置；若是只用一个中断引脚或者是一个通道，则它们的start和end成员值必须是相等的。

注册/注销平台设备用到的函数如下：

```C
int platform_device_register(struct platform_device *pdev);
void platform_device_unregister(struct platform_device *pdev);
```

这两个函数应该在模块的进入与退出函数中被调用。

### 平台驱动

内核使用`platform_driver`结构体来描述平台驱动：

```C
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(structg platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
};
```

除了提供一些回调函数之外，还有一个`id_table`的指针。这个指针用来表示该驱动能够兼容的`platform_device`。

我们来看一下`struct platform_device_id`结构体的定义：

```C
struct platform_device_id {
	char name[PLATFORM_NAME_SIZE];
	kernel_ulong_t driver_data;
};
```

name用于指定驱动的名称，总线进行匹配时，会根据该name成员与`platform_device`中的name成员进行匹配，一旦匹配就会调用`probe()`函数。driver_data用来保存设备的配置。为了减少代码的冗余，一个驱动可以匹配多个设备。

注册/注销平台驱动的函数如下：

```C
int platform_driver_register(struct platform_device *drv);
void platform_driver_unregister(struct platform_device *drv);
```

在平台设备中，`struct resource`结构体用来表示硬件信息，而软件信息则可以用设备结构体`device`中的成员platform_data来保存。

`platform_get_resource()`函数通常会在驱动的`probe()`函数中执行，用于获取`platform_device`中的`resource`结构体：

```C
struct resource *platform_get_resource(struct platform_device *dev, unsigned int type, unsigned int num);
```

> dev：指定要获取的平台设备。

> type：指定获取资源的类型，比如IORESOURCE_IO。

> num：指定要获取的资源编号。

如果资源类型是IORESOURCE_IRQ，可以使用以下接口还获取中断引脚：

```C
int platform_get_irq(struct platform_device *pdev, unsigned int num);
```

对于存放在`device`结构体中platform_data的软件信息，可以使用`dev_get_platdata()`函数来获取：

```C
static inline void *dev_get_platdata(const struct device *dev)
{
    return dev->platform_data;
}
```

总结一下平台驱动：需要手动实现`probe()`函数，当平台总线成功匹配`platform_device`和`platform_driver`时，则会调用驱动的`probe()`函数，在该函数中使用上述的函数接口来获取资源，以初始化设备，最后填充结构体`platform_driver`，调用`platform_driver_register()`进行注册。


