# 实时补丁

Linux 5.3 引入了 PREEMPT_RT 补丁，意味着开发了十几年的实时补丁将与主线 Linux 合并。

所谓实时，就是指一个特定的任务的执行时间是确定的，并且在任何情况下都能保证任务的时限（最大执行时间限制）。根据对实时性的严格程度又分为软实时和硬实时：软实时允许任务延迟一段时间执行，而硬实时则要求任务必须在规定的时间内完成。

## 实时性指标

判断一个系统是否是实时，主要有以下两个指标：

- 中断延迟：中断延迟指外部中断事件发生到中断处理函数的第一条指令开始执行，这中间的差值。
- 抢占延迟：即一个运行中的任务被更高优先级的任务抢占后，系统完成任务切换所需要的时间。抢占延迟包括检测延迟、决策延迟、调度延迟、切换延迟。

除了软件方面的延迟，硬件也会对系统实时性产生影响。比如现代 CPU 往往采用多级 cache 技术，指令或数据在 cache 中和不在 cache 中的执行时间差距是非常大的。又比如虚拟内存管理机制，缺页异常机制也会严重影响任务执行时间的可预测性和确定性。

## 标准Linux内核的制约

标准 Linux 内核的设计并没有考虑实时性，实时性主要受到下面几个机制的影响：

1. 内核不可抢占：

    在比较旧的内核版本中，内核是不可抢占的，也就是说如果当前任务运行在内核，即使有更紧迫的任务需要执行，也必须得等内核路径执行完毕之后，才能运行新的任务。在 2.6 内核中，内核已经支持抢占，实时性得到了加强。但是内核路径中仍然有大量不可抢占的操作，比如自旋锁保护的临界区，以及一些显示调用`preempt_disable()`函数的区域。

2. 关闭中断：

    Linux 在一些同步操作中使用了中断关闭指令，中断关闭将增大中断延迟，降低系统的实时性。

3. 自旋锁

    在有共享资源保护的临界区，使用自旋锁使得其他任务只能进入等待状态，直到当前任务释放锁，这无疑会降低系统的实时性。

4. 中断处理程序

    在 Linux 系统中，中断是最高优先级的，在任何时刻发生中断内核都会立即响执行响应的中断处理函数以及软中断，直到处理完毕才执行正常的任务。假如一个系统处于繁重的网络负载和 I/O 负载，那么系统可能一直处在中断处理状态而没有任何机会执行实时任务。

5. 调度算法

    Linux 内核并不是在任何场景下都可以调度，比如在中断上下文，即使唤醒了一个高优先级的进程，也不能立即运行。因为中断上下文不能发生调度，该进程必须得等到中断处理完毕之后才能运行。如果中断下半部使用了自旋锁保护临界区，那么该进程还要等待临界区释放后才能运行。


## PREEMPT_RT 补丁

PREEMPT_RT 补丁的目标是为了解决标准 Linux 内核的实时性问题，它通过以下几个方面来提升系统的实时性：

1. 锁分解：即将一把大锁改为小锁，将临界区分解为多个小锁，这样可以减少锁的粒度，提升系统的并发性。

2. 增加抢占点：在代码路径中增加抢占点，降低系统的抢占延迟。

3. 中断线程化：将中断处理程序放到一个单独的线程中，这样实时任务就可以有比中断更高的优先级。

4. 将自旋锁改为互斥锁：自旋锁循环等待的机制白白浪费了 CPU 资源，如果改为互斥锁，进程在获取不到共享资源的情况会放弃 CPU，调度其他进程来运行。
