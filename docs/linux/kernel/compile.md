# 编译内核

## 源码结构

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

Linux 内核可粗略分为五个子系统：

- 进程管理：负责进程的创建、调度、终止等；
- 内存管理：负责物理、虚拟内存的分配和回收
- 文件系统：负责管理文件系统，包括VFS、块设备、字符设备等；
- 网络系统：负责网络设备的驱动、协议栈、网络栈等；
- 进程间通信：负责进程间通信的管理、同步等。

## 编译内核

Linux 内核从源码到安装使用大致可以分为三个阶段：配置、编译、安装。配置的过程主要由 Kconfig 提供的图形界面完成，编译与安装主要由 Kbuild 系统，由`make`命令实现。

### Kconfig

使用`make menuconfig`命令可配置内核，其界面（ARM）如下图所示：

![内核菜单配置界面](../../images/kernel/menuconfig.png)

Kconfig 是内核的配置工具，它提供了一种基于菜单的配置方式，使得用户可以灵活地配置内核。Kconfig 配置工具首先查找<arch/[architecture]/Kconfig\>文件，该文件定义一系列 CONFIG 配置项，关键字`source`可以引用其他 Kconfig 文件。每个 Kconfig 文件分别描述了所属目录的内核配置菜单，最终形成一个分层的树形结构。

在内核中添加程序需要完成以下三个步骤：

1. 将编写的源代码复制到对应的目录中。
2. 在目录的 Kconfig 文件中添加对于新代码的编译配置选项。
3. 在目录的 Makefile 文件中添加对于新代码的编译条目。

在菜单界面的配置完成后，各种配置选项保存至主目录的 .config 文件。`make`命令会根据 .config 文件中的配置选项，编译出对应的可执行文件。

配置选项有三种：Y、N或M。分别对应编译、不编译、以模块形式编译。

假如 MMU 的源代码文件为：mmu.c，在该目录的 Kconfig 文件中有个条目 config MMU，则在 Makefile 文件中关于此目录的编译条目为：`obj-$(CONFIG_MMU) += mmu.o`。如果关于 MMU 的配置选项选择为"Y"或者"M"，则编译 mmu.o；如果选择为"N"，则不编译 mmu.o。

Kconfig 的主要关键字有：

| 关键字 | 描述 |
| :------ | :------ |
| config | Kconfig最基本单元，定义了配置项的详细信息 |
| menu/endmenu | 目录，可以把一部分相关的配置项包含在一个menu中，menu可以嵌套使用 |
| if/endif | 条件编译 |
| choice/endchoice | 一组选择项，只能是bool或者tristate |
| comment | 注释 |
| source | 引入其他Kconfig文件 |

### Makefile

Kbuild 对 Makefile 进行了功能上的扩充，使其在编译内核文件时更加高效、简洁。

内核 Makefile 一共包含五部分：

| 文件 | 描述 |
| :------ | :------ |
| 顶层Makefile | 定义了内核的编译规则，如编译器、编译选项等 |
| .config文件 | 定义了内核的配置选项 |
| arch/${ARCH}/Makefile | 定义了特定体系结构的编译规则 |
| scripts/Makefile | 包含了所有定义与规则 |
| 各目录中的Makefile | 定义了当前目录的编译规则 |

### 内核镜像

编译后的内核镜像有以下几种：

1. vmlinux: 原始的未经任何处理的 Linux ELF 镜像，无法烧写到开发板，需要通过`objcopy`命令生成二进制文件。
2. Image: 对 vmlinux 使用`objcopy`处理的可烧写到开发板的镜像，但是比较大。
3. zImage: 压缩 Image 后的镜像。
4. uImage: 由 uboot 的`mkimage`工具加工 zImage 得到 64 字节头的 Image，可供 uboot 启动。





