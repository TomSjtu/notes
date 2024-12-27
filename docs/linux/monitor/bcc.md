# BCC

BCC 的代码仓库见：https://github.com/iovisor/bcc。Ubuntu 可以用`sudo apt-get install bpfcc-tools linux-headers-$(uname -r)`安装。

BCC 工具的用户态组件用 Python 编写，内核态 BPF 程序用 C 语言编写。BCC 包含了许多单一功能的工具，即每个工具仅完成一个特定的任务。除了单一功能的工具之外，BCC 还提供了一些多用途工具：

| 工具 | 描述 |
| --- | --- |
| funccount | 统计指定函数的调用次数 |
| stackcount | 统计引发某事件的函数调用栈 |
| trace | 定制化打印每个事件的细节信息 |
| argdist | 统计事件的参数分布 |

## funccount

funccount 统计函数调用的次数，可以用它来回答以下问题：

- 某个内核态或用户态函数是否被调用过？
- 该函数每秒被调用了多少次？

它的语法如下：

```SHELL
funccount [options] eventname
```

eventname 可以是：

- name 或者 p:name：对内核函数进行插桩
- lib:name 或者 p:lib:name：对用户态 lib 库中的函数进行插桩
- path:name：对位于 path 路径下的用户态函数进行插桩
- t:system:name：对名为 system:name 的内核跟踪点进行插桩
- u:lib:name：对 lib 库中名为 name 的 USDT 探针进行插桩

## stackcount

stackcount 对导致某事件发生的函数调用栈进行计数，事件源可以是内核态或用户态函数、内核跟踪点或 USDT 探针。它可以用来回答以下问题：

- 某个事件为什么被调用？调用的代码路径是什么？
- 有哪些不同的代码路径会调用该事件？它们的调用频次如何？

stackcount 可以使用 -f 选项生成火焰图格式：

```SHELL
$ stackcount -f -P -D 10 ktime_get > out.stackcount.txt
$ ./flamegraph.pl --hash --bgcolors=grey < out.stackcount.txt > out.stackcount.svg
```

## trace

trace 可以针对多个数据源进行每个事件的跟踪，它可以用来回答以下问题：

- 当某个内核态/用户态函数被调用时，调用的参数是什么？
- 这个函数的返回值是什么？调用失败了吗？
- 这个函数是如何被调用的？相应的调用栈是什么？

trace 会对每个事件产生一行输出，比较适合用于低频事件，高频视频会产生太多输出，造成非常显著的额外开销。

## argdist

argdist 是一个针对函数调用参数分析的多用途工具，它的语法如下：

```SHELL
argdist {-C | -H } [options] probe
```

argdist 必须指定参数 -C 或者 -H：

- -C：频率统计
- -H：直方图统计

## 调试

BCC 工具的调试方法有以下几种：

- `bpf_trace_printk()`：在代码中打印信息，读取 /sys/kernel/tracing/trace_pipe 文件获取输出
- --ebpf 参数：打印该工具最终生成的 BPF 程序
- 调试标志位：在代码中声明
- bpflist：查看正在运行的 BPF 程序
- bpftool：查看 BPF 程序的状态