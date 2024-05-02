# OOM

OOM，即out of memory的缩写。尽管Linux内核有各种内存管理技巧来应对应用程序的内存需求，但是仍然会面临内存耗尽的情况，最典型的就是程序发生内存泄露，导致系统需要不断重启。

对于内核而言，触发OOM会根据`panic_on_oom`参数做相应的处理。它有两种选择：

1. 当该参数为2时：产生kernel panic，即系统崩溃
2. 当该参数为0时：祭出终极大杀器OOM killer，干掉一些进程，释放内存，让系统还能正常运转

当该参数为其他值时，表示要分情况处理。`enum oom_constraint`定义了四种情况：

```C
enum oom_constraint {
    CONSTRAINT_NONE,
    CONSTRAINT_CPUSET,
    CONSTRAINT_MEMORY_POLICY,
    CONSTRAINT_MEMCG,
}; 
```

对于UMA架构而言，`oom_constraint`永远是CONSTRAINT_NONE，因为UMA架构，内存是统一管理的，触发OOM只可能是内存不足。

对于NUMA架构就复杂得多，因为NUMA架构下，内存根据不同的node分区管理。当触发OOM，仅能说明当前分配内存的node出现了状况，而其它node可能有充足的内存。

`CONSTRAINT_CPUSET`：cpuset是内核的一种机制，将特定的cpu和memory node分配给特定的进程。

`CONSTRAINT_MEMORY_POLICY`：NUMA架构下的memory node的分配策略，可以针对特定的进程。

`CONSTRAINT_MEMCG`：memcg就是memory control group，可以限制进程的cpu和内存资源。

## killer策略

当系统启动OOM killer，选择要杀死的进程时，一个很头疼的问题就是，到底杀掉哪个进程？

随之又引申出另一个问题：是不是所有的进程都可以杀？显然不是，内核线程就不能杀，一旦杀了整个系统就崩了；还有一些进程设定为unkillable，也不能杀，比如init进程。

除了这些，剩下的进程都是可以杀的，杀掉哪个才合适呢？内核有两种选择：

- 谁触发了OOM就干掉谁
- 谁最耗费性能就干掉谁

`oom_kill_allocating_task`参数为0时选择策略2，否则选择策略1。