# 锁的实现

## arch_spinlock_t

早期的 ARM 平台 spin lock 的实现使用了`raw_spinlock_t`：

```C
typedef struct {
    volatile unsigned int lock;
}raw_spinlock_t;
```

这个版本的锁实现简单—— 0表示 unlocked，1表示 locked。但存在严重的不公平性问题。也就是所有的线程无序地争抢 spin lock，不管线程等待了多久。在冲突比较少的情况下，不公平性体现得并不明显。但是随着硬件的发展，处理器的核心数越来越多，多核之间的冲突愈发剧烈，不公平性就愈发明显。有人测试过在极端情况下，有的线程甚至要等待 100,000 次机会才会执行一次。

现在的 ARM 平台 spin lock 使用以下定义：

```C
typedef struct {
    union {
        u32 slock;
        struct __raw_tickets {
            u16 owner;
            u16 next;
        } tickets;
    };
} arch_spinlock_t; 
```

该锁实际上就是所谓的{==票号自旋锁==}。它的原理类似于去银行办业务，每个进来的人先取一个号然后等待。业务员处理完一个人的业务之后就会叫号，如果叫的号和自己手里取的号是一样的，就轮到自己去办业务。

owner 代表当前正在办理业务的票号，next 代表下一个人取号的号码。在最开始的时候，owner 和 next 都等于0，第一个线程进来时，发现这两者相等，于是加锁，并且将 next++。后来的线程在 next 的基础上累加并等待 owner 释放。当第一个线程释放之后，将 owner++，此时 owner 又等于 next，所以第二个线程加锁成功。以此类推。而 owner 与 next 的差值就代表排队等待线程的个数。

![票号自旋锁](../images/arm/ticket_lock.webp)

申请锁操作：

```C
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
    arch_spinlock_t old_lock;
    old_lock.slock = lock->slock;
    lock->tickets.next++;
    while(old_lock.tickets.next != old_lock.tickets.owner) {
        wfe();
        old_lock.tickets.owner = lock->tickets.owner;
        
    }
}
```

汇编指令`wfe()`让ARM核进入低功耗模式。

释放锁操作：

```C
static inline void arch_spin_unlock(arch_spinlock_t *lock)
{
    lock->tickets.owner++;
    sev();
}
```

汇编指令`sev()`唤醒所有睡眠的CPU。

!!! tips

    对于`arch_spin_lock()`、`arch_spin_unlock()`的底层实现，不同内核版本一直在变化。

## arch_rwlock_t

```C
typedef struct {
    u32 lock;
}arch_rwlock_t;
```

lock 的最高位代表是否有写线程进入临界区，低31位统计读线程个数。

read_lock 操作：

```C
static inline void arch_read_lock(arch_rwlock_t *rw)
{
    unsigned int tmp;
    do {
        wfe();
        tmp = rw>lock;
        temp++;
    }while(tep & (1 << 31);

    rw->lock = tmp;
}
```

read_unlock 操作：

```C
static inline void arch_read_unlock(arch_rwlock_t *rw)
{
    rw->lock++;
    sev();
}
```

write_lock 操作：

```C
static inline void arch_write_lock(arch_rwlock_t *rw)
{
    unsigned int tmp;
    sevl();

    do {
        wfe();
        temp = rw->lock;
    }while(temp);
    rw->lock = 1 << 31;
}
```

只有当 rw->lock 的值为0，写线程才可以进入临界区，同时将最高位置1。

write_unlock 操作：
```C
static inline arch_write_unlock(arch_rwlock_t *rw)
{
    rw->lock = 0;
    sev();
}
```

## semaphore

```C
struct semaphore_waiter {
    struct list_head list;
    struct task_struct *task;
};

sturct sempahore {
    unsigned int count;
    struct list_head wait_list;
};
```

申请 semaphore：

```C
void down(struct semaphore *sem)
{
   	struct semaphore_waiter waiter;
     
   	if (sem->count > 0) {
   		sem->count--;                               
   		return;
   	}
     
   	waiter.task = current;                          
   	list_add_tail(&waiter.list, &sem->wait_list);   
   	schedule();                                     
}
```

释放 semaphore：

```C
void up(struct semaphore *sem)
{
   	struct semaphore_waiter waiter;
    
   	if (list_empty(&sem->wait_list)) {
   		sem->count++;                              
   		return;
   	}
     
   	waiter = list_first_entry(&sem->wait_list, struct semaphore_waiter, list);
   	list_del(&waiter->list);                       
   	wake_up_process(waiter->task);                 
}
```

## mutex

mutex 类似于计数值只有1的 semaphore，但是只有持有锁的人才能解锁，而信号量可以由任何一个线程释放。

```C
struct mutex_waiter {
    struct list_head list;
    struct task_struct *task;
};

struct mutex {
    long owner;
    struct list_head wait_list;
};
```

mutex 的 owner 字段记录持有锁的线程 ID，wait_list 记录等待锁的线程。

申请 mutex：

```C
void mutex_take(struct mutex *mutex)
{
    struct mutex_waiter waiter;
     
    if (!mutex->owner) {
    	mutex->owner = (long)current;               
    	return;
    }
     
    waiter.task = current;
    list_add_tail(&waiter.list, &mutex->wait_list); 
    schedule();                                     
} 
```

当 mutex->owner 为0时，表示没有线程持有锁。当不能获取 mutex 时，将线程挂入等待链表。

释放 mutex：

```C
int mutex_release(struct mutex *mutex)
{
    struct mutex_waiter *waiter;
     
    if (mutex->owner != (long)current)                         
    	return -1;
     
    if (list_empty(&mutex->wait_list)) {
    	mutex->owner = 0;                                      
    	return 0;
    }
     
    waiter = list_first_entry(&mutex->wait_list, struct mutex_waiter, list);
    list_del(&waiter->list);
    mutex->owner = (long)waiter->task;                         
    wake_up_process(waiter->task);                             
     
    return 0; 
}
```




