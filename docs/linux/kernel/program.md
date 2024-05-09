# 内核编程

内核开发的特点如下：

- 不能使用标准C库头文件，只能使用内核提供的头文件。与体系结构无关的头文件位于内核源码根目录下的include目录。与体系结构相关的头文件位于<arch/[architecture]/include/asm\>目录，内核代码以asm前缀的形式包含这些头文件。

- 必须使用GNU C。gcc编译器支持以asm()指令开头嵌入汇编代码。

- 没有内存保护机制。如果是内核访问了非法内存，后果不堪设想。

- 无法执行浮点运算。内核对于浮点运算的支持度不够。

- 必须考虑同步与并发。Linux内核是抢占式多任务处理系统，当前任务随时有可能被另一个任务抢占。

- 必须考虑可移植性。比如保持字节序、64 位对齐、不假定字长和页面长度等一系列准则。

!!! tip

    <Documentation/CodingStyle\>描述了Linux内核对编码风格的要求，<scripts/checkpatch.pl\>提供了检查代码风格的脚本。

## 可移植性

页的大小为PAGE_SIZE，不一定是4KB。

字节序，注意平台是大端还是小端。

数据对齐，内核代码中必须保证所有数据结构都按照其定义的大小对齐。

## 内核数据类型

内核的数据类型主要分为三大类：int——标准C语言类型，u32——确定大小类型，pid_t——特定内核对象类型。

内核中最常用的数据类型由它们自己的typedef声明，这样可以防止出现任何移植性问题。比如pid_t就屏蔽了可能的差异。

## 内核数据结构

内核数据结构贯穿于整个内核代码中，这里介绍4个基本的内核数据结构。

- 链表
- 队列
- 映射
- 红黑树

### 链表

链表是linux内核中最简单，同时也是应用最广泛的数据结构，定义在文件<linux/list.h\>中。

与传统链表的使用方法不同，内核的链表没有数据，只有前后指针：

```C 
struct list_head {
    struct list_head *prev, *next;
};
```

链表在使用前必须调用`INIT_LIST_HEAD`宏进行初始化。链表头节点可以用`LIST_HEAD`宏声明。

内核会将链表嵌入到其他结构体中，然后使用`container_of`宏获取结构体的地址。

!!! example "链表的使用示例"

    ```C
    struct student {
        int id;
        char name[64];
        struct list_head list;
    };

    struct student *stu = container_of(list, struct student, list);
    ```

操作链表：

```C
/*在head节点后插入new节点*/
list_add(struct list_head *new, struct list_head *head);

/*在head节点前插入new节点*/
list_add_tail(struct list_head *new, struct list_head *head);

/*删除节点*/
list_del(struct list_head *entry);

/*移除链表中的list项，然后加入到另一链表的head节点后*/
list_move(struct list_head *list, struct list_head *head);

/*判断链表是否为空*/
list_empty(struct list_head *head);

/*拼接两个链表*/
list_splice(struct list_head *list, struct list_head *head);

/*遍历链表*/
list_for_each_entry(pos, head, member);
```

### 队列

Linux内核通用队列的实现被称为kfifo，定义在<linux/kfifo.h\>中：

```C
struct __kfifo {
	unsigned int	in;
	unsigned int	out;
	unsigned int	mask;
	unsigned int	esize;
	void		*data;
};
```

Linux的kfifo和多数其他队列的实现类似，提供了两个主要操作：enqueue(入队)和dequeue(出队)。kfifo对象维护了两个偏移量:人口偏移和出口偏移。入口偏移是指下一次入队列时的位置，出口偏移是指下一次出队列时的位置。出口偏移总是小于等于入口偏移，否则无意义，因为那样说明要出队列的元素根本还没有入队列。

enqueue操作拷贝数据到队列中的入口偏移位置。当上述动作完成后，入口偏移随之加上推入的元素数目。dequeue操作从队列中出口偏移处拷贝数据，当上述动作完成后，出口偏移随之减去摘取的元素数目。当出口偏移等于入口偏移时，说明队列空了：在新数据被推入前，不可再摘取任何数据了。当入口偏移等于队列长度时，说明在队列重置前，不可再有新数据推入队列。

kfifo在使用前，必须调用`kfifo_alloc`宏进行初始化。

操作队列：

```C
/*将数据推入队列*/
kfifo_in(struct __kfifo *fifo, const void *buf, unsigned int len);

/*从队列中摘取数据*/
kfifo_out(struct __kfifo *fifo, void *buf, unsigned int len);

/*获取队列的字节大小*/
kfifo_size(struct __kfifo *fifo);

/*判断队列是否为空*/
kfifo_is_empty(struct __kfifo *fifo);

/*判断队列是否已满*/
kfifo_is_full(struct __kfifo *fifo);
```

### 映射

映射，即关联数组，是一个由唯一键组成的集合，每个键关联一个特定的值，这种键值对的关联关系就叫做映射。定义在<linux/idr.h\>中。

```C
struct idr {
	struct radix_tree_root	idr_rt;
	unsigned int		idr_base;
	unsigned int		idr_next;
};
```

### 二叉树

一个二叉搜索树(BST)是一个节点有序的二叉树，遵循以下规则：

- 根的左分支节点值都小于根节点值
- 右分支节点值都大于根节点值
- 所有的子树也是二叉搜索树

红黑树是一种自平衡二叉搜索树，内核实现的红黑树叫rbtree，定义在<linux/rbtree.h\>，它遵循以下规则：

- 所有的节点要么红色，要么黑色
- 叶子节点都是黑色
- 叶子节点不包含数据
- 所有的非叶子节点都有两个子节点
- 如果一个节点是红色，那么它的子节点都是黑色
- 在一个节点到其叶子节点的路径中，如果总是包含同样数目的黑色节点，则该路径相比其他路径是最短的

```C
struct rb_node {
	unsigned long  __rb_parent_color;
	struct rb_node *rb_right;
	struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```



