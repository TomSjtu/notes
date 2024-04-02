# 内核的编译与启动

## 源码结构

要获取内核源码，请使用`git`。

```
git clone git://git.ernel.org/pub/scm/linux/kernel/git/torvalds/[linux版本号].git
```

内核源码树由许多目录组成，一些比较重要的目录的描述如下：

| 目录名 | 描述 |
| --- | --- |
| arch | 体系结构相关的代码 |
| block | 块设备I/O层 |
| drivers | 设备驱动程序 |
| fs | 文件系统 |
| include | 内核头文件 |
| init | 初始化代码 |
| ipc | 进程间通信代码 |
| kernel | 内核核心代码 |
| lib | 内核库函数 |
| mm | 内存管理 |
| net | 网络系统 |

Linux内核可大致分为五个子系统——进程管理、内存管理、文件系统、网络系统和进程间通信。其中进程管理在系统中处于核心位置，内存管理主要用于控制进程安全的访问内存区域，虚拟文件系统则隐藏了各种文件系统的差异，为用户提供统一的接口；网络系统则提供了对各种网络设备、网络协议的支持；进程间通信协助进程之间的通信与同步。


## 编译内核

Linux内核从源码到安装使用大致可以分为三个阶段：配置、编译、安装。配置的过程主要由Kconfig提供的图形界面完成，编译与安装主要由Kbuild系统，由`make`命令实现。

### Kconfig

要配置内核，可以使用以下指令：

```C
make config(基于文本)
make menuconfig(基于文本菜单)
make xconfig(要求安装QT)
make gconfig(要求安装GTK+)
```

其中最推荐的是`make menuconfig`，它不依赖于QT或者GTK+，且非常直观。运行`make menuconfig`后的界面（ARM）如下图所示：

![内核菜单配置界面](../../images/kernel/menuconfig.png)

运行`make menuconfig`后，配置工具首先查找<arch/[architecture]/Kconfig\>文件，该文件除了定义一系列CONFIG配置项，还通过关键字`source`语句引入了一系列Kconfig文件。每个Kconfig文件分别描述了所属目录源文档相关的内核配置菜单，最终形成一个分层的树形结构。就是我们在执行`make menuconfig`后看到的目录菜单。

在内核中添加程序需要完成以下三个步骤：

1. 将编写的源代码复制到对应的目录中。
2. 在目录的Kconfig文件中添加对于新代码的编译配置选项。
3. 在目录的Makefile文件中添加对于新代码的编译条目。

内核的各种配置，以CONFIG_FEATURE的形式写入主目录的.config文件。再执行`make`命令时，就会根据.config文件中的配置选项，编译出对应的可执行文件。

配置选项有三种：yes、no或module。分别对应编译、不编译、以模块形式编译。

假如MMU的源代码文件为：mmu.c，在该目录的Kconfig文件中有个条目config MMU，则在Makefile文件中关于此目录的编译条目为：obj-$(CONFIG_MMU) += mmu.o。如果关于MMU的配置选项选择为"Y"或者"M"，则编译mmu.o；如果选择为"N"，则不编译mmu.o。

!!! note

    - obj-y：编译进内核
    - obj-m：编译成模块
    - obj-n：不编译进内核

自己编写时可以仿照内核的写法。

Kconfig的主要关键字有：

| 关键字 | 描述 |
| :------ | :------ |
| config | Kconfig最基本单元，定义了配置项的详细信息 |
| menu/endmenu | 目录，可以把一部分相关的配置项包含在一个menu中，menu可以嵌套使用 |
| if/endif | 条件编译 |
| choice/endchoice | 一组选择项，只能是bool或者tristate |
| comment | 注释 |
| source | 引入其他Kconfig文件 |

### Makefile

Kbuild对Makefile进行了功能上的扩充，使其在编译内核文件时更加高效、简洁。

内核Makefile一共包含五部分：

| 文件 | 描述 |
| :------ | :------ |
| 顶层Makefile | 定义了内核的编译规则，如编译器、编译选项等 |
| .config文件 | 定义了内核的配置选项 |
| arch/${ARCH}/Makefile | 定义了特定体系结构的编译规则 |
| scripts/Makefile | 包含了所有定义与规则 |
| 各目录中的Makefile | 定义了当前目录的编译规则 |

!!! info "内核镜像"

    vmlinux: 原始的未经任何处理的Linux ELF镜像，无法烧写到开发板，需要通过`objcopy`命令生成二进制文件

    Image: 对vmlinux使用`objcopy`处理的可烧写到开发板的镜像，但是比较大

    zImage: 压缩Image后的镜像

    uImage: 由uboot的`mkimage`工具加工zImage得到64字节头的Image，供uboot启动

## 启动内核(待完成)

引导Linux系统有多种方式，这里只以Uboot启动ARM Linux为例来讲解。

SoC一般都内嵌了由厂家编写的bootrom，作为上电后运行的第一段代码，可以从SD、eMMC、NAND、USB等介质启动。对于多核CPU，bootrom首先去引导CPU0，其他CPU则睡眠。


### uboot命令

uboot支持的命令很多，这里只提下常用的，其他的命令用`help`查看即可。

| 命令 | 描述 |
| ---- | ---- |
| help | 获取帮助 |
| printenv | 打印环境变量 |
| setenv | 设置环境变量 |
| saveenv | 保存环境变量 |
| bootm | 启动内核 |
| md | 查看内存地址 |
| mw | 修改内存地址 |


### 制作根文件系统

根文件启动需要以下几个目录：

- bin
- dev
- etc
- lib
- lib64
- sbin

etc/fstab：文件挂载路径
etc/inittab：指定开机启动后运行的程序

init.d/rcS：启动脚本
init.d/rcK：关机脚本




