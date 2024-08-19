# 进程地址空间

除了管理本身的内存外，操作系统还必须管理用户空间的内存。我们称这个内存为进程地址空间。几乎所有的操作系统都采用了虚拟内存管理技术，它允许进程以虚拟地址的方式访问系统中的物理内存，由操作系统负责处理映射关系。

在没有虚拟内存的时代，进程访问的是实际的物理内存。开发者在编写程序时必须非常小心，防止访问到了不该访问的内存地址。同时，不同进程之间的内存地址也可能发生冲突，造成程序出错甚至是系统崩溃。

虚拟内存管理技术的出现，大大简化了程序员的工作——在编写程序时，无需考虑内存分配和释放，只需使用虚拟地址即可。

Linux采用了基于分页的虚拟内存管理机制，其中内存被划分为大小固定的页，每一页通常是 4KB。这种分页机制使得内存管理更加灵活和高效。而虚拟地址到物理地址的转换则通过多级页表来实现。为了提高地址转换的速度，现代 CPU 都会设计一个称为 {==TLB==}（转换后援缓冲器）的硬件缓存。TLB 缓存了最近使用过的页表项，这样当 CPU 需要访问一个内存地址时，它首先检查 TLB，如果找到了匹配的条目——这个操作被称为 {==TLB命中==}，就可以直接找到对应的物理地址，从而避免访问页表。

## 地址空间的概念

进程的地址空间由可寻址的虚拟内存组成，在32位体系中（只讨论32位架构，64位类似），每个进程可寻址 4GB 的虚拟内存，其中 1GB 分配给内核空间，剩下 3GB 分配给用户空间。一个进程只能访问有效内存区域内的内存地址，部分地址是非法的。一旦访问，就会立刻触发{==段错误==}(segmentation fault)，由内核终止该进程。每个内存区域还有相关的权限，比如可读可写可执行等。

进程的虚拟内存空间分布如下图所示：

![进程虚拟内存空间](../../images/kernel/vma.webp)

从上图中我们看到，0x0000 0000 到 0x0804 8000 这段空间是不可访问的{==保留区==}，在大多数操作系统中，数值较小的地址是不允许访问的。比如 NULL 指针就会指向这片区域。

0x0804 8000 至 0xC000 0000 是用户态空间地址，在往上就是所谓的高端地址——供内核使用。内核态与用户态的分界线由成员变量`task_size`定义。

保留区上方是{==代码段==}和{==数据段==}，它们是从可执行文件直接加载进来的。编译后的代码放在代码段，数据段用来存放已初始化的全局变量。

{==BSS==} 段用来存放未初始化的全局变量，以0填充。

BSS 段上方是{==堆空间==}，地址的增长方向是从低到高。内核使用`start_brk`标识堆的起始位置，`brk`标识堆当前的结束位置。当用户使用`malloc()`函数申请一块很小的内存（128K以内）时，就会在堆上分配内存。

堆空间上方是待分配区域，用来扩展堆空间的使用。接下来是{==内存映射区域==}。任何应用程序都可以通过`mmap()`系统调用映射至此区域。内存映射区可以用来加载动态库，比如{==ld-linux.so==}就被加载于此。

最后一块区域是{==栈空间==}，在这里保存函数运行过程需要的局部变量以及函数参数等信息。栈空间的地址是从高到低增长的。内核使用`start_stack`标识栈的起始位置。

你可以使用`cat /proc/[pid]/maps`查看一个进程的内存布局。

## 内存描述符

`mm_struct`用来表示进程的地址空间的信息，定义在文件<linux/mm_types.h\>中：

```C
struct mm_struct {
    atomic_t mm_users;
    atomic_t mm_count;
    struct vm_area_struct *mmap;
    struct rb_root mm_rb;
    struct list_head mmlist;

    unsigned long start_code, end_code;    //代码段的起始和结束地址
    unsigned long start_data, end_data;    //数据段的起始和结束地址
    unsigned long start_brk, brk,          //堆的起始和结束地址
    unsigned long start_stack;             //栈的起始地址
    unsigned long arg_start, arg_end,      //命令行参数的起始和结束地址
    unsigned long env_start, env_end;      //环境变量的起始和结束地址
    unsigned long mmap_base;               //内存映射区的起始地址
    unsigned long total_vm;                //总共映射的页数目
    unsigned long locked_vm;               //被锁定不能换出的页数目
    unsigned long pinned_vm;               //既不能换出，也不能移动的页数目
    unsigned long data_vm;                 //数据段中映射的页数目
    unsigned long exec_vm;                 //代码段中可执行文件的页数目
    unsigned long stack_vm;                //栈中映射的页数目
};
```

> mm_users：正在使用该地址的进程数。

> mm_count：mm_struct结构体的主引用计数，该值为0时，释放结构体。

> mmap：指向一个`vm_area_struct`结构体的链表头。

> mm_rb：指向一个`vm_area_struct`结构体的红黑树根节点。

> mmlist：指向一个`mm_struct`结构体的链表头。

在[进程管理](./task.md)中我们介绍了 mm 字段，用来存放该进程使用的内存描述符。如果父子进程共享地址空间，则在调用`clone()`函数时，会设置{==CLONE_VM==}标志，这样的进程就是线程。一旦指定了 CLONE_VM，线程就不需要另外分配地址空间了，而是将 mm 字段直接指向父进程的内存描述符即可。

内核线程没有进程地址空间，也没有相关的内存描述符，其 mm 字段为 NULL。当一个内核线程被调度时，它会直接使用前一个进程的内存描述符。

## 虚拟内存区域

虚拟内存区域（Virtual Memory Area）在内核中用`vm_area_struct`结构体描述。每个`vm_area_struct`结构都对应于指定地址空间上某块连续的内存区域。`vm_start`指向了这块虚拟内存区域的起始地址，`vm_end`则指向了这块虚拟内存区域的结束地址。`vm_area_struct`结构体描述的是[vm_start，vm_end)这样一段左闭右开的虚拟内存区域。

内核将每个内存区域作为一个单独的对象管理，每个区域都有一致的属性和操作，下面给出`vm_area_struct`结构体的定义：

```C
struct vm_area_struct {
    struct mm_struct *vm_mm;                      //相关的mm_struct结构体
    unsigned long vm_start;                       //区间的首地址
    unsigned long vm_end;                         //区间的尾地址
    struct vm_area_struct *vm_next;               //VMA链表
    pgprot_t vm_page_prot;                        //访问控制权限
    unsigned long vm_flags;                       //描述该区域的标志，重要的两个是VM_IO和VM_RESERVED
    struct rb_node vm_rb;                         //红黑树节点
    struct anon_vma *anon_vma;                    //匿名VMA对象
    const struct vm_operations_struct *vm_ops;    //VMA的操作函数
    unsigned long vm_pgoff;                       //文件中该区域的偏移量
    struct file *vm_file;                         //指向与该区域相关联的file结构指针
    void *vm_private_data;                        //私有数据
};
```

于是整个虚拟内存空间管理如下图所示，在每个`task_struct`结构体下的 mm 字段指向了该进程的`mm_struct`。内核用两种数据结构管理`vm_area_struct`结构体：`mm_rb`指向红黑树头节点，`mmap`指向链表头节点。红黑树和链表的每个元素都是一个`vm_area_struct`结构体，标识了每块VMA区域的属性：

![Alt text](../../images/kernel/vma3.webp)

### VMA标志

vm_flags 定义了VMA的标志，它表示内存区域的行为和信息。一些比较重要的标志比如：VM_READ、VM_WRITE 和 VM_EXEC 定义了内存区域中页面的可读、可写、可执行权限。

VM_SHARED 表示内存区域包含的映射是否可以在多进程之间共享，如果设置了该标志位，我们称其为共享映射，否则就是私有映射。

VM_IO 标志内存区域中对设备I/O空间的映射。该标志通常在设备驱动程序执行`mmap()`函数时才被设置。

VM_RESERVED 标志规定了内存区域不能被换出。

### VMA操作

`vm_ops`域指向与内存区域相关的操作函数，由`vm_operations_struct`结构体表示：

```C
struct vm_operations_struct {
    void (*open)(struct vm_area_struct *);
    void (*close)(sturct vm_area_struct *);
    int (*fault)(struct vm_area_struct *, struct vm_fault *);
    int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
    int (*access)(struct vm_area_struct *, unsigned long, void *, int, int);
};
```

## 进程内存指标

在进程的内存管理中，通常被分为几个不同的类别，其中 RSS(Resident Set Size)、PSS(Proportional Set Size)、USS(Unique Set Size) 是衡量进程内存使用情况的三个重要指标。

传统模式下，应用程序如果需要访问文件，需要先在虚拟地址空间中分配一段虚拟内存，这段内存通过 MMU，映射成对应的物理内存。当进程发出读/写请求时，磁盘先将数据拷贝到 page cache，再将 page cache 中的内容拷贝到物理内存。当使用了`mmap()`之后，应用程序可以直接访问磁盘上的文件，而不需要拷贝到 page cache。

1. RSS：进程实际占用的物理内存，包括了共享库所占用的内存
2. PSS：进程独占的内存加上按比例分摊的共享库内存
3. USS：进程独自占用的物理内存

## malloc()

`malloc()`是C库里的一个动态分配内存的函数，在申请内存时有两种方式：

- 方式一：通过`brk()`系统调用从堆分配内存。

- 方式二：通过`mmap()`系统调用从文件映射区分配内存。

`malloc()`源码规定了一个阈值 MMAP_THRESHOLD，默认为 128kB：

- 如果分配的内存小于这个阈值，则通过`brk()`申请内存，动态调整进程地址空间中的brk指针的位置。

- 如果分配的内存大于这个阈值，则通过`mmap()`申请内存，在堆和栈之间找到一片区域进行映射处理。

设计内存分配策略时，核心目标是在性能和资源管理之间找到最佳平衡点。对于小内存块的频繁分配与释放操作，`malloc()` 采用 `brk()` 系统调用进行内存管理。释放的内存并不会立即返回给操作系统，而是被缓存至内存池中，以便于后续的分配请求能够快速重用这些已释放的内存块，从而减少系统调用的次数和相关的性能开销。

对于大内存块的分配，`malloc()` 则会使用 `mmap()` 系统调用。这种方式在首次访问时可能会触发{==缺页异常==}，因为分配的内存初始处于未映射状态，需要操作系统介入以建立虚拟地址到物理内存的映射。这一机制相较于 `brk()` 系统调用，会带来更高的性能成本。

## mmap()

`mmap()`用于内存映射，将一块区域映射到自己的进程地址空间中，分为两种：

- 文件映射：将磁盘上的文件映射到进程的地址空间中。
- 匿名映射：不映射文件，映射物理内存

针对其他进程是否可见，又分为：

- 私有映射
- 共享映射

于是`mmap()`主要有四种工作模式：

- 私有匿名映射：通常用于分配大块内存
- 共享匿名映射：常用于父子进程通信
- 私有文件映射：常用于动态库加载
- 共享文件映射：常用于进程间通信

`mmap()`的函数原型如下：

```C
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

> addr：想要映射的内存起始地址，通常设置为NULL，即由系统自动选定

> length：映射到内存的文件长度

> prot：映射区的保护方式，可以是PROT_READ、PROT_WRITE等

> flags：映射选项：可以是MAP_SHARED、MAP_PRIVATE、MAP_ANONYMOUS等

> fd：想要映射的文件描述符

> offset：想要映射的文件内的偏移量


