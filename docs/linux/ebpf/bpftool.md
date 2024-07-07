# BPF工具

## bpftool

bpftool提供内核中与BPF程序相关的信息。

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

以JSON形式输出：

```SHELL
$ bpftool prog show --json id 30 | jq
```

检查BPF映射：

```SHELL
$ bpftool map show
```

列出系统中所有cgroup上的附加程序：

```SHELL
$ bpftool cgroup tree
```

## bpftrace

bpftrace是BPF高级跟踪语言，可以使用简明的语法来编写程序。它提供了许多内置功能，无须自己实现。

bpftrac程序的语法简洁，仅包含头部、操作块和尾部三个部分。头部和尾部是可选的，通常用来打印一些加载前后的信息。操作块是我们要指定跟踪的探针的位置，以及基于探针触发事件后执行的操作。下面是一个简单的示例程序：

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

你可以将这个程序保存为example.bt，然后运行：`bpftrace example.bt`。

bpftrace语言是以脚本思想设计的。编写程序当然可以，但是有的时候不必这么麻烦，用bpftrace编写的许多程序可以放在一行中执行，通过使用选项`-e`即可：

```SHEll
$ bpftrace -e 'kprobe:do_sys_open { printf("opening file descriptor: %s\n", str(arg1)) }'
```

