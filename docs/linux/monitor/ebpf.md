# eBPF入门

对于eBPF的介绍可以参考这篇文章：[eBPF简介](https://www.ebpf.top/post/head_first_bpf/)。

推荐Linux 4.9以上版本的内核来使用eBPF，部分Linux Event和BPF版本支持见下图：

![BPF版本支持](../../images/kernel/linux_kernel_event_bpf.png)

eBPF技术主要用于以下场景：

1. 追踪和性能分析(Tracing & Profiling)

2. 观测和监控(Obervability & Monitoring)

3. 网络(Network)

4. 安全(Security)

eBPF的整体架构如图所示：

![eBPF架构](../../images/kernel/linux_ebpf_internals.png)

eBPF分为用户空间和内核空间两部分：

- 用户空间的程序被编译成BPF字节码然后送至内核空间
- 内核空间的verifier检测字节码安全性，然后送至BPF虚拟机执行，将执行的结果回传给用户空间

其中，BPF相关的程序类型可以是kprobes/uprobes/tracepoint/perf_events中的一个或多个：

- kprobes：内核级别动态跟踪，可以跟踪内核中的函数入口或返回点，但是ABI接口不稳定
- uprobes：用户级别动态跟踪，只跟踪用户态函数
- tracepoints：内核静态跟踪
- perf_events：定时采样

内核中运行的BPF字节码可以使用两种方式将测量数据回传给用户空间：

- maps：内核中实现的统计摘要信息（比如测量延迟、堆栈信息）等。
- perf-event：内核采集的事件实时发送至用户空间，用户空间程序实时读取分析。

在Linux观测方面，eBPF总是会拿来与内核模块方式进行对比，eBPF在安全性、入门门槛上比内核模块都有优势，下面是一些对比项：

| 维度 | 内核模块 | eBPF |
|----|----|----|
| kprobes/tracepoints | 支持 | 支持 |
| 安全性 | 可能引入安全漏洞或导致内核Panic | 通过验证器进行检查，可以保障内核安全 |
| 内核函数 | 可以调用内核函数 | 只能通过BPF Helper函数调用 |
| 编译性 | 需要编译内核 | 不需要编译内核，只需要引入头文件 |
| 运行 | 必须是相同内核版本 | 基于稳定ABI的BPF程序可以编译一次，各处运行 |
| 与应用程序交互 | 打印日志或文件 | 通过perf_event或maps结构 |
| 数据结构丰富性 | 一般 | 丰富 |
| 入门门槛 | 高 | 低 |
| 升级 | 需要卸载和加载，可能导致处理流程中断 | 原子替换升级，不会造成处理流程中断 | 
| 内核内置 | 视情况而定 | 内核内置支持 |

要编写BPF程序可以有以下几种方式：

- BCC：即BPF Compiler Collection，可以让用户采用Python、C和Lua等高级语言快速开发BPF程序。
- bpftrace：采用类似于awk语言快速编写eBPF程序。
- libbpf库：需要使用LLVM clang编译成BPF字节码，门槛较高。

## BCC工具

![BCC Tools](../../images/kernel/bcc_tracing_tools.png)


BCC是一个开源的工具集，用于简化eBPF程序的编写与调试，它提供了一系列的工具和库，包括C库API和Python前端，适用于许多场景，比如性能分析和网络流量控制等。

安装BCC请参考官方文档：[Installing BCC](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

安装完之后运行[hello_world.py](https://github.com/iovisor/bcc/tree/master/examples/hello_world.py)，同时在另一个窗口运行一些命令(比如"ls")，你应该能看到窗口会打印"Hello, World!"。如果没有，那说明BCC安装失败。


## bpftrace

![bpftrace probe types](../../images/kernel/bpftrace_probes.png)