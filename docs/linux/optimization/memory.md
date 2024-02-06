# 内存性能调优


## 内存性能指标

### Buffer/Cache

Buffer和Cache被设计用来提升系统I/O的性能，它们缓存在内存中，充当起慢速磁盘与快速CPU之间的桥梁。

- Buffers是对**磁盘数据**的缓存，通常不会特别大。

- Cached是对**文件数据**的缓存。

不论是读还是写，应用程序都优先从缓存中读取，这样可以避免访问低速的磁盘，从而提高系统的I/O性能。

## 内存性能分析工具

- 使用`free`查看系统内存使用情况：

```SHELL
$ free
              total        used        free      shared  buff/cache   available
Mem:        8169348      263524     6875352         668     1030472     7611064
Swap:             0           0           0
```

> total：总内存大小

> used：已经使用的内存大小

> free：空闲的内存大小

> shared：多个进程共享的内存大小

> buff/cache：缓存和缓冲区占用的内存大小

> available：可用的内存大小


- 使用`top`查看进程内存使用情况：
  
```SHELL
$ top
...
KiB Mem :  8169348 total,  6871440 free,   267096 used,  1030812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7607492 avail Mem


  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  430 root      19  -1  122360  35588  23748 S   0.0  0.4   0:32.17 systemd-journal
 1075 root      20   0  771860  22744  11368 S   0.0  0.3   0:38.89 snapd
 1048 root      20   0  170904  17292   9488 S   0.0  0.2   0:00.24 networkd-dispat
    1 root      20   0   78020   9156   6644 S   0.0  0.1   0:22.92 systemd
12376 azure     20   0   76632   7456   6420 S   0.0  0.1   0:00.01 systemd
12374 root      20   0  107984   7312   6304 S   0.0  0.1   0:00.00 sshd
...
```

> VIRT：进程使用的虚拟内存大小，申请过但尚未分配物理内存也会计算在内。

> RES：进程实际使用的物理内存大小。

> SHR：进程使用的共享内存大小，比如进程间通信。

> %MEM：进程使用的物理内存占系统总内存的百分比。


- 使用`cachestat`和`cachetop`查看缓存命中情况：

这两个工具是bcc软件包的一部分，基于ebpf技术。

```SHELL
$ cachestat 1 3
   TOTAL   MISSES     HITS  DIRTIES   BUFFERS_MB  CACHED_MB
       2        0        2        1           17        279
       2        0        2        1           17        279
       2        0        2        1           17        279 
```

> TOTAL：总缓存次数。

> MISSES：缓存未命中的次数。

> HITS：缓存命中的次数。

> DIRTIES：缓存中dirty pages脏页）的次数。

> BUFFERS_MB：缓存Buffers的大小，单位是MB。

> CACHED_MB：缓存中Cache的数量，单位是MB。


```SHELL
$ cachetop
11:58:50 Buffers MB: 258 / Cached MB: 347 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   13029 root     python                  1        0        0     100.0%       0.0%
```

> READ_HIT：读缓存命中率。

> WRITE_HIT：写缓存命中率。




