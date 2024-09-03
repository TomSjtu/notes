# 虚拟内存管理

虚拟内存管理技术允许进程以虚拟地址的方式访问系统中的物理内存，由操作系统负责处理映射关系。在 32 位系统中，一个普通进程可以访问 4GB 的线性空间，按照经典的 3：1 划分，用户态可以访问前 3GB，内核态可以访问后 1GB—— 起始于 OxC00000000，终止于 0xFFFFFFFF。

## 内核线性空间布局

虚拟内存的访问也是有限制的。内核会在启动过程中，将一部分的内存直接映射到线性空间，这部分内存就叫 low memory，剩下的部分叫 high memory。所谓直接映射，指的是映射后的虚拟地址和物理地址有直接的对应关系：va = pa + PAGE_OFFSET。

除了直接映射区，内核还划分了以下几个区：

- 动态映射区
- 永久映射区
- 固定映射区

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

虚拟内存区域（Virtual Memory Area）在内核中用`vm_area_struct`结构体描述。每个`vm_area_struct`结构都对应于指定地址空间上某块[vm_start, vm_end)的虚拟地址区域。`vm_start`指向了这块虚拟内存区域的起始地址，`vm_end`则指向了这块虚拟内存区域的结束地址。

内核将每块内存区域作为一个单独的对象管理，每个区域都有一致的属性和操作。进程运行过程中会不断分配和释放`vm_are_struct`，因此需要使用合适的数据结构对其进行管理。在 Linux 6.1版本之前，一直使用的是红黑树和双向链表来管理`vm_area_struct`对象：

```C title="linux-5.15.10:include/linux/mm_types.h"
struct vm_area_struct {
    ......
    //双向链表
    struct vm_area_struct *mmap;
    //红黑树
    struct rb_root mm_rb;
    struct rw_semaphore mmap_sem;
};
```

整个场景如下所示：

![Alt text](../../images/kernel/vma3.webp)

红黑树+双向链表的搭配提供了不错的性能，一直沿用了很长时间，但也存在缺陷。最明显的是随着 CPU 核心数的增多，应用程序的线程也越来越多，多线程下锁争抢的问题开始浮现出来。需要加锁的原因是红黑树的平衡操作必须是原子的，同时还要将修改同步至双向链表。于是在 Linux 6.1版本中引入了`mapple tree`数据结构，使用 RCU 无锁编程实现。

## 堆内存管理

堆内存管理是操作系统中最为复杂的部分之一。由于程序在运行过程中会涉及大量数据对象的申请和释放，如果仅使用内核提供的`mmap()`和`brk()`函数，会产生大量的内存碎片。因此在应用开发者和内核之间还需要一个内存分配器。`malloc()`函数其实就是 glibc 提供的内存分配器接口。业界有许多优秀的内存分配器，这里只介绍 glibc 内置的 ptmalloc 内存分配器的大致工作原理。

### ptmalloc概念

在 ptmalloc 中，是通过分配去、空闲链表、内存块等几个数据结构来管理内存的。

#### 分配区

多线程在操作分配区的时候需要加锁，为了提升性能，ptmalloc 支持多个分配区：

```C
struct malloc_state {
    mutex_t mutex;
    struct malloc_state *next;
};
```

所有的分配区以一个链表的形式组织起来。

#### 内存块

最基本的内存分配的单位是`malloc chunk`，简称 chunk。它包含 header 和 body 两部分：

```C
struct malloc_chunk{
    INTERNAL_SIZE_T prev_size;  //前一个chunk的大小
    INTERNAL_SIZE_T size;       //当前chunk的大小

    struct malloc_chunk* fd;    //指向上一个空闲的chunk
    struct malloc_chunk* bk;    //指向下一个空闲的chunk

    struct malloc_chunk* fd_nextsize;
    struct malloc_chunk* bk_nextsize;
};
```

每次调用`malloc()`函数申请内存时，分配器都会分配一个大小合适的 chunk，然后将 body 部分的 user data 的地址返回。在调用`free()`释放内存后，chunk 并不会归还给内核，而是由 glibc 管理起来。

#### 空闲链表

相似大小的 chunk 串联成一个空闲链表，下次分配的时候直接从链表中取一个 chunk 即可。这样的一个链表称为 bin。ptmalloc 中根据内存块的大小，总共有 fastbins、smallbins、largebins 和 unsortedbins 四种 bin。

另外还有一个特殊的 bin 称为 top chunk，当没有空闲的 chunk 可用时，或者需要分配的 chunk 大小超过了最大的 bin 时，就会从 top chunk 处分配。

#### malloc

`malloc()`函数会根据要申请内存的大小，从不同的 bin 中查找合适的 chunk。如果没有找到，就向操作系统发起内存申请。


`malloc()`函数在 glibc 中的实现名为`public_mALLOc()`:

```C
Void_t *public_mALLOc(size_t bytes) {
    //选择分配区，并加锁
    arena_lookup(ar_ptr);
    arena_lock(ar_ptr, bytes);

    //从分配区申请内存
    victim = _int_malloc(ar_ptr, bytes);

    //如果分配失败，换一个分配球
    ......

    //释放锁并返回
    mutex_unlock(&ar_ptr->mutex);
    return victim;
}
```

真正的申请逻辑在`_int_malloc()`函数中，这里只列出其骨干逻辑：

```C
static Void_t *_int_malloc(mstate av, size_t bytes){
    //对齐字节数
    INTERNAL_SIZE_T nb;
    checked_request2size(bytes, nb);

    //1.从fastbins中分配
    if((unsigned long)(nb) <= (unsigned long)(get_max_fast())>){
        .......
    }

    //2.从smallbins中分配
    if(in_smallbin_range(nb)){
        .......
    }

    for(;;){
        //3.遍历搜索unsortedbins
        while((victim = unsorted_chunks(av)->bk) != unsorted_chunks(av)){
            //判断是否对chunk进行切割
            .......

            //判断是否精准匹配
            .......

            //若不精准匹配，将对应chunk放到bins中
            .......

            //避免遍历unsortedbins太多时间
            if(++iters >= MAX_ITERS){
                break;
            }
        }
    //4.从largebins中分配
    .......

    //5.尝试从top chunk中分配
use_top:
    victim = av->top;
    size = chunksize(victim);
    .......

    //分配区中申请失败，向操作系统申请
    void *p = sYSMALLOc(nb, av);
    }
}
```

在`sYSMALLOc()`函数中，调用`mmap()`函数向操作系统申请内存：

```C
static Void_t *sYSMALLOc(size_t bytes, mstate av){
    ......
    mm = (char *)(MMAP(0, size, PROT_READ|PROT_WRITE, MAP_PRIVATE));
    ......
}
```