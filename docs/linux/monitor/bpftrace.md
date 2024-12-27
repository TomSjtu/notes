# bpftrace

bpftrace 是基于 BPF 和 BCC 的开源跟踪器，特别适合用来创建单行程序和短小脚本。

bpftrace 程序的语法简洁，仅包含头部、操作块和尾部三个部分。头部和尾部是可选的，通常用来打印一些加载前后的信息。操作块是我们要指定跟踪的探针的位置，以及基于探针触发事件后执行的操作。下面是一个简单的示例程序：

```SHELL
BEGIN
{
       printf("starting bpftrace program\n")
}

kprobe:do_sys_open
{
       printf("opening file descriptor: %s\n", str(arg1))
}

END
{
       printf("ending bpftrace program\n")
}
```

你可以将这个程序保存为 example.bt，然后运行：`bpftrace example.bt`。

bpftrace 语言是以脚本思想设计的。有的时候可以不用编写程序，直接在命令行中使用选项`-e`即可：

```SHEll
$ bpftrace -e 'kprobe:do_sys_open { printf("opening file descriptor: %s\n", str(arg1)) }'
```

## 单行程序

单行程序的语法为：

```SHELL
$ bpftrace -e program
```

program 的结构是一系列探针及其对应的动作，动作可以是单个语句，也可以是用分号分隔的多条语句：

```
probes { actions}
probes { actions}
```

探针以类型名字开始，由冒号分隔：

```SHELL
type:identifier1[:identifier2[...]]
```

kprobe 只需要内核函数名，而 uprobe 需要二进制文件路径和函数名。

可以为表达式添加过滤器，以便只处理满足条件的事件：

```SHELL
/pid == 123 /
```

## 变量

bpftrace 提供了三种变量类型：内置变量、临时变量和映射表变量。

| 内置变量 | 类型 | 描述 |
| --- | --- | --- |
| pid | integer | 进程ID |
| tid | integer | 线程ID |
| uid | integer | 用户ID |
| username | string | 用户名 |
| nsecs | integer | 时间戳，单位纳秒 |
| elapsed | integer | 时间戳，单位纳秒，自启动开始计时 |
| cpu | integer | CPU ID |
| comm | string | 进程名 |
| kstack | string | 内核调用栈信息 |
| ustack | string | 用户调用栈信息 |
| arg0, ..., argN | integer | 探针参数 |
| args | struct | 探针参数 |
| retval | integer | 探针返回值 |
| func | string | 被跟踪函数的名字 |
| probe | string | 探针的名字 |


## 函数

bpftrace 提供了一些内置函数：

| 函数 | 描述 |
| --- | --- |
| printf(char *fmt [,...]) | 格式化输出 |

## 映射表

映射表是 BPF 的一种特殊类型的哈希表存储对象，可以用来存储键值对和统计数值，一些重要的映射表函数如下所示：

| 函数 | 描述 |
| --- | --- |
| count() | 对出现次数进行计数 |
| sum(int n ) | 求和 |
| min(int n ) | 最小值 |
| max(int n ) | 最大值 |
| avg(int n ) | 平均值 |

## bpftool

bpftool 提供内核中与 BPF 程序相关的信息。

```SHELL
$ bpftool --help
Usage: 
       OBJECT := { prog | map | link | cgroup | perf | net | feature | btf | gen | struct_ops | iter }
       OPTIONS := { {-j|--json} [{-p|--pretty}] | {-d|--debug} |
                    {-V|--version} }

```

检查系统中运行程序的情况：

```SHELL
$ bpftool prog show
```

以 JSON 形式输出：

```SHELL
$ bpftool prog show --json id 30 | jq
```

检查 BPF 映射：

```SHELL
$ bpftool map show
```

列出系统中所有 cgroup 上的附加程序：

```SHELL
$ bpftool cgroup tree
```

