# 进程地址空间

内核除了管理本身的内存外，还必须管理用户空间中进程的内存。我们称这个内存为进程地址空间。Linux操作系统采用虚拟内存技术，对一个进程而言，它以虚拟的方式访问整个系统的所有物理内存，甚至其虚拟空间可以远远大于物理内存。

为什么要有虚拟内存？

我们想象一下如果两个程序操作的都是物理内存，那么我们必须非常谨慎地给这两个程序分配各自的内存地址，并且要牢记哪个程序操作了哪一部分的内存，防止两个程序访问同一个内存地址而造成冲突。显然，这样的方式让程序的编写变得非常复杂，因为程序员需要手动管理每一个程序的内存地址。如果我们有一种机制可以把进程之间使用的地址**隔离**开来，由操作系统为每个进程分配一套独立的虚拟地址，大家都使用自己的地址，互不干涉。这就给程序的编写带来了极大的便利，这套机制就是虚拟内存管理机制，而操作系统的MMU（内存管理单元）负责将虚拟地址翻译成物理地址。

Linux采用了内存分页技术，把内存分为大小固定的页，每一页的大小为4KB。同时使用了多级页表技术，虚拟地址要转换到物理地址需要到页表去查询。为了提高查询速度，在CPU芯片中集成了TLB，缓存了最近查询的页表项。

## 地址空间的概念

进程的地址空间由可寻址的虚拟内存组成，在32位体系中（只讨论32位架构，64位类似），每个进程可寻址4GB的虚拟内存，其中1GB分配给内核空间，剩下3GB分配给用户空间。一个进程只能访问有效内存区域内的内存地址，部分地址是非法的，一旦访问，就会立刻触发段错误，由内核终止该进程。每个内存区域也有相关的权限，比如可读可写可执行等。

进程的虚拟内存空间分布如下图所示：

![进程虚拟内存空间](../../images/kernel/vma.webp)

从上图中我们看到，0x0000 0000到0x0804 8000这段空间是不可访问的保留区，在大多数操作系统中，数值比较小的地址是非法地址，不允许访问的。比如NULL指针就会指向这片区域。

保留区上方是代码段和数据段，它们是从可执行文件直接加载进来的。编译后的代码放在代码段，数据段用来存放已初始化的全局变量。

BSS段用来存放未初始化的全局变量，以0填充。

BSS段上方是堆空间，地址的增长方向是从低到高。内核使用**start_brk**标识堆的起始位置，**brk**标识堆当前的结束位置。当堆申请一块很小的内存（128K以内），只需要将**brk**指针增加对应的大小即可。堆内存的管理比较复杂，堆内存最大的困难就是频繁地分配和释放会造成内存碎片。为了应对这个问题，内核采用了基于伙伴系统的内存分配算法。

堆空间上方是待分配区域，用来扩展堆空间的使用。接下来是内存映射区域。任何应用程序都可以通过`mmap()`系统调用映射至此区域。内存映射可以用来加载动态库，比如**ld-linux.so**就被加载于此，另外，如果你通过`malloc()`申请了超过128K内存，内核将直接为你分配一块映射区域作为内存，而不是使用堆内存。

最后一块区域是栈空间，在这里保存函数运行过程需要的局部变量以及函数参数等信息。栈空间的地址是从高到低增长的。内核使用**start_stack**标识栈的起始位置。SP寄存器保存栈顶指针，BP寄存器保存栈基地址。

## 内存描述符mm_struct

**mm_struct**用来表示进程的地址空间的信息，定义在文件<linux/mm_types.h\>中：

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

**mm_users**记录正在使用该地址的进程数目，比如如果有两个线程共享该地址空间，那么**mm_users**的值便等于2。**mm_count**是**mm_struct**结构体的主引用计数，当值为0时，该结构体会被释放。

**mmap**使用单独链表连接所有的内存区域对象。每一个**vm_area_struct**结构体通过自身的**vm_next**指针被连入链表。**mmap**指向链表中第一个节点。

**mm_rb**则使用红黑树。**mm_rb**指向根节点，每一个**vm_area_struc**结构体通过自身的**vm_rb**连接到树中。

在[进程管理与调度](./sched.md/#_9)中我们介绍了mm变量，用来存放该进程使用的内存描述符，**current->mm**就指向当前进程的内存描述符。`fork()`函数利用`copy_mm()`函数复制父进程的内存描述符。如果父子进程共享地址空间，则在调用`clone()`时，设置**CLONE_VM**标志，这样的进程就是线程。在Linux环境下，是否共享地址空间几乎是进程和线程本质上的唯一区别。如果指定了**CLONE_VM**,线程就不需要另外分配地址空间了，而是直接将mm域直接指向父进程的内存描述符即可。

内核线程没有进程地址空间，也没有相关的内存描述符，其mm域为NULL。当一个内核线程被调度时，它会直接使用前一个进程的内存描述符。

## 虚拟内存区域

虚拟内存区域（Virtual Memory Area, VMA）在内核中用**vm_area_struct**结构体描述。每个**vm_area_struct**结构都对应于指定地址空间上某块连续的内存区域。**vm_start**指向了这块虚拟内存区域的起始地址，**vm_end**指向了这块虚拟内存区域的结束地址。**vm_area_struct**结构体描述的是[vm_start，vm_end)这样一段左闭右开的虚拟内存区域。

![vm_area_struct](../../images/kernel/vma2.webp)

内核将每个内存区域作为一个单独的内存对象管理，每个区域又一致的属性和操作，下面给出**vm_area_struct**结构体的定义：

```C
struct vm_area_struct {
    struct mm_struct *vm_mm;                      //相关的mm_struct结构体
    unsigned long vm_start;                       //区间的首地址
    unsigned long vm_end;                         //区间的尾地址
    struct vm_area_struct *vm_next;               //VMA链表
    pgprot_t vm_page_prot;                        //访问控制权限
    unsigned long vm_flags;                       //标志位
    struct rb_node vm_rb;                         //红黑树节点
    struct anon_vma *anon_vma;                    //匿名VMA对象
    const struct vm_operations_struct *vm_ops;    //VMA的操作函数
    unsigned long vm_pgoff;                       //文件中的偏移量
    struct file *vm_file;                         //被映射的文件
    void *vm_private_data;                        //私有数据
};
```

每个内存描述符对应进程地址空间中的唯一空间，范围是[vm_start, vm_end)。**vm_mm**指向与VMA相关的**mm_struct**结构体，**vm_next**负责将VMA串联成链表。

![Alt text](../../images/kernel/vma3.webp)

### VMA标志

**vm_flags**定义了VMA的标志，它表示内存区域的行为和信息。一些比较重要的标志比如：VM_READ、VM_WRITE和VM_EXEC标志了内存区域中页面的可读、可写、可执行权限。当访问VMA时，需要查看其访问权限。

VM_SHARED表示内存区域包含的映射是否可以在多进程之间共享，如果设置了该标志位，我们称其位共享映射，否则就是私有映射。

VM_IO标志内存区域中对设备I/O空间的映射。该标志通常在设备驱动程序执行`mmap()`函数时才被设置。VM_RESERVED标志规定了内存区域不能被换出。

### VMA操作

**vm_ops**域指向与内存区域相关的操作函数，由**vm_operations_struct**结构体表示：

```C
struct vm_operations_struct {
    void (*open)(struct vm_area_struct *);
    void (*close)(sturct vm_area_struct *);
    int (*fault)(struct vm_area_struct *, struct vm_fault *);
    int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
    int (*access)(struct vm_area_struct *, unsigned long, void *, int, int);
};
```

## malloc原理

