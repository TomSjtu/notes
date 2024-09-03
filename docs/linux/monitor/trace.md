# 静态跟踪

`ftrace`和`perf`工具都可以用来实现静态跟踪。

## ftrace

首先进入到 /sys/kernel/debug/tracing 目录，查看当前支持的跟踪器：

```SHELL
$ cat available_tracers 
timerlat osnoise hwlat blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function nop
```

其中常用的是`function`和`function_graph`，`function`只显示函数名，而`function_graph`还显示该函数的调用关系。

/events/ 目录列出了可以跟踪的各个模块，进入模块目录，enable 写入1表示打开静态跟踪，然后访问 /tracing/ 目录的`trace_pipe`就可以看到内核打印的日志了。

### 常用配置

| 名称 | 说明 | 
| ---- | ---- |
| available_tracers | 支持的跟踪器，current_tracer必须是这里面支持的跟踪器 |
| available_events | 支持的跟踪事件|
| available_filter_functions | 可跟踪的函数 |
| current_tracer | 当前使用的跟踪器，默认为nop |
| trace | 跟踪结果，用cat查看 |
| tracing_on | 开启或暂停 |
| max_graph_depth | 函数嵌套的最大深度 |
| set_ftrace_filter | 仅追踪特定的函数 |
| set_ftrace_notrace | 忽略特定的函数 |
| set_ftrace_pid | 追踪特定的PID |

### 使用方法

1. 关闭追踪

    ```SHELL
    $ echo 0 > /sys/kernel/debug/tracing/tracing_on
    ```

2. 设置追踪器

    ```SHELL
    $ echo function > /sys/kernel/debug/tracing/current_tracer
    ```

3. 开启追踪

    ```SHELL
    $ echo 1 > /sys/kernel/debug/tracing/tracing_on
    ```

4. 获取trace信息

    ```SHELL
    $ cat /sys/kernel/debug/tracing/trace
    ```
    
5. 关闭追踪

    ```SHELL
    $ echo 0 > /sys/kernel/debug/tracing/tracing_on
    ```
    
## trace-cmd

使用`ftrace`需要对文件进行频繁的写入和读出，操作起来比较麻烦。而`trace-cmd`是对`ftrace`的封装，使用起来更加方便。

```SHELL
trace-cmd version 2.9.1 (d8edc93bf4a92a4575eb3fb1108fef8030ede48b)

usage:
  trace-cmd [COMMAND] ...

  commands:
     record - record a trace into a trace.dat file
     set - set a ftrace configuration parameter
     start - start tracing without recording into a file
     extract - extract a trace from the kernel
     stop - stop the kernel from recording trace data
     restart - restart the kernel trace data recording
     show - show the contents of the kernel tracing buffer
     reset - disable all kernel tracing and clear the trace buffers
     clear - clear the trace buffers
     report - read out the trace stored in a trace.dat file
     stream - Start tracing and read the output directly
     profile - Start profiling and read the output directly
     hist - show a histogram of the trace.dat information
     stat - show the status of the running tracing (ftrace) system
     split - parse a trace.dat file into smaller file(s)
     options - list the plugin options available for trace-cmd report
     listen - listen on a network socket for trace clients
     agent - listen on a vsocket for trace clients
     setup-guest - create FIFOs for tracing guest VMs
     list - list the available events, plugins or options
     restore - restore a crashed record
     snapshot - take snapshot of running trace
     stack - output, enable or disable kernel stack tracing
     check-events - parse trace event formats
     dump - read out the meta data from a trace file

```

### 使用方法

`trace-cmd`有两种追踪方式：

- `start`和`stop`：记录至ringbuffer中
- `record`和`report`：记录至文件trace.dat，类似于`perf`

显示可被追踪的函数：

```SHELL
$ trace-cmd list -f
```

启用追踪器：

```SHELL
$ trace-cmd start -p function
```

查看输出：

```SHELL
$ trace-cmd show | head -20
```

关闭追踪器：

```SHELL
$ trace-cmd stop
```

清除缓冲区：

```SHELL
$ trace-cmd clear
```

## perf

| 名称 | 说明 |
| --- | --- |
| perf list | 列出所有能够触发perf采样点的事件 |
| perf probe | 定义新的动态tracepoint |
| perf trace | 类似strace，不过性能更强 |
| perf stat | 统计程序运行时的性能信息 |
| pert top | 实时查看当前系统函数占用率情况 |
| perf record | 采样并保存profile到perf.data |
| perf report | 分析perf.data文件 |
| perf script | 读取perf.data并显示详细的采样数据 |

子命令：

`perf list`:

查看当前环境支持的事件，包括硬件事件、软件事件、静态跟踪点等。

`perf stat`：

- -e：指定要统计的事件
- -r：制定运行次数

`perf record`：

- -g：记录调用栈信息
- -F：指定每秒采样频率
- -p：指定进程ID
- -o：指定输出文件

`perf report`：

- -i：指定perf.data文件
- -n：不进行排序
- -v：显示详细信息

### 生成火焰图

1. 使用`perf`记录事件

    ```SHELL
    $ perf record -F 99 -a -g -- sleep 60
    $ perf script > out.perf
    ```

2. 折叠栈

    ```SHELL
    $ ./stackcollapse-perf.pl out.perf > out.folded
    ```

3. 生成火焰图

    ```SHELL
    $ FlameGraph/flamegraph.pl out.folded > out.svg
    ```

4. 在浏览器中打开

### 分析火焰图

- 每一列代表一个调用栈，每一个格子代表一个函数
- Y轴表示调用栈的深度
- X轴表示函数被抽样到的次数，越宽表示被抽样的次数越多
