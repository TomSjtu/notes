# 内核模块

模块（modules）是一种特殊的机制，可以不编译进内核而动态加载，使得内核的维护与开发变得更加灵活。

!!! exmaple "最简单的内核模块hello.c"

    ```C

    #include <linux/modul.h\>
    #include <linux/init.h\>

    static int __init hello_init(void) {
        printk(KERN_INFO "Hello world init\n");
        return 0;
    }

    static void __exit hello_exit(void) {
        printk(KERN_INFO "Hello world exit\n");
    }

    module_init(hello_init);
    module_exit(hello_exit);
    MODULE_LICENSE("GPL");
    MODULE_AUTHOR("Tom");
    MODULE_DESCRIPTION("A simple hello world module");
    ```

这个模块只包含了模块的加载和卸载函数，编译后产生hello.ko目标文件，通过`insmod`命令加载到内核，输出"Hello world init"；`rmmod`命令卸载，输出"Hello world exit"。

使用`modprobe`命令除了加载该模块，还会将依赖一并加载。

使用`modinfo`命令可以获得模块的信息。

## 内核模块程序结构

一个Linux内核模块主要由以下几个部分组成：

1. 模块加载函数
2. 模块卸载函数
3. 模块许可证声明
4. 模块参数
5. 模块导出符号
6. 模块作者信息
7. 模块描述信息

编写内核模块时，必须要包含的头文件为`linux/module.h`和`linux/init.h`。

### 模块加载函数

模块加载函数以`__init`宏作为前缀，带有此标识的函数如果在编译时进入内核，在链接时会放在.init.text段中。

在模块加载失败时，应返回正确的值，同时释放在加载的过程中申请的资源。

### 模块卸载函数

模块卸载函数以`__exit`宏作为前缀，主要用于资源的释放。

### 模块参数

内核模块可以在加载、系统启动时或者系统运行时动态设置其参数。

模块参数以`module_param(参数名，参数类型，读写权限)`作为前缀，参数类型可以是byte、short、ushort、int、uint、long、ulong、charp、bool、invbool。参数数组形式为`module_param_array(参数名，参数类型，数组长度，读写权限)`。

!!! example "动态修改模块参数"

    ```C

    static int num = 5;
    module_param(num, int, S_IRUGO);
    ...
    ```

    在加载模块时，可以通过`insmod hello.ko num=10`来修改num的值。

### 导出符号

模块导出符号以`EXPORT_SYMBOL(符号名)`作为前缀，导出的符号可以在其他模块中被访问。`EXPORT_SYMBOL_GPL(符号名)`导出的符号在模块加载时会受到GPL协议的约束。

### 模块的声明与描述

我们可以用`MODULE_AUTHOR`、`MODULE_DESCRIPTION`、`MODULE_LICENSE`、`MODULE_ALIAS`、`MODULE_DEVICE_TABLE`、`MODULE_VERSION`分别声明模块的作者、描述、许可证、别名、设备表、版本号。

### 模块的编译

一个典型的Makefile文件示例如下：

```Makefile
ifneq ($(KERNELRELEASE),)
obj-m := hello.o
else
PWD := $(shell pwd)
KDIR :=/lib/modules/$(shell uname -r)/build
all:
	$(MAKE) -C $(KDIR) M=$(PWD)
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

endif
```



