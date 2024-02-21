# 进程间通信

## 管道


## 消息队列

## 共享内存


## 信号

在Linux系统中，为了响应各种事件定义了非常多的信号。比如当我们发送`kill -9 ${pid}`时，其实就是发送SIGKILL信号给指定进程，将它杀死。我们可以通过`kill -l`命令查看所有的信号。每个信号都有一个唯一的ID和对应的默认操作。

进程对信号的处理方式有三种：

1. 执行默认操作。
2. 自定义信号处理函数。
3. 忽略信号。注意，SIGKILL和SIGSTOP无法忽略。

Linux推荐使用`sigaction()`函数来自定义信号处理函数。它的定义如下：

```C
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

而sigaction结构体的定义如下：

```C
struct sigaction {
  __sighandler_t sa_handler;
  unsigned long sa_flags;
  __sigrestore_t sa_restorer;
  sigset_t sa_mask;   
};
```

其中`sa_handler`就是你要定义的信号处理函数。

发送信号由两种方式，一种是发送给整个线程组的，还有一种是发送给某个单独线程的。信号分为不可靠信号和可靠信号。在`task_struct`中有一个结构体`sigpending`，它的定义如下：

```C
struct sigpending{
    struct list_head list;
    sigset_t signal;
};
```

对于不可靠信号，也就是编号小于32的信号，会放在`sigset_t`集合中，不论发送多少次，在被处理前都只会保留一份。对于可靠信号，则会挂在`struct sigpending`的链表中挨个处理。

