# 定时器

定时器可以用来在将来的某个特定时间点执行特定的函数。被调度运行的函数会异步地运行，就像硬件中断发生时的场景一样。定时器函数必须以原子的方式运行。在SMP系统中，定时器函数会由注册它的同一CPU执行，以尽可能获得缓存的局部性。任何通过定时器函数访问的共享资源都应该针对并发访问进行保护。

系统级别的定时器能以固定的频率产生中断，这种中断被称为{==定时器中断==}，它所对应的中断处理程序负责更新系统时间和执行周期性任务。

系统定时器中断处理程序需要执行的操作有：

- 更新系统启动以来所经过的时间
- 更新时间和日期
- 确定当前进程在每个CPU上已运行了多少时间
- 运行超时的动态定时器
- 更新资源使用统计

## 节拍率和jiffies变量

系统定时器的频率被称为{==节拍率==}（tick rate），产生的两次时钟中断的间隔就被称为{==节拍==}（tick），它等于节拍率分之一秒。内核通过时钟中断间隔来计算系统运行时间。

定时器频率是通过静态预处理定义的，在系统启动时被设置，该值与体系结构相关。如果设置为1000，那么每秒就产生1000次中断（每1ms产生一次）。

定时器频率应设置为一个理想的值，高HZ可以提高系统的性能，使得由时间驱动的事件更为精确。但同时也带来了额外的系统负担，因为处理器会被更频繁地打断去执行时钟中断处理程序。

全局变量`jiffies`用来记录系统启动以来产生的节拍总数。内核在启动时将该变量初始化为0，此后每产生一次时钟中断该值就+1。因为一秒内时钟中断的次数等于HZ，所以`jiffies`一秒内增加的值也就为HZ。系统运行时间以秒为计，其值等于jiffies/HZ。

`jiffies`变量被定义为unsigned long类型。在32位体系结构上是32位。如果时钟频率为100HZ，那么497天后会溢出。如果是64位体系结构，可以放心地认为它不会溢出。

当`jiffies`变量的值溢出后再继续增加的话，它会回绕至0。内核使用`time_after`、`time_after_eq`、`time_before`和`time_before_eq`宏来判断两个时间值的关系。

为了解决`jiffies`变量的溢出问题，内核引入了`jiffies_64`变量，这是一个64位的无符号整数。在64位架构上，这两个变量是一个值；但在32位架构上，对64位值的访问不是原子的。这意味着读取64位值的高32位和低32位时，变量的值可能会发生更新而导致获取错误的值。因此在32位架构上不能对64位的值直接读取，必须使用一个特殊的辅助函数`get_jiffies_64`。

## 动态定时器的使用

`struct timer_list`结构体用来表示一个动态定时器：
```C
struct timer_list {
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;
};
```

定时器的使用很简单。你只需要执行一些初始化工作，设置一个超时时间，指定超时发生后需要执行的函数，然后激活定时器就可以了。内核提供了一组与定时器相关的接口用来简化定时器的操作。

创建定时器：

```C
struct timer_list my_timer;
```

初始化定时器：

```C
DEFINE_TIMER(name, func)；
void setup_timer(struct timer_list *timer, void (*function)(unsigned long), unsigned long data);
```

定时器到期激活函数原型：
```C
void my_function(unsigned long data);
```

一般来说，定时器都会在超时后马上执行，但也有可能会推迟到下一个时钟节拍时才运行，所以不能用定时器来实现任何硬实时的任务。如果需要修改定时器超时时间，可以通过`mod_timer()`函数来实现：

```C
int mod_timer(struct timer_list *timer, unsigned long expires);
```

注册定时器：

```C
void add_timer(struct timer_list *timer);
```

如果在定时器超时前停止定时器，可以使用`del_timer()`函数：

```C
int del_timer(struct timer_list *timer);
```

需要注意的是，在SMP系统中，删除定时器时可能需要等待在其他处理器上运行的定时器处理程序都退出，这时需要用到`del_timer_syn()`函数来执行删除工作。在大多数情况下，都应该调用这个函数而不是`del_timer()`。在持有锁时，需要格外小心`del_timer_syn()`函数的使用，因为如果定时器函数试图获取相同的锁，系统就会进入死锁。

## 延迟执行

内核代码往往需要推迟某些任务的执行，这种推迟通常发生在等待硬件完成某些工作，而且等待时间往往非常短。内核提供了多种延迟方法来处理延迟请求。

### 忙等待

最简单的延迟方法就是忙等待。该方法仅仅在想要延迟的时间是节拍的整数倍，或者精确率要求不高的情况下才可以使用。忙等待的实现非常简单——在一个循环中不断等待直到希望的时钟节拍数耗尽，比如：

```C
unsigned long timeout = jiffies + 10;    //等待10个节拍
while(time_before(jiffies, timeout)){
    //relax
}

```

该循环将不断执行，直到`jiffies`大于delay为止。这是一种低效的办法， 因为处理器除了等待不会做任何事情。

### 让出处理器

忙等待大大浪费了系统的性能，一种解决办法是在不需要CPU的情况下主动释放：

```C
while(time_before(jiffies, timeout)){
	schedule();
}
```

### 超时执行

如果驱动程序使用等待队列来等待其他一些事件，而我们同时希望在特定时间段中运行，可以使用`wait_event_timeout()`或者`wait_event_interruptible_timeout()`函数。调用上述函数会使进程进入等待状态，直到某个条件为真：

```C
long wait_event_timeout(wait_queue_head_t q, int condition, long timeout);
long wait_event_interruptible_timeout(wait_queue_head_t q, int condition, long timeout);
```

这两个函数的区别是，第一个函数不可中断，而第二个函数可以响应信号。两个函数都会在给定的等待队列上休眠，在超时到期时返回。如果有其他进程在等待队列上调用了`wake_up()`函数，也会使得进程从等待队列中被唤醒。

对于无需等待特定事件而延迟的情况，可以使用`schedule_timeout()`函数：

```C
signed long schedule_timeout(signed long timeout);
```

timeout是用`jiffies`表示的延迟时间。`schedule_timeout()`函数要求预先设置进程状态：`set_current_state(TASK_INTERRUPTIBLE)`。

注意，由于`schedule_timeout()`函数需要调用调度程序，所以调用它的代码必须保证能够睡眠。也就是调用函数必须位于进程上下文中，且不能持有锁。

当任务被重新调度时，将返回代码进入睡眠前的位置继续执行。如果任务提前被唤醒，那么定时器被撤销。

### 短延迟

> https://www.kernel.org/doc/html/v6.6/timers/timers-howto.html

使用短延迟时需要注意自己的代码是否位于原子上下文中。

原子上下文：

必须使用*delay函数族，这些函数忙等待足够的循环周期以实现所需的延迟：

- `ndelay(unsigned long nsecs)`
- `udelay(unsigned long usecs)`
- `mdelay(unsigned long msecs)`

首选udelay，非PC设备可能无法满足ns的精度。mdelay也不建议使用。

非原子上下文：

应该使用*sleep函数族，这些函数底层机制都不同：

- 由忙等待支持：`udelay(unsigned long usecs)`
- 由高精度定时器支持：`usleep_range(unsigned long min, unsigned long max)`
- 由jiffies支持：`msleep(unsigned long msecs)`

休眠"几"微秒(< 10us)：使用udelay

休眠~微秒或小毫秒(10us ~ 20ms)：使用usleep_range

休眠更长毫秒(10ms+)：使用msleep

灵活的休眠(任意延迟，不可中断)：使用fsleep

