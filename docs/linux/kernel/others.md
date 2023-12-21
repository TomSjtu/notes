# 其他

> 这里主要放一些零碎的内容。

## 系统调用

> 在阅读本节内容之前，建议先了解一下[中断](./interrupt.md)。

当你的程序调用open, read, write, close等函数时，就会触发系统调用。系统调用本质是用户态进程与硬件设备交互的接口，内核负责检查系统调用的正确性，然后发出指令给硬件。作为应用程序开发者，不用关心底层硬件的实现细节，而只需要调用普通函数就可以使用系统调用了。glibc库进一步封装了细节，因此我们只需要使用glibc库暴露的API即可。比如glibc中的open函数，它是这样定义的：

```C
int open(const char *pathname, int flags, mode_t mode);
```

在glibc的源代码中，有个syscall.list，里面存放着所有glibc函数对应的系统调用。

### 处理流程

x86体系进入和退出系统调用有两种方式：

- int $0x80 和 iret
- sysenter 和 sysexit

第二种被称为快速系统调用。

无论哪种方式，最终结果都是跳转到系统调用处理函数（system call handler）。由于内核实现了很多不同的系统调用，因此进程必须传递一个名为系统调用号（system call number）的参数来识别所需的系统调用，这个参数存放在eax寄存器中。执行完系统调用后的返回值也放在eax寄存器中，其中正数或0表示系统调用成功，负数表示出错，存放于errno全局变量中。

系统调用处理流程是：

- 将系统调用的参数写入CPU寄存器
- 检查所有的系统调用参数
- 将CPU中的参数拷贝至内核态堆栈
- 调用名为系统调用服务例程（system call service routine）的C函数来处理系统调用
- 退出系统调用处理程序，将内核栈中的值加载至寄存器，并从内核态切换回用户态

为了将系统调用号与对应的服务例程联系起来，内核定义了一个系统调用分派表（system call dispatch table），这个表存放在sys_call_table数组中。内核拿到系统调用号之后，就去sys_call_table中找到对应的系统调用实现函数去执行。执行完毕后，使用返回指令从内核态返回至用户态。

## 信号

在Linux系统中，为了响应各种事件定义了非常多的信号。比如当我们发送kill -9 ${pid}时，其实就是发送SIGKILL信号给指定进程，将它杀死。我们可以通过kill -l命令查看所有的信号。每个信号都有一个唯一的ID和对应的默认操作。

进程对信号的处理方式有三种：

1. 执行默认操作。
2. 自定义信号处理函数。
3. 忽略信号。注意，SIGKILL和SIGSTOP无法忽略。

Linux推荐使用sigaction函数来自定义信号处理函数。它的定义如下：

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

其中sa_handler就是你要定义的信号处理函数。

发送信号由两种方式，一种是发送给整个线程组的，还有一种是发送给某个单独线程的。信号分为不可靠信号和可靠信号。在task_struct中有一个结构体sigpending，它的定义如下：

```C
struct sigpending{
    struct list_head list;
    sigset_t signal;
};
```

对于不可靠信号，也就是编号小于32的信号，会放在sigset_t集合中，不论发送多少次，在被处理前都只会保留一份。对于可靠信号，则会挂在struct sigpending的链表中挨个处理。



