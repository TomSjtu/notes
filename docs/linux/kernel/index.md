# 内核基础知识

## 内核源码

要获取内核源码，请使用git。

```
git clone git://git.ernel.org/pub/scm/linux/kernel/git/torvalds/[linux版本号].git
```

内核源码树由许多目录组成，各目录的描述如下：

| 目录名 | 描述 |
| --- | --- |
| arch | 体系结构相关的代码 |
| block | 块设备I/O层 |
| crypto | 加密API |
| Documentation | 内核文档 |
| drivers | 设备驱动程序 |
| firmware | 设备固件 |
| fs | 文件系统 |
| include | 内核头文件 |
| init | 初始化代码 |
| ipc | 进程间通信代码 |
| kernel | 内核核心代码 |
| lib | 内核库函数 |
| mm | 内存管理 |
| net | 网络系统 |
| samples | 示例代码 |
| scripts | 编译脚本 |
| security | 安全模块 |
| sound | 声音系统 |
| usr | 用户空间代码 |
| tools | 工具 |
| virt | 虚拟化系统 |

要编译内核，请使用make menuconfig。内核的各种配置，以CONFIG_FEATURE的形式写入.config文件。配置选项有三种：yes、no或module。分别对应编译、不编译、以模块形式编译。

内核开发的特点如下：

- 不能使用标准C库头文件，只能使用内核提供的头文件。与体系结构无关的头文件位于内核源码根目录下的include目录。与体系结构相关的头文件位于<arch/architecture/include/asm\>目录，内核代码以asm前缀的形式包含这些头文件。

- 必须使用GNU C。gcc编译器支持以asm()指令开头嵌入汇编代码。

- 没有内存保护机制。如果是内核访问了非法内存，后果不堪设想。

- 无法执行浮点运算。内核对于浮点运算的支持度不够。

- 必须考虑同步与并发。Linux内核是抢占式多任务处理系统，当前任务随时有可能被另一个任务抢占。

- 必须考虑可移植性。比如保持字节序、64 位对齐、不假定字长和页面长度等一系列准则。



