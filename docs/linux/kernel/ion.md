# ION内存管理器

ION通用内存管理器是由谷歌开发的一个用于管理内存的子系统，旨在提高内存分配的效率和灵活性，尤其是在需要高性能内存访问的应用中，如图形、多媒体和相机子系统。

在SoC中，许多设备都具有访问DMA的能力以及不同的内存分配机制，ION提供了一种通用的内存分配方法，解决了不同设备之间内存管理碎片化的问题。

ION允许不同的系统组件（如GPU、视频编码器、相机接口等）通过共享内存缓冲区来高效地交换数据，这种共享可以是零拷贝的。这减少了数据复制和转换的需要，从而降低了系统开销，提高了性能。它可以提供驱动之间、用户进程之间、内核空间和用户空间之间的共享内存。

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

下面这个结构体需要由用户空间初始化并传递给`ioctl()`函数：

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

联合体`ion_ioctl_arg`用来判断用户是分配内存还是查询：

```C
union ion_ioctl_arg {
	struct ion_allocation_data allocation;
	struct ion_heap_query query;
};
```

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

`ion_buffer`的定义如下：

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

`ion_heap`的定义如下：

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

操作`ion_heap`的回调函数：

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

> map_kernel：将内存映射到内核空间

> unmap_kernel：取消内核空间映射

> map_user：将内存映射到用户空间

## scatterlist

在继续深入之前，我们有必要了解下scatterlist——散列表，它用于组织分散的内存。

我们知道，CPU、DMA都有自己的访问内存的方式：

- CPU以MMU虚拟地址的方式
- DMA以直接物理地址的方式

于是，当内存要在这二者之间共享时，存在一个根本矛盾：在CPU的视角，由于有MMU机制，它根本不关心物理地址是否连续，只要虚拟地址连续即可。但是DMA没有MMU机制，其申请的内存必须是连续的，尤其是在传输图像、视频时，更是需要一大块连续的内存地址。

为了解决这个根本矛盾，scatterlist诞生了，它用来描述这一个个不连续的物理内存块：

```C 
struct scatterlist {
	unsigned long	page_link;
	unsigned int	offset;
	unsigned int	length;
	dma_addr_t	dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
	unsigned int	dma_length;
#endif
};
```

> page_link：指示该内存块所在的页面

> offset：该内存块在页面中的偏移

> length：该内存块的长度

> dma_address：该内存块的起始地址

> dma_length：相应的长度信息

### struct sg_table

在实际场景中，单个scatterlist是无法使用的，我们需要多个scatterlist组成一个数组，以表示在物理上不连续的虚拟地址空间：

```C
struct sg_table {
	struct scatterlist *sgl;	/* the list */
	unsigned int nents;		/* number of mapped entries */
	unsigned int orig_nents;	/* original size of list */
};
```

> sgl：内存块数组的首地址

> nents：有效的内存块个数

> orig_nents：内存块数组的size

sg_table中到底有多少个有效内存块？其实是由`struct scatterlist`中的page_link字段决定的。如果它的 bit0 为1，表示它不是一个有效的内存块，而是指向另一个scatterlist数组。如果 bit1 为1，表示它是scatterlist数组中最后一个有效的内存块。

## 分配heap

前面提到，ION对于内存主要分成了四个区，因此heap的分配也有四种策略：

- 不连续内存
- 连续内存
- 保留区内存
- CMA内存

整体流程为：

1. 用户层打开/dev/ion，并通过`ioctl()`函数传递参数给驱动层。
2. 驱动层调用`ion_ioctl()`函数解析用户传递的参数。

	```C
	static long ion_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
	{
		int ret = 0;
		union ion_ioctl_arg data;

		if (_IOC_SIZE(cmd) > sizeof(data))
			return -EINVAL;

		/*
		* The copy_from_user is unconditional here for both read and write
		* to do the validate. If there is no write for the ioctl, the
		* buffer is cleared
		*/
		if (copy_from_user(&data, (void __user *)arg, _IOC_SIZE(cmd)))
			return -EFAULT;

		ret = validate_ioctl_arg(cmd, &data);
		if (ret) {
			pr_warn_once("%s: ioctl validate failed\n", __func__);
			return ret;
		}

		if (!(_IOC_DIR(cmd) & _IOC_WRITE))
			memset(&data, 0, sizeof(data));

		switch (cmd) {
		case ION_IOC_ALLOC:
		{
			int fd;

			fd = ion_alloc(data.allocation.len,
					data.allocation.heap_id_mask,
					data.allocation.flags);
			if (fd < 0)
				return fd;

			data.allocation.fd = fd;

			break;
		}
		case ION_IOC_HEAP_QUERY:
			ret = ion_query_heaps(&data.query);
			break;
		default:
			return -ENOTTY;
		}

		if (_IOC_DIR(cmd) & _IOC_READ) {
			if (copy_to_user((void __user *)arg, &data, _IOC_SIZE(cmd)))
				return -EFAULT;
		}
		return ret;
	}
	```

3. 继续调用`ion_alloc()`寻找合适的heap分配内存，并且将`ion_buffer`以`dma_buffer`的形式返回给用户。

	```C
	static int ion_alloc(size_t len, unsigned int heap_id_mask, unsigned int flags)
	{
		struct ion_device *dev = internal_dev;
		struct ion_buffer *buffer = NULL;
		struct ion_heap *heap;
		DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
		int fd;
		struct dma_buf *dmabuf;

		pr_debug("%s: len %zu heap_id_mask %u flags %x\n", __func__,
			len, heap_id_mask, flags);
		/*
		* traverse the list of heaps available in this system in priority
		* order.  If the heap type is supported by the client, and matches the
		* request of the caller allocate from it.  Repeat until allocate has
		* succeeded or all heaps have been tried
		*/
		len = PAGE_ALIGN(len);

		if (!len)
			return -EINVAL;

		down_read(&dev->lock);

		//遍历ion_device的所有heap，查找与用户匹配的heap
		plist_for_each_entry(heap, &dev->heaps, node) {
			/* if the caller didn't specify this heap id */
			if (!((1 << heap->id) & heap_id_mask))
				continue;
			buffer = ion_buffer_create(heap, dev, len, flags);
			if (!IS_ERR(buffer))
				break;
		}
		up_read(&dev->lock);

		if (!buffer)
			return -ENODEV;

		if (IS_ERR(buffer))
			return PTR_ERR(buffer);

		exp_info.ops = &dma_buf_ops;
		exp_info.size = buffer->size;
		exp_info.flags = O_RDWR;
		exp_info.priv = buffer;

		//导出为dma_buffer
		dmabuf = dma_buf_export(&exp_info);
		if (IS_ERR(dmabuf)) {
			_ion_buffer_destroy(buffer);
			return PTR_ERR(dmabuf);
		}

		//将dma_buffer转换成fd，返回给用户空间
		fd = dma_buf_fd(dmabuf, O_CLOEXEC);
		if (fd < 0)
			dma_buf_put(dmabuf);

		return fd;
	}
	```

4. 根据选中的heap，调用对应的`allocate()`分配函数。
5. 使用完后，调用`free()`函数释放内存。

### 不连续内存

> 源码位于<drivers/staging/android/ion/ion_system_heap.c\>

1. 分配内存：

	```C
	static int ion_system_heap_allocate(struct ion_heap *heap,
						struct ion_buffer *buffer,
						unsigned long size,
						unsigned long flags)
	{
		struct ion_system_heap *sys_heap = container_of(heap,
								struct ion_system_heap,
								heap);
		struct sg_table *table;
		struct scatterlist *sg;
		struct list_head pages;
		struct page *page, *tmp_page;
		int i = 0;
		unsigned long size_remaining = PAGE_ALIGN(size);
		unsigned int max_order = orders[0];

		if (size / PAGE_SIZE > totalram_pages() / 2)
			return -ENOMEM;

		INIT_LIST_HEAD(&pages);
		while (size_remaining > 0) {
			page = alloc_largest_available(sys_heap, buffer, size_remaining,
							max_order);
			if (!page)
				goto free_pages;
			list_add_tail(&page->lru, &pages);
			size_remaining -= page_size(page);
			max_order = compound_order(page);
			i++;
		}
		table = kmalloc(sizeof(*table), GFP_KERNEL);
		if (!table)
			goto free_pages;

		if (sg_alloc_table(table, i, GFP_KERNEL))
			goto free_table;

		sg = table->sgl;
		list_for_each_entry_safe(page, tmp_page, &pages, lru) {
			sg_set_page(sg, page, page_size(page), 0);
			sg = sg_next(sg);
			list_del(&page->lru);
		}

		buffer->sg_table = table;
		return 0;

	free_table:
		kfree(table);
	free_pages:
		list_for_each_entry_safe(page, tmp_page, &pages, lru)
			free_buffer_page(sys_heap, buffer, page);
		return -ENOMEM;
	}
	```

2. 释放内存：

	```C
	static void ion_system_heap_free(struct ion_buffer *buffer)
	{
		struct ion_system_heap *sys_heap = container_of(buffer->heap,
								struct ion_system_heap,
								heap);
		struct sg_table *table = buffer->sg_table;
		struct scatterlist *sg;
		int i;

		/* zero the buffer before goto page pool */
		if (!(buffer->private_flags & ION_PRIV_FLAG_SHRINKER_FREE))
			ion_heap_buffer_zero(buffer);

		for_each_sgtable_sg(table, sg, i)
			free_buffer_page(sys_heap, buffer, sg_page(sg));
		sg_free_table(table);
		kfree(table);
	}
	```

### 连续内存

> 源码位于<drivers/staging/android/ion/ion_system_heap.c\>

1. 分配内存：

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

2. 释放内存

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

### 保留区内存

1. 分配内存：

2. 释放内存：




### CMA内存

> 源码位于<drivers/staging/android/ion/ion_cma_heap.c\>

1. 分配内存：

	```C
	static int ion_cma_allocate(struct ion_heap *heap, struct ion_buffer *buffer,
					unsigned long len,
					unsigned long flags)
	{
		struct ion_cma_heap *cma_heap = to_cma_heap(heap);
		struct sg_table *table;
		struct page *pages;
		unsigned long size = PAGE_ALIGN(len);
		unsigned long nr_pages = size >> PAGE_SHIFT;
		unsigned long align = get_order(size);
		int ret;

		if (align > CONFIG_CMA_ALIGNMENT)
			align = CONFIG_CMA_ALIGNMENT;

		pages = cma_alloc(cma_heap->cma, nr_pages, align, false);
		if (!pages)
			return -ENOMEM;

		if (PageHighMem(pages)) {
			unsigned long nr_clear_pages = nr_pages;
			struct page *page = pages;

			while (nr_clear_pages > 0) {
				void *vaddr = kmap_atomic(page);

				memset(vaddr, 0, PAGE_SIZE);
				kunmap_atomic(vaddr);
				page++;
				nr_clear_pages--;
			}
		} else {
			memset(page_address(pages), 0, size);
		}

		table = kmalloc(sizeof(*table), GFP_KERNEL);
		if (!table)
			goto err;

		ret = sg_alloc_table(table, 1, GFP_KERNEL);
		if (ret)
			goto free_mem;

		sg_set_page(table->sgl, pages, size, 0);

		buffer->priv_virt = pages;
		buffer->sg_table = table;
		return 0;

	free_mem:
		kfree(table);
	err:
		cma_release(cma_heap->cma, pages, nr_pages);
		return -ENOMEM;
	}
	```

2. 释放内存：

	```C
	static void ion_cma_free(struct ion_buffer *buffer)
	{
		struct ion_cma_heap *cma_heap = to_cma_heap(buffer->heap);
		struct page *pages = buffer->priv_virt;
		unsigned long nr_pages = PAGE_ALIGN(buffer->size) >> PAGE_SHIFT;

		/* release memory */
		cma_release(cma_heap->cma, pages, nr_pages);
		/* release sg table */
		sg_free_table(buffer->sg_table);
		kfree(buffer->sg_table);
	}
	```




