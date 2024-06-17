# ftrace

`ftrace`可追踪的事件源包括`tracepoints`，`kprobes`，`uprobes`，依赖debugfs。

为了启用`ftrace`，你必须在内核配置界面选择：

- CONFIG_FUNCTION_TRACER
- CONFIG_FUNCTION_GRAPH_TRACER
- CONFIG_STACK_TRACER
- CONFIG_DYNAMIC_FTRACE

前端工具：

- /sys/kernel/debug/tracing
- trace-cmd
- perf-tools

## tracing

| 名称 | 描述 |
| --- | --- |
| `available_tracers` | 当前编译及内核的跟踪器列表，`current_tracer`必须是这里面支持的跟踪器。 |
| `current_tracer` | 用于设置或者显示当前使用的跟踪器列表。系统启动缺省值为`nop`，使用`echo`将跟踪器名字写入即可打开。可以通过写入`nop`重置跟踪器。 |
| `buffer_size_kb` | 用于设置单个CPU所使用的跟踪缓存的大小。跟踪缓存为RingBuffer形式，如果跟踪太多，旧的信息会被新的跟踪信息覆盖掉。需要先将`current_trace`设置为`nop`才可以。 |
| `buffer_total_size_kb` | 显示所有的跟踪缓存大小，不同之处在于`buffer_size_kb`是单个CPU的，`buffer_total_size_kb`是所有CPU的和。 |
| `free_buffer` | 此文件用于在一个进程被关闭后，同时释放RingBuffer内存，并将调整大小到最小值。 |
| `hwlat_detector/` |  |
| `instances/` | 创建不同的trace buffer实例，可以在不同的trace buffers中分开记录。 |
| `tracing_cpumask` | 可以通过此文件设置允许跟踪特定CPU，二进制格式。 |
| `per_cpu` | CPU相关的trace信息，包括`stats`、`trace`、`trace_pipe`和`trace_pipe_raw`。`stats`：当前CPU的trace统计信息；`trace`：当前CPU的trace文件；`trace_pipe`：当前CPU的trace_pipe文件。 |
| `printk_formats` | 提供给工具读取原始格式trace的文件。 |
| `saved_cmdlines` | 存放pid对应的comm名称作为ftrace的cache，这样ftrace中不光能显示pid还能显示comm。 |
| `saved_cmdlines_size` | `saved_cmdlines`的数目。 |
| `snapshot` | 对trace的snapshot。`echo 0`清空缓存，并释放对应内存；`echo 1`进行对当前trace进行snapshot，如没有内存则分配；`echo 2`清空缓存，不释放也不分配内存。 |
| `trace` | 查看获取到的跟踪信息的接口，`echo > trace`可以清空当前RingBuffer。 |
| `trace_pipe` | 输出和`trace`一样的内容，但是此文件输出Trace同时将RingBuffer中的内容删除，这样就避免了RingBuffer的溢出。可以通过`cat trace_pipe > trace.txt &`保存文件。 |
| `trace_clock` | 显示当前Trace的timestamp所基于的时钟，默认使用local时钟。`local`：默认时钟；可能无法在不同CPU间同步；`global`：不同CPU间同步，但是可能比local慢；`counter`：这是一个跨CPU计数器，需要分析不同CPU间event顺序比较有效。 |
| `trace_marker` | 从用户空间写入标记到trace中，用于用户空间行为和内核时间同步。 |
| `trace_marker_raw` | 以二进制格式写入到trace中。 |
| `trace_options` | 控制Trace打印内容或者操作跟踪器，可以通过`trace_options`添加很多附加信息。 |
| `options` | `trace`选项的一系列文件，和`trace_options`对应。 |
| `trace_stat/` | 每个CPU的Trace统计信息。 |
| `tracing_max_latency` | 记录Tracer的最大延时。 |
| `tracing_on` | 用于控制跟踪打开或停止，`0`停止跟踪，`1`继续跟踪。 |
| `tracing_thresh` | 延时记录Trace的阈值，当延时超过此值时才开始记录Trace。单位是ms，只有非0才起作用。 |
| `available_events` | 列出系统中所有可用的Trace events，分两个层级，用冒号隔开。 |
| `events/` | 系统Trace events目录，在每个events下面都有`enable`、`filter`和`format`。`enable`是开关；`format`是events的格式，然后根据格式设置`filter`。 |
| `set_event` | 将Trace events名称直接写入`set_event`就可以打开。 |
| `set_event_pid` | 指定追踪特定进程的events。 |
| `available_filter_functions` | 记录了当前可以跟踪的内核函数，不在该文件中列出的函数，无法跟踪其活动。 |
| `dyn_ftrace_total_info` | 显示`available_filter_functins`中跟中函数的数目，两者一致。 |
| `enabled_functions` | 显示有回调附着的函数名称。 |
| `function_profile_enabled` | 打开此选项，在`trace_stat`中就会 | 

### 常用配置

| 名称 | 说明 | 
| ---- | ---- |
| available_tracers | 跟踪器列表，current_tracer必须是这里面支持的跟踪器 |
| available_events | 跟踪事件列表 |
| available_filter_functions | 跟踪函数列表 |
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

使用`ftrace`需要对文件进行频繁的写入和读出，效率很低。而`trace-cmd`是对`ftrace`的封装，使用起来更加方便。


### 使用方法

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

显示可被追踪的函数：

```SHELL
$ trace-cmd list -f
```

## perf-tools

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

`perf stat`：

- -e：指定要统计的事件
- -r：制定运行次数

`perf record`：

- -g：记录调用栈信息
- -F：指定采样频率
- -p：指定进程ID
- -o：指定输出文件

`perf report`：

- -i：指定perf.data文件
- -n：不进行排序
- -v：显示详细信息

