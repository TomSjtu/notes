# ION内存管理器

ION通用内存管理器是由谷歌开发的一个用于管理内存的子系统，ION旨在提高内存分配的效率和灵活性，尤其是在需要高性能内存访问的应用中，如图形、多媒体和相机子系统。

在SoC中，许多设备都具有访问DMA的能力以及不同的内存分配机制，ION提供了一种通用的内存分配方法，解决了不同设备之间内存管理碎片化的问题。

ION允许不同的系统组件（如GPU、视频编码器、相机接口等）通过共享内存缓冲区来高效地交换数据，这种共享可以是零拷贝的。这减少了数据复制和转换的需要，从而降低了系统开销，提高了性能。它可以提供驱动之间、用户进程之间、内核空间和用户空间之间的共享内存。

在实际使用中，ION 和VIDEOBUF2、DMA-BUF、V4L2等结合的很紧密。

ION源码文件在<drivers/staging/android/ion\>目录下。

## 数据结构

### 用户态

在ION中，为了管理不同的内存类型，首先要对它们进行分类：

```C
enum ion_heap_type {
	ION_HEAP_TYPE_SYSTEM,
	ION_HEAP_TYPE_SYSTEM_CONTIG,
	ION_HEAP_TYPE_CARVEOUT,
	ION_HEAP_TYPE_CHUNK,
	ION_HEAP_TYPE_DMA,
	ION_HEAP_TYPE_CUSTOM, /*
			       * must be last so device specific heaps always
			       * are at the end of this enum
			       */
};
```

> ION_HAEP_TYPE_SYSTEM：`vmalloc`分配的内存

> ION_HAEP_TYPE_SYSTEM_CONTIG：`kmalloc`分配的内存

> ION_HEAP_TYPE_CARVEOUT：在启动时就保留的内存区，物理上连续

> ION_HEAP_TYPE_DMA：分配给DMA的内存

该结构体由用户空间传递：

```C
struct ion_allocation_data {
	__u64 len;
	__u32 heap_id_mask;
	__u32 flags;
	__u32 fd;
	__u32 unused;
};
```

> len：需要分配的字节数

> heap_id_mask：需要从哪个heap中分配内存

> fd：分配后的内存转换成dma-buffer的fd

该结构体表示heap的信息：

```C
struct ion_heap_data {
	char name[MAX_HEAP_NAME];
	__u32 type;
	__u32 heap_id;
	__u32 reserved0;
	__u32 reserved1;
	__u32 reserved2;
};
```

> name：heap的名字

> type：heap的类型

> heap_id：heap的ID

该结构体表示所有heap的集合：

```C
struct ion_heap_query {
	__u32 cnt; /* Total number of heaps to be copied */
	__u32 reserved0; /* align to 64bits */
	__u64 heaps; /* buffer to be populated */
	__u32 reserved1;
	__u32 reserved2;
};
```

### 驱动态

buffer的定义如下：

```C
struct ion_buffer {
	struct list_head list;
	struct ion_device *dev;
	struct ion_heap *heap;
	unsigned long flags;
	unsigned long private_flags;
	size_t size;
	void *priv_virt;
	struct mutex lock;
	int kmap_cnt;
	void *vaddr;
	struct sg_table *sg_table;
	struct list_head attachments;
};
```

> flags：buffer特定标志

> private_flags：内部buffer特定标志

> size：buffer的大小

> priv_virt：buffer私有数据

> kmap_cnt：buffer映射到内核的次数

> vaddr：buffer虚拟地址

> sg_table：该buffer的SG表，SG表可以让数据在不连续的内存块之间传输。

> attachments：表示与该buffer关联的设备链表

`struct ion_device`包含了ion device node的元数据：
```C
struct ion_device {
	struct miscdevice dev;
	struct rw_semaphore lock;
	struct plist_head heaps;
	struct dentry *debug_root;
	int heap_cnt;
};
```

> dev：以杂项设备注册

> heaps：该设备关联的所有heap

> heap_cnt：heap的数量

堆数据结构：
```C
struct ion_heap {
	struct plist_node node;
	struct ion_device *dev;
	enum ion_heap_type type;
	struct ion_heap_ops *ops;
	unsigned long flags;
	unsigned int id;
	const char *name;

	/* deferred free support */
	struct shrinker shrinker;
	struct list_head free_list;
	size_t free_list_size;
	spinlock_t free_lock;
	wait_queue_head_t waitqueue;
	struct task_struct *task;

	/* heap statistics */
	u64 num_of_buffers;
	u64 num_of_alloc_bytes;
	u64 alloc_bytes_wm;

	/* protect heap statistics */
	spinlock_t stat_lock;
};
```

操作ION heap的函数有：

```C
struct ion_heap_ops {
	int (*allocate)(struct ion_heap *heap,
			struct ion_buffer *buffer, unsigned long len,
			unsigned long flags);
	void (*free)(struct ion_buffer *buffer);
	void * (*map_kernel)(struct ion_heap *heap, struct ion_buffer *buffer);
	void (*unmap_kernel)(struct ion_heap *heap, struct ion_buffer *buffer);
	int (*map_user)(struct ion_heap *mapper, struct ion_buffer *buffer,
			struct vm_area_struct *vma);
	int (*shrink)(struct ion_heap *heap, gfp_t gfp_mask, int nr_to_scan);
};
```

> allocate：分配内存

> free：释放内存

> map_kerne：将内存映射到内核空间

> unmap_kernel：取消内核空间映射

> map_user：将内存映射到用户空间

## 分配heap

前面提到，ION对于内存主要分成了四个区，因此heap的分配也有四种策略：

- 连续内存
- 不连续内存
- 保留区内存
- CMA内存

### 连续内存

> 源码位于<drivers/staging/android/ion/ion_system_heap.c\>

连续内存的分配比较简单，在初始化`struct ion_heap`之后，设置回调函数指针，其中最重要的分配函数为`ion_system_contig_heap_allocate()`。

1. 创建type为ION_HEAP_TYPE_SYSTEM_CONTIG的heap，并添加到device中：

	```C
	static int ion_system_contig_heap_create(void)
	{
		struct ion_heap *heap;

		heap = __ion_system_contig_heap_create();
		if (IS_ERR(heap))
			return PTR_ERR(heap);

		ion_device_add_heap(heap);

		return 0;
	}
	```

	该函数又调用以下函数：

	```C
	static struct ion_heap *__ion_system_contig_heap_create(void)
	{
		struct ion_heap *heap;

		heap = kzalloc(sizeof(*heap), GFP_KERNEL);
		if (!heap)
			return ERR_PTR(-ENOMEM);
		heap->ops = &kmalloc_ops;
		heap->type = ION_HEAP_TYPE_SYSTEM_CONTIG;
		heap->name = "ion_system_contig_heap";

		return heap;
	}
	```

2. 设置了heap-ops回调函数：

	```C
	static struct ion_heap_ops kmalloc_ops = {
		.allocate = ion_system_contig_heap_allocate,
		.free = ion_system_contig_heap_free,
		.map_kernel = ion_heap_map_kernel,
		.unmap_kernel = ion_heap_unmap_kernel,
		.map_user = ion_heap_map_user,
	};
	```

3. 分配函数解析

	```C
	static int ion_system_contig_heap_allocate(struct ion_heap *heap,
						struct ion_buffer *buffer,
						unsigned long len,
						unsigned long flags)
	{
		int order = get_order(len);
		struct page *page;
		struct sg_table *table;
		unsigned long i;
		int ret;
		//直接从buddy中分配内存页
		//分配的内存页是可能比实际请求的大的，比如申请len是3个page大小，那么order就为2，实际申请了4个page
		page = alloc_pages(low_order_gfp_flags, order);
		if (!page)
			return -ENOMEM;
		//将申请到的连续内存页分割成一页页
		split_page(page, order);

		//由于在分配时可能多分配，因此需要将多余的page释放回去。比如申请3个page，实际分配了4个
		len = PAGE_ALIGN(len);
		for (i = len >> PAGE_SHIFT; i < (1 << order); i++)
			__free_page(page + i);
		//接着申请sg_table
		table = kmalloc(sizeof(*table), GFP_KERNEL);
		if (!table) {
			ret = -ENOMEM;
			goto free_pages;
		}
		//由于是连续的内存，因此只需要申请一个scatterlist
		ret = sg_alloc_table(table, 1, GFP_KERNEL);
		if (ret)
			goto free_table;
		//将连续内存首页地址存到sg_table中
		sg_set_page(table->sgl, page, len, 0);

		buffer->sg_table = table;

		return 0;

	free_table:
		kfree(table);
	free_pages:
		for (i = 0; i < len >> PAGE_SHIFT; i++)
			__free_page(page + i);

		return ret;
	}
	```

4. 释放内存

	```C
	static void ion_system_contig_heap_free(struct ion_buffer *buffer)
	{
		struct sg_table *table = buffer->sg_table;
		struct page *page = sg_page(table->sgl);
		unsigned long pages = PAGE_ALIGN(buffer->size) >> PAGE_SHIFT;
		unsigned long i;

		//释放时就直接将内存页归还给buddy，__free_page是buddy system的接口
		for (i = 0; i < pages; i++)
			__free_page(page + i);
		sg_free_table(table);
		kfree(table);
	}
	```


### 不连续内存

> 源码位于<drivers/staging/android/ion/ion_system_heap.c\>

创建type为ION_HEAP_TYPE_SYSTEM的heap，并添加到device中：
```C
static int ion_system_heap_create(void)
{
	struct ion_heap *heap;

	heap = __ion_system_heap_create();
	if (IS_ERR(heap))
		return PTR_ERR(heap);
	heap->name = "ion_system_heap";

	ion_device_add_heap(heap);

	return 0;
}
```

核心逻辑：

```C
static struct ion_heap *__ion_system_heap_create(void)
{
	struct ion_system_heap *heap;

	heap = kzalloc(sizeof(*heap), GFP_KERNEL);
	if (!heap)
		return ERR_PTR(-ENOMEM);
	heap->heap.ops = &system_heap_ops;
	heap->heap.type = ION_HEAP_TYPE_SYSTEM;
	heap->heap.flags = ION_HEAP_FLAG_DEFER_FREE;

	if (ion_system_heap_create_pools(heap->pools))
		goto free_heap;

	return &heap->heap;

free_heap:
	kfree(heap);
	return ERR_PTR(-ENOMEM);
}
```







```C
static int ion_system_contig_heap_allocate(struct ion_heap *heap,
					   struct ion_buffer *buffer,
					   unsigned long len,
					   unsigned long flags)
{
	int order = get_order(len);
	struct page *page;
	struct sg_table *table;
	unsigned long i;
	int ret;

	//allocate pages from buddy system
	page = alloc_pages(low_order_gfp_flags | __GFP_NOWARN, order);
	if (!page)
		return -ENOMEM;

	split_page(page, order);

	len = PAGE_ALIGN(len);
	for (i = len >> PAGE_SHIFT; i < (1 << order); i++)
		__free_page(page + i);

	table = kmalloc(sizeof(*table), GFP_KERNEL);
	if (!table) {
		ret = -ENOMEM;
		goto free_pages;
	}

	ret = sg_alloc_table(table, 1, GFP_KERNEL);
	if (ret)
		goto free_table;

	sg_set_page(table->sgl, page, len, 0);

	buffer->sg_table = table;

	return 0;

free_table:
	kfree(table);
free_pages:
	for (i = 0; i < len >> PAGE_SHIFT; i++)
		__free_page(page + i);

	return ret;
}
```


```C
static void ion_system_contig_heap_free(struct ion_buffer *buffer)
{
	struct sg_table *table = buffer->sg_table;
	struct page *page = sg_page(table->sgl);
	unsigned long pages = PAGE_ALIGN(buffer->size) >> PAGE_SHIFT;
	unsigned long i;

	for (i = 0; i < pages; i++)
		__free_page(page + i);
	sg_free_table(table);
	kfree(table);
}
```

### 保留区内存

### CMA内存








