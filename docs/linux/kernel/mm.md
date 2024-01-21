# 物理内存管理

> 这部分内容和[进程地址空间](./vma.md)结合起来看。

## 页和区

我们知道，Linux内核是以页为单位来组织和管理内存的。在Linux系统中，内存被划分为许多固定大小的页，每一页通常是4KB。这种组织方式允许内核更加高效地进行内存分配和回收，同时也便于实现虚拟内存的管理。内核使用**struct page**结构体来表示每个页，该定义的简化版如下：

```C
struct page {
    unsigned long flags;
    atomic_t _count;
    atomic_t _mapcount;
    unsigned long private;
    struct address_space *mapping;
    pgoff_t index;
    struct list_head lru;
    void *virtual;
};
```

> flags：存放页的状态，是不是脏页，是不是锁定在内存中等。

> _count：存放页的引用计数，当页被分配给一个或多个进程时，这个计数器会增加。当页被释放时，计数器减少。直到计数器降至0，页才能被回收。

> _mapcount：用于记录页表项映射到该物理页的次数。这通常用于共享内存页，多个进程可能映射到同一个物理页。

> mapping：指向一个address_space结构的指针，这个结构体用于描述与文件相关的内存映射信息。如果页属于文件缓存，这个字段将指向相应的address_space结构。

> index：表示页在映射的文件中的偏移量，即页在文件中的索引号。

> lru：一个双向链表节点，用于将页链入到各种链表中，比如活跃链表、不活跃链表等。这是内核实现页置换算法（如LRU）的一部分。

> virtual：页在虚拟内存中的地址。

为了更好地管理不同类型的内存和优化内存分配策略，内核将页划分为不同的**区**(zone)，通常划分为以下几个区：

- ZONE_DMA：用于DMA操作的内存区，通常位于物理内存的低地址部分。

- ZONE_DMA32：与ZONE_DMA类似，但适用于32位地址的DMA操作。

- ZONE_NORMAL：普通的内存区，可以由内核和用户空间进程使用。

- ZONE_HIGHMEM：高端内存区，用于处理超过直接映射范围的内存。

区的使用和分布与体系结构密切相关，内核将页划分为区，就可以根据用途进行分配。区的划分没有物理意义，只是为了方便管理而采取的一种逻辑分组。

## 页操作

内核提供了一些获得页和释放页的操作，这里不做详细说明。

| 函数 | 描述 |
| --- | --- |
| alloc_page(gfp_mask) | 只分配一页，返回指向页的指针 |
| alloc_pages(gfp_mask, order) | 分配2<sup>order</sup>个页，返回指向第一页的指针 |
| __get_free_page(gfp_mask) | 只分配一页，返回指向其逻辑地址的指针 |
| __get_free_pages(gfp_mask, order) | 分配2<sup>order</sup>个页，返回指向第一页逻辑地址的指针 |
| get_zero_page(gfp_mask) | 只分配一页，填充为0，返回指向其逻辑地址的指针 |
| __free_pages(page, order) | 传入页的指针，释放2<sup>order</sup>个页 |
| free_pages(addr, order) | 传入第一页的逻辑地址，释放2<sup>order</sup>个页 |
| free_page(addr) | 释放单张页 |

释放页时要谨慎，一旦传递了错误的page或者address，系统就会崩溃。在获得页之后，需要对返回值进行检查以确认内核正确地分配了页。

对于常用的以字节为单位的内存分配来说，内核提供的函数是`kmalloc()`。

## kmalloc()

`kmalloc()`与用户空间的`malloc()`类似，都是分配以字节为单位的一块内存，区别在于`kmalloc()`多了一个flags参数：

> 注意：使用`kmalloc()`函数分配的内存只能使用`kfree()`函数释放。

```C
void *kmalloc(size_t size, gfp_t flags)
```

该函数返回一个指向内存块的指针，至少有size大小。所分配的内存区在物理上是连续的。除非没有足够的内存可用，否则内核总能分配成功。当然，在对`kmalloc()`调用之后，你还是需要检查返回值是否为NULL：

```C
struct dog *p;
p = kmalloc(sizeof(struct dog), GFP_KERNEL)；
if(!p)
    /*处理错误*/
```

不管是在页分配函数还是在`kmalloc()`中，都用到了分配器标志。标志分为三类：行为修饰符、区修饰符和类型标志。类型标志组合了前两者，简化了修饰符的使用，因此我们只需要知道类型标志即可。内核中最常用的就是GFP_KERNEL。这种分配方式可能会引起睡眠，所以只能用在可以重新安全调度的进程上下文中。另一个截然相反的标志是GFP_ATOMIC，这个标志表示不能睡眠的内存分配。与GFP_KERNEL相比，它分配成功的机会较小，但是在一些无法睡眠的代码中，也只能选择GFP_ATOMIC。GFP_DMA标志表示分配器必须满足从ZONE_DMA进行分配的请求，该标志用在需要DMA的内存的设备驱动程序中。在编写的绝大多数代码中，要么是GFP_KERNEL，要么是GFP_ATOMIC，其他标志用到的情况极少，就不做说明了。下面这张表格总结了标志的使用场景。

| 情形 | 相应标志 |
| --- | --- |
| 可以睡眠的进程上下文 | GFP_KERNEL |
| 不可以睡眠的进程上下文 | GFP_ATOMIC |
| 中断处理程序 | GFP_ATOMIC |
| 软中断 | GFP_ATOMIC |
| tasklet | GFP_ATOMIC |
| 可以睡眠的DMA内存 | （GFP_DMA \| GFP_KERNEL） |
| 不可以睡眠的DMA内存 | （GFP_DMA）\| GFP_ATOMIC |

## vmalloc()





