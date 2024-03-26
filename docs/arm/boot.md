# ARM Linux启动



## 从设备树获取信息


### 识别机器类型


内核通过设备树中的信息来识别机器类型。理想情况下，特定的平台不应该影响内核，因为所有的平台细节都会完美地被设备树以一致且可靠的方式描述。但是硬件并不完美，因此内核必须在启动早期识别出具体的机型，这样就有机会执行机型特有的硬件修复代码。大多数时候，机型并不重要，内核根据机型的CPU或者SoC来选择执行相应的启动代码。以ARM为例，<arch/arm/kernel/setup.c\>中的`setup_arch()`函数会调用位于<arch/arm/kernel/devtree.c\>中的`setup_machine_fdt()`函数。该函数会查找machine_desc表并选择与设备树最为匹配的machine_desc。通过比较设备树根节点的compatible属性和定义在<arch/arm/include/asm/mach/arch.h\>中的machine_desc数据结构的dt_compat列表来选择最佳匹配。

compatible属性包含了以确切机器名开始的一组有序字符串。以ARM为例，对于每一个machine_desc，内核会检查其dt_compat列表中的条目是否出现在compatible属性中。如果有的话，对应的machine_desc就会作为驱动机器的一个候选。

当machine_descs表被全部检索后，`setup_machine_fdt()`函数返回一个最佳匹配的machine_desc——依据machine_desc与compatible属性的哪一个条目相匹配来确定。如果没有匹配的machine_desc，那么该函数返回NULL。在选择了machine_desc之后，`setup_machine_fdt()`还负责早期的设备树扫描。


### 运行时配置

大多数情况下，设备树将作为u-boot与内核传递数据的唯一方法。所以内核参数、initrd镜像的位置等运行时配置项也可以通过设备树传递。这些数据一般存放在/chosen节点下。

bootargs属性包含了内核参数，initrd-开头的属性则定义了initrd数据块的起始地址和大小。在启动阶段的早期，分页机制还没有开启，`setup_machine_fdt()`借助不同的辅助函数调用`of_scan_flat_dt()`对设备树的数据进行扫描。`of_scan_flat_dt()`扫描整个设备树，利用传入的辅助函数来提取启动阶段早期所需要的信息。辅助函数`early_init_dt_scan_chosen()`主要用来解析包含内核启动参数的chosen节点，`early_init_dt_scan_root()`则负责初始化设备树的地址空间模型，`early_init_dt_scan_memory()`的调用决定了可用内存的位置和大小。


### 设备填充

在机型识别完成并且解析完早期的配置信息之后，内核的初始化就可以按常规方式继续了。在这个过程中的某个时间点，`unflatten_device_tree()`函数负责将设备树的数据转化为更为有效的运行时描述方式。在ARM平台上，这里也是机型特有的启用钩子（比如`init_early()`、`init_irq()`和`init_machine()`）被调用的地方。这些函数的目的可以从名字看出来。

在设备树上下文中最有趣的钩子函数当属`init_machine()`，主要负责根据平台相关的信息填充Linux设备模型。设备列表可以通过解析设备树获取，然后动态地为这些设备分配device数据结构。




