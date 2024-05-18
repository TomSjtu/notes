# 连续内存分配

连续内存分配(CMA，Continuous Memory Allocator)是内存管理中的一个模块，主要负责物理地址上连续的内存分配。一般系统会在启动过程中，从整个memory中配置一段连续内存用于CMA，然后内核其他的模块可以通过CMA的接口API进行连续内存的分配。CMA的核心并不是设计精巧的算法来管理地址连续的内存块，实际上它的底层还是依赖内核伙伴系统这样的内存管理机制，或者说CMA是处于需要连续内存块的其他内核模块（例如DMA mapping framework）和内存管理模块之间的一个中间层模块，主要功能包括：

- 解析DTS或者命令行中的参数，确定CMA内存的区域，这样的区域我们定义为CMA area。

- 提供`cma_alloc()`和`cma_release()`两个接口函数用于分配和释放CMA pages。

- 记录和跟踪CMA area中各个pages的状态。

- 调用伙伴系统接口，进行真正的内存分配。

!!! question 

    为什么需要CMA？

对大块连续内存的需求主要来自于各种驱动，比如图像、视频驱动。假如拍摄一张高清图像需要分配20MB的内存：在系统刚刚启动时，内存的余量还比较充足，因此这可能没什么大不了。但是随着系统的运行，内存不断地分配和释放，大块内存不断地被分解、分解，再分解，内存的碎片化导致分配地址连续的大块内存越来越艰难。一旦内存分配失败，camera功能就无法使用，用户当然不会答应。

ARM64支持多种页面大小——4k、64K甚至更大的page size。但在大多数CPU架构上，Linux内核总是倾向于使用最小的page size——4K。大于4K的page统称为"huge page"。huge page可以帮我们应对大块内存分配的需求，但是page size的设置牵一发而动全身，关系到整个系统的稳定性和性能，因此不能把它当作解决大块内存分配的方法。

CMA就是为了解决上述困境而被引入的。对于标记为CMA的内存，当前驱动没有分配使用的时候，这些内存可以被内核的其他模块使用（当然有一定的要求）。而当驱动要求分配CMA内存后，其他模块需要吐出来，形成物理地址连续的大块内存，给具体的驱动来使用。

## 设备树配置

按照CMA的使用范围，它也可以分为两种类型，一种是通用的CMA区域，该区域是给整个系统分配使用的，另一种是专用的CMA区域，这种是专门为单个模块定义的，定义它的目的是不太希望和其他模块共享该区域，我们可以在dts中定义不同的CMA区域，每个区域实际上就是一个reserved memory，对于共享的CMA：

```DTS
reserved_memory: reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;
 
    /* global autoconfigured region for contiguous allocations */
    linux,cma {
        compatible = "shared-dma-pool";
        alloc-ranges = <0x0 0x00000000 0x0 0xffffffff>;
        reusable;
        alignment = <0x0 0x400000>;
        size = <0x0 0x2000000>;
        linux,cma-default;
    };
};
```

对于CMA区域的dts配置来说，有三个关键点：

1. 一定要包含有reusable，表示当前的内存区域除了被dma使用之外，还可以被内存管理子系统reuse。
2. 不能包含有no-map属性，该属性表示是否需要创建页表映射，对于通用的内存，必须要创建映射才可以使用，而CMA是可以作为通用内存进行分配使用的，因此必须要创建页表映射。
3. 对于共享的CMA区域，需要配置上`linux,cma-default`属性，表示它是共享的CMA。

对于一个专用的CMA，它的配置方式如下：

```DTS
reserved_memory: reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;
 
    priv_mem: priv_region {
        compatible = "shared-dma-pool";
        alloc-ranges = <0x0 0x00000000 0x0 0xffffffff>;
        reusable;
        alignment = <0x0 0x400000>;
        size = <0x0 0xC00000>;
    };
};
```

这里和上面共享的唯一区别就是在专用CMA区域中是不包含`linux,cma-default`属性的/

## 数据结构

`struct cma`用于管理一个CMA区域：

```C
struct cma {
	unsigned long   base_pfn;
	unsigned long   count;
	unsigned long   *bitmap;
	unsigned int order_per_bit; /* Order of pages represented by one bit */
	struct mutex    lock;
	char name[CMA_MAX_NAME];
};

//全局CMA数组
extern struct cma cma_areas[MAX_CMA_AREAS];
```

> base_pfn：CMA区域物理地址起始帧号

> count：CMA区域总的页数

> bitmap：CMA区域位图，用于记录哪些page被分配了，0表示free，1表示已分配

> order_per_bit：每个bit代表page的order值

配置CMA内存区有两种方法，一种是通过dts的reserved memory，另外一种是通过command line参数和内核配置参数。

## MIGRATE_CMA

在<include/linux/mmzone.h\>中定义了如下migrate type：

```C
enum migratetype {
	MIGRATE_UNMOVABLE,
	MIGRATE_MOVABLE,
	MIGRATE_RECLAIMABLE,
	MIGRATE_PCPTYPES,	/* the number of types on the pcp lists */
	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
	MIGRATE_CMA,

#ifdef CONFIG_MEMORY_ISOLATION
	MIGRATE_ISOLATE,	/* can't allocate from here */
#endif
	MIGRATE_TYPES
};
```

被标记为MIGRATE_MOVABLE的页，表示该页面上的数据是可以迁移的。也就是说，如果需要，我们可以分配一个新的页，将数据拷贝到这个新的页上，然后释放旧的页。这样的拷贝操作对系统没有任何的影响。

[伙伴系统](./mm.md/#_5)不会跟踪每一个页框的migrate type，这样成本太高了，它是基于页块(多个页框的组合)来管理的。在处理内存分配请求的时候，一般会优先从相同migrate type的页块中分配页面。如果分配不成功，不同migrate type的页块也会考虑，甚至可能改变页块的migrate type属性。这意味着一个UNMOVABLE的页面请求也可以从migrate type是MOVABLE的页块中分配。这一点对于CMA来说是不能接受的，所以我们引入了一个新的migrate type：MIGRATE_CMA。这种类型具有一个重要性质：只有可移动的页面可以从MIGRATE_CMA的页块中分配。

## 分配内存

```C
struct page *cma_alloc(struct cma *cma, size_t count, unsigned int align,
		       bool no_warn)
{
	unsigned long mask, offset;
	unsigned long pfn = -1;
	unsigned long start = 0;
	unsigned long bitmap_maxno, bitmap_no, bitmap_count;
	size_t i;
	struct page *page = NULL;
	int ret = -ENOMEM;

	if (!cma || !cma->count || !cma->bitmap)
		return NULL;

	pr_debug("%s(cma %p, count %zu, align %d)\n", __func__, (void *)cma,
		 count, align);

	if (!count)
		return NULL;

	mask = cma_bitmap_aligned_mask(cma, align);
	offset = cma_bitmap_aligned_offset(cma, align);
	bitmap_maxno = cma_bitmap_maxno(cma);
	bitmap_count = cma_bitmap_pages_to_bits(cma, count);

	if (bitmap_count > bitmap_maxno)
		return NULL;

	for (;;) {
		mutex_lock(&cma->lock);

        //查找连续多个为0的bit位
		bitmap_no = bitmap_find_next_zero_area_off(cma->bitmap,
				bitmap_maxno, start, bitmap_count, mask,
				offset);
		if (bitmap_no >= bitmap_maxno) {
			mutex_unlock(&cma->lock);
			break;
		}

        //将查找到的多个bit设为1，表示内存被占用
		bitmap_set(cma->bitmap, bitmap_no, bitmap_count);
		/*
		 * It's safe to drop the lock here. We've marked this region for
		 * our exclusive use. If the migration fails we will take the
		 * lock again and unmark it.
		 */
		mutex_unlock(&cma->lock);

        //计算分配的起始页的页号
		pfn = cma->base_pfn + (bitmap_no << cma->order_per_bit);
		mutex_lock(&cma_mutex);

        //从起始页开始连续分配count个页
		ret = alloc_contig_range(pfn, pfn + count, MIGRATE_CMA,
				     GFP_KERNEL | (no_warn ? __GFP_NOWARN : 0));
		mutex_unlock(&cma_mutex);

        //分配成功，返回起始页
		if (ret == 0) {
			page = pfn_to_page(pfn);
			break;
		}

		cma_clear_bitmap(cma, pfn, count);
		if (ret != -EBUSY)
			break;

		pr_debug("%s(): memory range at %p is busy, retrying\n",
			 __func__, pfn_to_page(pfn));
		/* try again with a bit different memory target */
		start = bitmap_no + mask + 1;
	}

	trace_cma_alloc(pfn, page, count, align);

	/*
	 * CMA can allocate multiple page blocks, which results in different
	 * blocks being marked with different tags. Reset the tags to ignore
	 * those page blocks.
	 */
	if (page) {
		for (i = 0; i < count; i++)
			page_kasan_tag_reset(page + i);
	}

	if (ret && !no_warn) {
		pr_err("%s: alloc failed, req-size: %zu pages, ret: %d\n",
			__func__, count, ret);
		cma_debug_show_areas(cma);
	}

	pr_debug("%s(): returned %p\n", __func__, page);
	return page;
}
```

## 释放内存

```C
bool cma_release(struct cma *cma, const struct page *pages, unsigned int count)
{
	unsigned long pfn;

	if (!cma || !pages)
		return false;

	pr_debug("%s(page %p)\n", __func__, (void *)pages);

	pfn = page_to_pfn(pages);

	if (pfn < cma->base_pfn || pfn >= cma->base_pfn + cma->count)
		return false;

	VM_BUG_ON(pfn + count > cma->base_pfn + cma->count);

    //释放回buddy系统
	free_contig_range(pfn, count);

    //清零bit位，表示cma内存可用
	cma_clear_bitmap(cma, pfn, count);
	trace_cma_release(pfn, pages, count);

	return true;
}
```


