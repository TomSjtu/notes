# 性能观测技术概览

Linux 性能观测技术主要分为{==指标观测==}和{==跟踪观测==}。

指标观测是指从系统中获取一个数字化的值，用来展示当前系统的运行情况，比如 CPU 利用率、负载这些都属于指标观测。指标观测只能用于发现问题，但是无法精细地定位问题。跟踪观测是更高级的观测技术，可以收集系统相关活动的信息，从而定位出真正的性能问题。最著名的火焰图就是通过采样的方式，对用户或内核的调用栈进行分析，找出哪些函数消耗了更多的 CPU 周期。
类似的跟踪工具还有`ftrace`、`trace-cmd`，最近流行起来的`eBPF`，以及在其基础上封装的`BCC`、`bpftrace`等工具。

不管是何种工具，都依赖底层提供的事件源。事件源可以分为硬件事件、内核软件事件、内核跟踪和用户跟踪。其中，内核跟踪又分为静态跟踪(tracepoint)、动态跟踪(kprobe)，用户跟踪分为静态跟踪(USDT probe)、动态跟踪(uprobe)。

## 软件事件

`perf`工具可以用来查看当前系统支持的软件事件：

```SHELL
$ perf list sw
List of pre-defined software events(to be used in -e):
  alignment-faults
  context-switches OR cs
  cpu-migrations OR migrations
  emulation-faults
  major-faults OR major_faults
  major-faults
  minor-faults
  page-faults OR faults
  task-clock
```

- `alignment-faults`：发生内存对齐错误的次数。
- `context-switches`：发生上下文切换的次数。
- `cpu-migrations`：发生 CPU 迁移的次数。
- `emulation-faults`：发生模拟错误的次数。
- `page-faults`：发生缺页异常的次数。

## 内核跟踪

静态跟踪是内核在源码中提前插入了桩，当没有跟踪需求时，这些桩处于关闭状态。当某个跟踪点被打开，桩代码得以运行。桩是静态的，它的优势是比较稳定，但是内核不可能在所有函数中都插入桩。

动态跟踪是在内核运行时动态插入代码，内核在执行函数时，先执行插入的代码，然后返回函数的中断处继续执行剩余代码。