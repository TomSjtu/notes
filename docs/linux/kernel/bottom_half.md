# 中断下半部

## 软中断

!!! note

    软中断的英文名是softirq，与软件中断（software irq）没有任何关系。

软中断的本质就是在硬件中断执行完毕后，通过`wakeup_softirqd()`的方式唤醒一个softirq队列，然后中断程序返回，softirq队列在适当的时候开始执行。

普通的驱动程序一般不会用到软中断，只有那些对于性能要求非常高的比如网卡驱动会用到。由于tasklet是基于软中断实现的，所以了解软中断是理解tasklet的关键。

为了性能，同一类型的软中断有可能在不同的CPU上并发执行，这给使用者带来了极大的痛苦，因为驱动工程师在撰写软中断的回调函数的时候要考虑重入，考虑并发，要引入同步机制。

### 软中断的实现

软中断在编译期间由内核静态分配：

```C
/*内核支持的软中断类型*/
enum {
    HI_SOFTIRQ=0,       /* 高优先级tasklet */
	TIMER_SOFTIRQ,      /* Timer定时器软中断 */
	NET_TX_SOFTIRQ,     /* 发送网络数据包软中断 */
	NET_RX_SOFTIRQ,     /* 接收网络数据包软中断 */
	BLOCK_SOFTIRQ,      /* 块设备软中断 */
	IRQ_POLL_SOFTIRQ,   /* 块设备软中断 */
	TASKLET_SOFTIRQ,    /* 普通tasklet */
	SCHED_SOFTIRQ,      /* 进程调度及负载均衡的软中断 */
	HRTIMER_SOFTIRQ,    /*高精度timer定时器软中断 */
	RCU_SOFTIRQ,        /* Preferable RCU should always be the last softirq， RCU相关的软中断 */
	NR_SOFTIRQS
};

/*软中断描述符，只包含一个handler指针*/
struct softirq_action {
    void (*action)(struct softirq_action *);
};

/*软中断描述符表，一个全局数组*/
static struct softirq_action softirq_vec[NR_SOFTIRQS]__cacheline_aligned_in_smp;

/*CPU软中断状态描述，当某个软中断触发时，__softirq_pending会置位对应的bit*/
typedef struct {
    unsigned int __softirq_pending;
    unsigned int ipi_irqs[NR_IPI];
}__cacheline_aligned irq_cpustat_t;

/*每个CPU维护一个状态信息*/
irq_cpustat_t irq_stat[NR_CPUS];

/*每个CPU创建一个ksoftirq线程*/
DEFINE_PER_CPU(struct task_struct *, ksoftirqd);
```

![softirq](../../images/kernel/softirq.png)

软中断可以在不同的CPU上并发运行，但是在同一个CPU上只能串行运行。

### 触发软中断

调用`open_softirq()`函数注册softirq action回调函数：

```C
void open_softirq(int nr, void (*action)(struct softirq_action*))
{
    softirq_vec[nr].action = action;
}
```

`raise_softirq()`函数用来触发本地CPU上的软中断：

```C
void raise_softirq(unsigned int nr)
{
    unsigned long flagss;

    local_irq_save(flags);
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}
```

虽然大部分场景是在中断处理程序中(本地CPU中断已关闭)执行软中断的触发动作，但是其他上下文也可以调用`raise_softirq()`函数。因此触发软中断的函数有两个接口，我们来看没有中断保护的`raise_softirq_irqoff()`函数：

```C
inline void raise_softirq_irqoff(unsigned int nr)
{
	__raise_softirq_irqoff(nr);

	if (!in_interrupt() && should_wake_ksoftirqd())
		wakeup_softirqd();
}
```

- `__raise_softirq_irqoff()`函数会设置`__softirq_pending`的对应bit位，表示软中断已经挂起。

- 如果不是在中断上下文，我们必须要调用`wakeup_softirqd()`函数来唤醒ksoftirqd线程。

软中断执行的时候，允许响应中断，本地软中断被禁止，但在其他CPU上可以执行相同的软中断。因此大部分软中断处理，都通过采取单处理器数据来避免加锁，从而提供更出色的性能。

### 执行软中断

前面我们说到，在调用`raise_softirq()`函数触发软中断之后，内核就会选择合适的时机来执行软中断。那么具体延迟到什么时候呢？

软中断在内核中执行的入口函数是`invoke_softirq()`，它的实现如下：

```C
static inline void invoke_softirq(void)
{
	if (ksoftirqd_running(local_softirq_pending()))
		return;

	if (!force_irqthreads() || !__this_cpu_read(ksoftirqd)) {
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
		__do_softirq();
#else
		do_softirq_own_stack();
#endif
	} else {
		wakeup_softirqd();
	}
}
```

`invoke_softirq()`函数首先会检查当前CPU是否正在执行软中断，如果正在执行软中断，则直接返回。

`force_irqthread()`是与强制线程化相关，也就是说如果对软中断线程化处理，则直接调用`wakeup_softirqd()`函数来唤醒ksoftirqd线程。

核心的处理逻辑在`__do_softirq()`函数中，它会遍历softirq_vec中的每一个bit，处理pending的软中断。

软中断的触发场景有：

1. 在中断返回用户空间时，如果有pending的软中断，执行对应的处理函数。
2. 在中断返回内核态，如果`local_bh_enable()`，则执行pending的软中断。
3. 系统过于繁忙，不断产生软中断，则推迟到kosftirq内核线程中执行。

## tasklet

tasklet是利用软中断实现的一种下半部机制，但是它的接口更简单，锁保护要求较低。大多数情况都可以使用tasklet来完成你需要的工作。

tasklet有一些比较有意思的特性：

- 一个tasklet可在稍后被禁止或者重新启用；只有启用的次数和禁止的次数相同时，tasklet才会被执行。

- tasklet可以注册自己本身。

- tasklet可被调度在正常优先级或者更高优先级执行。

- 当系统负载低时，tasklet会被立刻执行，但再晚不会晚于下一个定时器tick。

- 一个tasklet可以与其他tasklet并发，但是同一个tasklet永远不会在多个CPU上同时运行。

### tasklet的实现

tasklet由两类软中断代表：HI_SOFTIRQ和TASKLET_SOFTIRQ。前者优先级比后者高。

tasklet结构体如下：

```C
struct tasklet_struct {
    struct tasklet_struct *next;    //链表中下一个tasklet
    unsigned long state;            //tasklet的状态
    atomic_t count;                 //引用计数器
    void (*func)(unsigned long);    //处理函数
    unsigned long data;             //传递给函数的参数
};
```

state成员只能在0、TASKLET_STATE_SCHED和TASKLET_STATE_RUN之间取值。TASKLET_STATE_SCHED表明tasklet已被调度，正准备投入运行，TASKLET_STATE_RUN表明该tasklet正在运行。TASKLET_STATE_RUN只有在多处理器的系统上才会作为一种优化来使用，单处理器系统任何时候都清楚单个tasklet是不是正在运行(它要么就是当前正在执行的代码，要么不是)。

count成员是tasklet的引用计数器。如果它不为0，则tasklet被禁止；只有当它为0时，tasklet才被激活。

已调度（或者叫已激活）的tasklet存放在tasklet_vec（普通tasklet）和tasklet_hi_vec（高优先级的tasklet）数组中。这两个数据结构都是由`tasklet_struct`结构体构成的链表，链表中每个元素代表一个不同的tasklet。

tasklet由`tasklet_schedule()`和`tasklet_hi_schedule()`函数进行调度，它们接受一个指向`tasklet_struct`结构的指针作为参数，后者将指定的tasklet以高优先级运行。

### 使用tasklet

```C
/*静态分配tasklet*/
DECLARE_TASKLET(name, func, data)

/*动态分配tasklet*/
void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data)

/*禁用tasklet*/
void taskelt_disable(struct tasklet_struct *t)

/*使能tasklet*/
void tasklet_enable(struct tasklet_struct *t)

/*调度tasklet*/
void tasklet_schedule(struct tasklet_struct *t)

/*杀死tasklet*/
void tasklet_kill(struct tasklet_struct *t)
```

!!! example "tasklet的使用"

    ```C
    void do_tasklet(unsigned long);
    DECLARE_TASKLET(my_tasklet, do_tasklet, 0);

    void do_tasklet(unsigned long)
    {
        ...
    }

    irqreturn_t do_interrupt(int irq, void *dev_id)
    {
        ...
        tasklet_schedule(&do_tasklet);
    }

    int __init my_init(void)
    {
        /*申请中断*/
        result = requset_irq(irq, do_interrupt, 0, "my_irq", NULL);
        return IRQ_HANDLED;
    }

    void __exit my_exit(void)
    {
        free_irq(irq, do_interrupt);
    }

    ```

### ksoftirqd

每个处理器都有一组辅助处理软中断（和tasklet）的内核线程。当内核中出现大量软中断的时候，内核线程就会选择合适的时机来处理软中断。

在大流量的网络通信中，软中断的触发频率可能很高，甚至还会自行重复触发，这会导致用户空间的进程无法获得足够的处理器时间。如果软中断和重复触发的软中断都被立即处理，那么当负载很高的时候，系统就会出现明显的卡顿现象。如果选择不处理重新触发的软中断，又会浪费闲置的系统资源，导致软中断出现饥饿现象。

内核中的方案时不会立即处理重复触发的软中断。当大量软中断出现的时候，内核会唤醒一组内核线程来处理这些负载。这些线程在最低优先级（nice=19）运行，避免与其他任务抢占资源。

每个处理器都有一个这样的线程，名字为{==ksoftirqd/n==}，n为处理器编号。只要有待处理的软中断，ksoftirqd就会调用`do_softirq()`函数来处理它们。

## 工作队列

工作队列是一种延后执行的机制，可以将后续的工作交给一个内核线程执行——这个下半部分总是在进程上下文中执行。这样，通过工作队列实现的代码就能享受进程上下文的所有优势，比如可以重新调度甚至是睡眠。

如果推后执行的任务需要睡眠，那么就选择工作队列。否则就选择软中断或tasklet。工作队列在你需要获得大量内存时，需要获取信号量时，需要执行阻塞式的I/O操作时，它都会非常有用。

### 工作队列的实现

工作队列子系统提供了缺省的工作者线程（worker thread）来处理需要推后的工作。缺省的工作者线程叫做{==events/n==}，n为处理器的编号。你可以创建自己的工作者线程，不过一般使用缺省线程即可。专用的工作者线程可以更好地控制和优化任务的执行，但也需要更多的资源和管理开销。

工作队列用`workqueue_struct`结构体表示：

```C
struct workqueue_struct {
    struct cpu_workqueue_struct cpu_wq[NR_CPUS]；
    struct list_head list;
    const char *name;
    int singlethread;
    int freezeable;
    int rt;
};
```

该结构体内有一个`cpu_workqueue_struct`结构组成的数组，数组的每一项对应系统中的一个处理器。也就是说系统中每个处理器对应一个工作者线程。

工作由`work_struct`结构体表示：

```C
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
};
```

这些结构体被连接成链表，每个处理器上的每种类型的队列都对应这样一个链表。当一个工作者线程被唤醒时，它会执行链表上的所有工作，当没有剩余的操作时，它就会继续休眠。

### 使用工作队列

静态创建一个`work_struct`结构体：

```C
DECLARE_WORK(name, void(*func)(void *), void *data);
```

动态创建：

```C
INIT_WORK(struct work_struct *work, void (*func)(void *), void *data);
```

处理工作的回调函数：

```C
void work_handler(struct work_struct *work);
```

使用工作队列时要注意，我们既可以使用内核线程工作队列，也可以使用自定义的工作队列。

如果要自定义一个工作队列，可以使用宏：
```C
create_workqueue(name);
create_singlethread_workqueue(name);
```

这两个宏的区别是，第一个宏在每个处理器上为该工作队列创建专用的线程。第二个宏只创建一个工作者线程。

注意：不管使用哪个宏，在创建自定义工作队列后，必须在退出时，调用以下函数确保资源的释放：

```C
void flush_workqueue(struct work_struct *work);
void destroy_workqueue(struct workqueue_struct *wq);
```

提交工作给自定义工作队列：

```C
int queue_work(struct workqueue_struct *queue, struct work_struct *work);
int queue_delayed_work(struct workqueue_struct *queue, struct work_struct *work, unsigned long delay);
```

在多核系统中，每个CPU上都有一个工作队列，这两个函数不会指定提交至哪个CPU，但会优先选择本地CPU。

在许多情况下，驱动程序不需要单独的工作队列。如果我们只是偶尔需要向队列中提交任务，则可以使用内核默认的共享工作队列：

```C
bool schedule_work(struct work_struct *work);
bool schedule_delayed_work(struct work_struct *work, unsigned long delay);
```

## 下半部的同步

在上半部中，不应该调用disable/enable下半部来保护共享数据，因为下半部不能抢占上半部。`local_bh_disable()`和`local_bh_enable()`是给进程上下文使用的，用于防止下半部抢占进程上下文。

还有一种更强劲的同步——{==关本地中断==}，它实际上是同时禁止了上半部和下半部的抢占。

需要注意的是，软中断在同一个CPU上只可能串行执行，但是有可能在不同CPU上并发执行。而两个相同类型的tasklet不允许同时执行，即便是不同的处理器也不行。tasklet之间的同步，只需要正确使用锁机制即可。

于是同步场景可以分为：

- 进程上下文和下半部共享数据：禁止下半部
- 中断上下文和下半部共享数据：禁止中断并获取锁
- 工作队列中的共享数据：获取锁


