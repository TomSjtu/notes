# 动态跟踪

## kprobes

kprobes 几乎可以在内核或者模块的所有函数中插入指令，它提供了调用前、调用后和访问出错三种回调方式。这里只介绍利用 ftrace 工具进行 kprobes 的方法。在 /sys/kernel/tracing 目录下，我们关心以下几个文件：

1. /sys/kernel/tracing/kprobe_events：用于配置要探测的函数
2. /sys/kernel/tracing/events/kprobes/enable：用于开启探测
3. /sys/kernel/tracing/trace：用于查看探测结果

下面以一个例子说明用法：

```SHELL
$ echo 'p:myprobe do_sys_open dfd=%ax filename=%dx flags=%cx mode=+4($stack)' > /sys/kernel/tracing/kprobe_events
$ ls -l /sys/kernel/tracing/events/kprobes
total 0
-rw-r----- 1 root root 0 Dec  6 18:01 enable
-rw-r----- 1 root root 0 Dec  6 18:01 filter
drwxr-x--- 2 root root 0 Dec  6 18:01 myprobe

$ ls -l /sys/kernel/tracing/events/kprobes/myprobe
total 0
-rw-r----- 1 root root 0 Dec  6 18:01 enable
-rw-r----- 1 root root 0 Dec  6 18:01 filter
-r--r----- 1 root root 0 Dec  6 18:01 format
-r--r----- 1 root root 0 Dec  6 18:01 hist
-r--r----- 1 root root 0 Dec  6 18:01 id
--w------- 1 root root 0 Dec  6 18:01 inject
-rw-r----- 1 root root 0 Dec  6 18:01 trigger
```

- enable：使能
- filter：过滤器
- format：输出格式
- hist：历史记录
- id：探测点ID
- inject：注入BPF程序
- trigger：触发器

```SHELL
$ cat /sys/kernel/tracing/events/kprobes/myprobe/format 
name: myprobe
ID: 1527
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:unsigned long __probe_ip; offset:8;       size:8; signed:0;
        field:u64 dfd;  offset:16;      size:8; signed:0;
        field:u64 filename;     offset:24;      size:8; signed:0;
        field:u64 flags;        offset:32;      size:8; signed:0;
        field:u64 mode; offset:40;      size:8; signed:0;

print fmt: "(%lx) dfd=0x%Lx filename=0x%Lx flags=0x%Lx mode=0x%Lx", REC->__probe_ip, REC->dfd, REC->filename, REC->flags, REC->mode
```

```SHELL
$ echo 1 > /sys/kernel/tracing/events/kprobes/myprobe/enable
$ echo 1 > /sys/kernel/tracing/tracing_on
$ cat /sys/kernel/tracing/trace_pipe
```

通过读取 trace_pipe 文件，我们就可以看到输出。

## uprobes

uprobes 提供了用户态程序的动态插桩，可以在函数入口、特定偏移处以及函数返回处插入指令。uprobes 提供以下两个接口：
- 基于 ftrace：通过/sys/kernel/tracing/uprobe_events 写入特定字符串
- `perf_event_open()`：类似于 perf 工具

uprobes 为 BCC 和 bpftrace 提供的支持包括：

- BCC：`attach_uprobe()`和`attach_uretprobe()`
- bpftrace：uprobe 和 uretprobe 探针类型

