# 虚拟文件系统

虚拟文件系统（VFS）是Linux内核的一个重要子系统，它为用户空间程序提供了一套统一的接口，以实现对不同文件系统的访问和操作。通过这些接口，用户空间程序可以采用标准的系统调用来执行文件操作，而无需关心这些操作实际是在哪种文件系统上执行的。VFS使得内核能够支持多种文件系统类型，如ext4、ReiserFS、XFS等。对于用户空间程序来说，这些文件系统看起来都是一样的，因为VFS隐藏了底层文件系统的差异。VFS的设计使得Linux内核能够灵活地支持多种文件系统，同时保持用户空间程序的稳定性和可移植性。这对于Linux系统的可扩展性和灵活性至关重要。

为了支持多文件系统，VFS提供了一个通用文件系统模型的抽象层，该模型囊括了任何文件系统常用的功能和行为。用户空间程序只需要使用VFS暴露的接口，而无须关心底层文件系统的实现细节。比如调用一个简单的`write()`函数，首先由VFS转换为`sys_write()`，再根据fd文件描述符所在的文件系统，找到具体的实现方法，最后再执行该操作。

Linux系统总共采用了四种核心概念来抽象文件系统：文件、目录项、索引点和挂载点（mount point）。这些概念共同构建了一个分层的数据存储结构，用于管理文件和目录及相关的控制信息。Linux文件系统的一个重要特点是统一的命名空间，所有的文件被挂载到一个全局的层次结构中，形成一个类似根文件系统的结构。这与Windows系统将文件系统划分不同的分区不同。

Linux系统区分文件本身和文件相关的信息，比如访问控制权限、大小、所有者、创建时间等。这些相关信息，有时被称作文件的元数据，被存储在一个单独的数据结构中——索引节点（inode）。

## VFS对象

VFS使用一组数据结构来代表通用文件对象，它们分别是：

- 超级块对象，代表一个具体的已安装的文件系统。
- 索引节点对象，代表一个具体文件。
- 目录项对象，代表一个目录项，是一个路径的组成部分。
- 文件对象，代表由进程打开的文件。

每个通用对象都包含一个操作对象，描述了内核对通用对象的方法——super_operations、inode_operations、dentry_operations、file_operations。

超级块对象是用来描述一个已安装的文件系统的数据结构，包含了文件系统的关键元数据，这些元数据用于内核管理和维护文件系统。超级块对象是文件系统在内核中表示自己的方式，它提供了文件系统的总体视图。


## 进程中的文件数据结构

在[进程管理](./sched.md/)章节中，我们提到了在进程描述符`tast_struct`中有两个与文件系统相关的成员变量：

```C
struct fs_struct *fs;
struct files_struct *files;
```

`fs_struct`包含文件系统和进程相关的信息，定义在文件<linux/fs_struct.h\>中：

```C
struct fs_struct {
	int users;
	int umask;
	int in_exec;
	struct path root, pwd;
};
```

> user：当前打开该文件的用户数量。

> umask：用户模式掩码，决定了系统在创建文件和目录时的默认权限掩码。

> in_exec：表示文件系统是否处于执行模式。

> root，pwd：根目录路径和当前工作目录路径。

`files_struct`标识进程打开的文件信息，定义在文件<linux/fdtable.h\>中：

```C
struct files_struct {
	atomic_t count;
	
	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
  
	unsigned int next_fd;
	unsigned long close_on_exec_init[1];
	unsigned long open_fds_init[1];
	unsigned long full_fds_bits_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};
```

> count：跟踪当前打开的文件描述符的数量。

> fd：缓存下一个可用的fd。

> fd_array：指向一个`struct file`结构体的数组，用于存储打开的文件描述符。在64位体系中，一个进程默认可以打开64个文件对象。

还有一个非常重要的结构体是`namespace`，定义在文件<linux/mount.h\>中。它可以使每一个进程在系统中都看到唯一的文件系统层次结构，是Linux三大隔离技术之一。具体的细节就不在这里探讨了，在[容器技术](../../cloud/docker.md)一章我会再做详细介绍。

上述这些数据结构都是通过进程描述符链表连接起来的。对于多数进程，它们的描述符都指向唯一的`files_struct`和`fs_struct`结构体。但是对于那些在调用`clone()`函数时指定了克隆标志CLONE_FILES或者CLONE_FS的进程，会共享这两个结构体。

`namespace`结构体的使用方法比较特殊。默认情况下，所有进程都是共享一个命名空间的，只有当`clone()`函数时指定了CLONE_NEWS标志，才会给进程一个唯一的命名空间结构体的拷贝。