# 内核线程

内核线程是直接由内核本身启动的进程，它们不属于任何用户空间进程，通常用于特定的目的，如处理中断、执行后台任务等。

内核线程可以大致划分为两类：

1. 阻塞型：线程启动后就一直等待，直到内核唤醒它
2. 周期型：线程启动后就按周期间隔运行

## init线程

init 线程是所有用户进程的父进程，它是系统启动的第一个进程，也是所有用户进程的祖先。在内核启动阶段，它作为内核线程，执行一系列初始化任务：

```C title="init/main.c"
static int __ref kernel_init(void *unused)
{
	int ret;

	/*
	 * Wait until kthreadd is all set-up.
	 */
	wait_for_completion(&kthreadd_done);

	kernel_init_freeable();	//
	/* need to finish all async __init code before freeing the memory */
	async_synchronize_full();
	kprobe_free_init_mem();
	ftrace_free_init_mem();
	kgdb_free_init_mem();
	exit_boot_config();
	free_initmem();
	mark_readonly();

	/*
	 * Kernel mappings are now finalized - update the userspace page-table
	 * to finalize PTI.
	 */
	pti_finalize();

	system_state = SYSTEM_RUNNING;
	numa_default_policy();

	rcu_end_inkernel_boot();

	do_sysctl_args();

	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}

	if (CONFIG_DEFAULT_INIT[0] != '\0') {
		ret = run_init_process(CONFIG_DEFAULT_INIT);
		if (ret)
			pr_err("Default init %s failed (error %d)\n",
			       CONFIG_DEFAULT_INIT, ret);
		else
			return 0;
	}

	//查找可以执行的init程序
	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/admin-guide/init.rst for guidance.");
}
```

### kernel_init_freeable函数

该函数是 init 线程中最重要的初始化函数，内核在这里完成大部分的初始化工作，并准备跳入用户空间：

```C
static noinline void __init kernel_init_freeable(void)
{
	/* Now the scheduler is fully set up and can do blocking allocations */
	gfp_allowed_mask = __GFP_BITS_MASK;

	/*
	 * init can allocate pages on any node
	 */
	set_mems_allowed(node_states[N_MEMORY]);

	cad_pid = get_pid(task_pid(current));

	smp_prepare_cpus(setup_max_cpus);

	workqueue_init();

	init_mm_internals();

	rcu_init_tasks_generic();
	do_pre_smp_initcalls();
	lockup_detector_init();

	smp_init();
	sched_init_smp();

	padata_init();
	page_alloc_init_late();
	/* Initialize page ext after all struct pages are initialized. */
	page_ext_init();

	do_basic_setup();

	kunit_run_all_tests();

	wait_for_initramfs();
	console_on_rootfs();

	/*
	 * check if there is an early userspace init.  If yes, let it do all
	 * the work
	 */
	if (init_eaccess(ramdisk_execute_command) != 0) {
		ramdisk_execute_command = NULL;
		prepare_namespace();
	}

	/*
	 * Ok, we have completed the initial bootup, and
	 * we're essentially up and running. Get rid of the
	 * initmem segments and start the user-mode stuff..
	 *
	 * rootfs is available now, try loading the public keys
	 * and default modules
	 */

	integrity_load_keys();
}
```

`console_on_rootfs()`函数用于打开 /dev/console 设备，并将其设置为标准输入、标准输出和标准错误：
```C
/* Open /dev/console, for stdin/stdout/stderr, this should never fail */
void __init console_on_rootfs(void)
{
	struct file *file = filp_open("/dev/console", O_RDWR, 0);

	if (IS_ERR(file)) {
		pr_err("Warning: unable to open an initial console.\n");
		return;
	}
	init_dup(file);
	init_dup(file);
	init_dup(file);
	fput(file);
}
```

`prepare_namespace()`函数用于创建用户空间的根目录和文件系统：

```C
void __init prepare_namespace(void)
{
	if (root_delay) {
		printk(KERN_INFO "Waiting %d sec before mounting root device...\n",
		       root_delay);
		ssleep(root_delay);
	}

	/*
	 * wait for the known devices to complete their probing
	 *
	 * Note: this is a potential source of long boot delays.
	 * For example, it is not atypical to wait 5 seconds here
	 * for the touchpad of a laptop to initialize.
	 */
	wait_for_device_probe();

	md_run_setup();

	if (saved_root_name[0]) {
		root_device_name = saved_root_name;
		if (!strncmp(root_device_name, "mtd", 3) ||
		    !strncmp(root_device_name, "ubi", 3)) {
			mount_block_root(root_device_name, root_mountflags);
			goto out;
		}
		ROOT_DEV = name_to_dev_t(root_device_name);
		if (strncmp(root_device_name, "/dev/", 5) == 0)
			root_device_name += 5;
	}

	if (initrd_load())
		goto out;

	/* wait for any asynchronous scanning to complete */
	if ((ROOT_DEV == 0) && root_wait) {
		printk(KERN_INFO "Waiting for root device %s...\n",
			saved_root_name);
		while (driver_probe_done() != 0 ||
			(ROOT_DEV = name_to_dev_t(saved_root_name)) == 0)
			msleep(5);
		async_synchronize_full();
	}

	mount_root();
out:
	devtmpfs_mount();
	init_mount(".", "/", NULL, MS_MOVE, NULL);
	init_chroot(".");
}
```

## kthreadd线程

我们知道Linux所有进程的祖先是0号 idle 进程，idle 进程创建了1号 init 进程。init 进程是所有用户进程的父进程。而内核线程同样也有自己的祖先那就是2号 kthreadd 线程。它的职责是创建和管理其他内核线程，它运行在一个循环中，不断遍历`kthread_create_list`链表，寻找新的内核线程创建请求。

这两个进程都是在初始化阶段被创建的：

```C
start_kernel()		// init/main.c
-->arch_call_rest_init()
	-->rest_init()
		-->pid = kernel_thread(kernel_init, NULL, CLONE_FS);
		-->pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

`kernel_thread()`函数类似于用户空间的`fork()`函数，它被用来创建内核线程：

```C
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	struct kernel_clone_args args = {
		.flags		= ((lower_32_bits(flags) | CLONE_VM |
				    CLONE_UNTRACED) & ~CSIGNAL),
		.exit_signal	= (lower_32_bits(flags) & CSIGNAL),
		.stack		= (unsigned long)fn,
		.stack_size	= (unsigned long)arg,
	};

	return kernel_clone(&args);
}
```

`kernel_clone()`其实最终也会调用`copy_process()`函数。根据前面所说，idle 进程是所有进程的父进程，那么内核线程的创建也是通过拷贝 idle 进程来实现的。idle 进程的部分内容如下：

```C
//init/init_task.c
struct task_struct init_task = {
	...
	.flags		= PF_KTHREAD,		// 标记为内核线程
	.mm = NULL,
	.active_mm = &init_mm,
};
```

可以看到，idle 进程的 .mm 为空。因此内核线程的 mm 域始终为空，相关代码如下：

```C
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
	struct mm_struct *mm, *oldmm;

	....

	tsk->mm = NULL;
	tsk->active_mm = NULL;

	/*
	 * Are we cloning a kernel thread?
	 *
	 * We need to steal a active VM for that..
	 */
	oldmm = current->mm;
	if (!oldmm)		//内核线程在这里直接返回
		return 0;

	/* initialize the new vmacache entries */
	vmacache_flush(tsk);

	if (clone_flags & CLONE_VM) {
		mmget(oldmm);
		mm = oldmm;
	} else {
		mm = dup_mm(tsk, current->mm);
		if (!mm)
			return -ENOMEM;
	}

	tsk->mm = mm;
	tsk->active_mm = mm;
	return 0;
}
```

当判断 oldmm 为空时，说明是内核线程，直接返回0。当 idle 进程创建完 kthreadd 线程后，它被调度执行`kthreadd()`函数。

### kthreadd函数

```C
int kthreadd(void *unused)
{
	struct task_struct *tsk = current;

	/* Setup a clean context for our children to inherit. */
	set_task_comm(tsk, "kthreadd");
	ignore_signals(tsk);
	set_cpus_allowed_ptr(tsk, housekeeping_cpumask(HK_FLAG_KTHREAD));
	set_mems_allowed(node_states[N_MEMORY]);

	current->flags |= PF_NOFREEZE;
	cgroup_init_kthreadd();

	for (;;) {
		set_current_state(TASK_INTERRUPTIBLE);
		if (list_empty(&kthread_create_list))
			schedule();
		__set_current_state(TASK_RUNNING);

		spin_lock(&kthread_create_lock);
		while (!list_empty(&kthread_create_list)) {
			struct kthread_create_info *create;

			create = list_entry(kthread_create_list.next,
					    struct kthread_create_info, list);
			list_del_init(&create->list);
			spin_unlock(&kthread_create_lock);

			create_kthread(create);

			spin_lock(&kthread_create_lock);
		}
		spin_unlock(&kthread_create_lock);
	}

	return 0;
}
```

该函数的核心其实就是围绕着`kthread_create_list`。这个链表存放其他内核路径创建的`struct kthread_create_info`结构体：

```C
struct kthread_create_info
{
	/* Information passed to kthread() from kthreadd. */
	int (*threadfn)(void *data);
	void *data;
	int node;

	/* Result passed back to kthread_create() from kthreadd. */
	struct task_struct *result;
	struct completion *done;

	struct list_head list;
};
```

kthreadd 线程先判断`kthread_create_list`是否为空，如果为空，则调用`schedule()`函数进入休眠状态。一旦有`kthread_create_info`结构体加入到链表中，则 kthreadd 线程被唤醒，并从`__set_current_state(TASK_RUNNING)`处开始执行。它进入一个循环，不断从`kthread_create_list`中取出`struct kthread_create_info`结构体，并调用`create_kthread()`函数创建内核线程。

`create_kthread()`函数很简单，就是调用`kernel_thread()`函数创建内核线程：

```C
create_kthread()
-->pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
```

## kthread函数

当 kthreadd 线程创建完内核线程之后就完成了它的使命，内核线程执行的其实是`kthread()`函数。该函数会初始化`struct kthread`结构体，设置完成变量，然后通知 kthreadd 线程它已经准备好。当开始执行时，就会调用用户定义的`threadfn()`函数来执行实际的工作：

```C
static int kthread(void *_create)
{
	/* Copy data: it's on kthread's stack */
	struct kthread_create_info *create = _create;
	int (*threadfn)(void *data) = create->threadfn;
	void *data = create->data;
	struct completion *done;
	struct kthread *self;
	int ret;

	set_kthread_struct(current);
	self = to_kthread(current);

	/* If user was SIGKILLed, I release the structure. */
	done = xchg(&create->done, NULL);		//获取完成变量done
	if (!done) {
		kfree(create);
		do_exit(-EINTR);
	}

	if (!self) {
		create->result = ERR_PTR(-ENOMEM);
		complete(done);
		do_exit(-ENOMEM);
	}

	self->threadfn = threadfn;
	self->data = data;						//设置线程执行函数和参数
	init_completion(&self->exited);
	init_completion(&self->parked);
	current->vfork_done = &self->exited;

	/* OK, tell user we're spawned, wait for stop or wakeup */
	__set_current_state(TASK_UNINTERRUPTIBLE);
	create->result = current;
	/*
	 * Thread is going to call schedule(), do not preempt it,
	 * or the creator may spend more time in wait_task_inactive().
	 */
	preempt_disable();
	complete(done);					//通知kthreadd线程完成
	schedule_preempt_disabled();	//睡眠
	preempt_enable();				//唤醒时从这里开始执行

	ret = -EINTR;
	if (!test_bit(KTHREAD_SHOULD_STOP, &self->flags)) {
		cgroup_kthread_ready();
		__kthread_parkme(self);
		ret = threadfn(data);		//这里才真正执行用户定义的函数
	}
	do_exit(ret);
}
```

### kthread_run函数

`kthread_run()`函数在创建完内核线程后就立即唤醒它：

```C
#define kthread_run(threadfn, data, namefmt, ...)			   \
({									   \
	struct task_struct *__k						   \
		= kthread_create(threadfn, data, namefmt, ## __VA_ARGS__); \
	if (!IS_ERR(__k))						   \
		wake_up_process(__k);					   \
	__k;								   \
})
```

### kthread_create函数

```C
kthread_create()
-->kthread_create_on_node()
	-->__kthread_create_on_node()
```

`__kthread_create_on_node()`函数的逻辑并不复杂，它首先填充`struct kthread_create_info`结构体，然后唤醒 kthreadd 线程来处理创建内核线程的请求，最后再进行一些后续的处理。需要注意的是，该函数创建的内核线程处于睡眠状态：

```C
struct task_struct *__kthread_create_on_node(int (*threadfn)(void *data),
						    void *data, int node,
						    const char namefmt[],
						    va_list args)
{
	DECLARE_COMPLETION_ONSTACK(done);
	struct task_struct *task;
	struct kthread_create_info *create = kmalloc(sizeof(*create),
						     GFP_KERNEL);

	if (!create)
		return ERR_PTR(-ENOMEM);
	create->threadfn = threadfn;
	create->data = data;
	create->node = node;
	create->done = &done;			//填充kthread_create_info结构体

	spin_lock(&kthread_create_lock);
	list_add_tail(&create->list, &kthread_create_list);		//加入链表
	spin_unlock(&kthread_create_lock);

	wake_up_process(kthreadd_task);		//唤醒kthreadd线程
	/*
	 * Wait for completion in killable state, for I might be chosen by
	 * the OOM killer while kthreadd is trying to allocate memory for
	 * new kernel thread.
	 */
	if (unlikely(wait_for_completion_killable(&done))) {
		/*
		 * If I was SIGKILLed before kthreadd (or new kernel thread)
		 * calls complete(), leave the cleanup of this structure to
		 * that thread.
		 */
		if (xchg(&create->done, NULL))
			return ERR_PTR(-EINTR);
		/*
		 * kthreadd (or new kernel thread) will call complete()
		 * shortly.
		 */
		wait_for_completion(&done);
	}
	task = create->result;			//获取创建的内核线程
	if (!IS_ERR(task)) {			//后续处理
		static const struct sched_param param = { .sched_priority = 0 };
		char name[TASK_COMM_LEN];

		/*
		 * task is already visible to other tasks, so updating
		 * COMM must be protected.
		 */
		vsnprintf(name, sizeof(name), namefmt, args);
		set_task_comm(task, name);
		/*
		 * root may have changed our (kthreadd's) priority or CPU mask.
		 * The kernel thread should not inherit these properties.
		 */
		sched_setscheduler_nocheck(task, SCHED_NORMAL, &param);
		set_cpus_allowed_ptr(task,
				     housekeeping_cpumask(HK_FLAG_KTHREAD));
	}
	kfree(create);
	return task;
}
```

### kthread_stop函数

一般，通过`kthread_create()`函数创建的内核线程可以通过`kthread_stop()`函数来停止：

```C
int kthread_stop(struct task_struct *k)
{
	struct kthread *kthread;
	int ret;

	trace_sched_kthread_stop(k);

	get_task_struct(k);
	kthread = to_kthread(k);
	set_bit(KTHREAD_SHOULD_STOP, &kthread->flags);
	kthread_unpark(k);
	wake_up_process(k);
	wait_for_completion(&kthread->exited);
	ret = k->exit_code;
	put_task_struct(k);

	trace_sched_kthread_stop_ret(ret);
	return ret;
}
```

## kthread_worker机制

`kthread_worker`和`kthread_work`是用来实现内核异步执行工作的机制。`kthread_worker`代表工作队列，它不执行具体工作，而是负责协调和调度工作项的执行。`kthread_work`代表一个可以被`kthread_worker`执行的工作单元，每个工作单元包含一个回调函数，当该工作被调度执行时，这个处理函数就会被调用。

我们来看下这两个结构体的组成：

```C
struct kthread_worker {
	unsigned int		flags;
	raw_spinlock_t		lock;
	struct list_head	work_list;
	struct list_head	delayed_work_list;
	struct task_struct	*task;
	struct kthread_work	*current_work;
};

struct kthread_work {
	struct list_head	node;
	kthread_work_func_t	func;
	struct kthread_worker	*worker;
	/* Number of canceling calls that are running at the moment. */
	int			canceling;
};
```

为了提供延迟执行机制，内核还设计了`kthread_delayed_work`结构体，它和`kthread_work`类似，但它可以指定一个延迟时间，在指定的时间后才会被执行：

```C
struct kthread_delayed_work {
	struct kthread_work	work;
	struct timer_list	timer;
};
```

由于原理与`kthread_work`类似，因此这里就不细讲了。

### 使用方法

1. 初始化`kthread_worker`和`kthread_work`：

   	```C
   	kthread_init_worker(&worker);
   	kthread_init_work(&work, work_function);
   	```

2. 创建并启动一个内核线程来服务这个工作队列：

	```C
	kthread_run(kthread_worker_fn, &worker, "woker_thread");
	```

3. 将工作项添加到工作队列：

	```C
	kthread_queue_work(&worker, &work);
	```

4. 当不再需要工作队列时，销毁它：

	```C
	kthread_flush_worker(&worker);
	kthread_destroy_worker(&worker);
	```

注意：

1. `kthread_run()`函数必须指定`kthread_worker_fn`作为内核线程函数。
2. 用户需要自己实现`kthread_init_work()`中的`work_function()`函数。

### kthread_worker_fn函数

`kthread_worker_fn()`函数是由内核实现的，它负责从工作队列中取出工作项，并执行它们。它首先判断是否需要停止，如果需要停止，则将`worker->task`设置为`NULL`，并返回。然后它尝试从`kthread_worker`的`work_list`中取出一个工作项，如果取到，则将`worker->current_work`设置为该工作项，并调用该工作项的回调函数。如果工作队列为空，则调用`schedule()`函数进入休眠状态。跳转到 repeat 处，重复执行上面的操作。

```C
int kthread_worker_fn(void *worker_ptr)
{
	struct kthread_worker *worker = worker_ptr;
	struct kthread_work *work;

	/*
	 * FIXME: Update the check and remove the assignment when all kthread
	 * worker users are created using kthread_create_worker*() functions.
	 */
	WARN_ON(worker->task && worker->task != current);
	worker->task = current;

	if (worker->flags & KTW_FREEZABLE)
		set_freezable();

repeat:
	set_current_state(TASK_INTERRUPTIBLE);	/* mb paired w/ kthread_stop */

	if (kthread_should_stop()) {
		__set_current_state(TASK_RUNNING);
		raw_spin_lock_irq(&worker->lock);
		worker->task = NULL;
		raw_spin_unlock_irq(&worker->lock);
		return 0;
	}

	work = NULL;
	raw_spin_lock_irq(&worker->lock);
	if (!list_empty(&worker->work_list)) {
		work = list_first_entry(&worker->work_list,
					struct kthread_work, node);
		list_del_init(&work->node);
	}
	worker->current_work = work;
	raw_spin_unlock_irq(&worker->lock);

	if (work) {
		kthread_work_func_t func = work->func;
		__set_current_state(TASK_RUNNING);
		trace_sched_kthread_work_execute_start(work);
		work->func(work);
		/*
		 * Avoid dereferencing work after this point.  The trace
		 * event only cares about the address.
		 */
		trace_sched_kthread_work_execute_end(work, func);
	} else if (!freezing(current))
		schedule();

	try_to_freeze();
	cond_resched();
	goto repeat;
}
```