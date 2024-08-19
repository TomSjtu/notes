# ARM64的启动

## 进入内核前

在系统启动前，控制权在引导程序(bootloader)手中，这里的 bootloader 是一个宽泛的概念，泛指一切为内核准备好执行环境的那些软件，可以是 uboot，也可以是 Hypervisor 或者是 secure monitor。

不管是哪种引导程序，都需要完成以下步骤：

1. 初始化 RAM 信息
2. 准备好设备树信息，并将设备树首地址告知内核
3. 解压内核(非强制)
4. 将控制权交给内核

在跳转至内核前，必须满足以下状态：

- 关闭 MMU
- 关闭 D-cache(数据缓存)，I-cache(指令缓存)开启或关闭都可
- x0 寄存器用来保存设备树的物理地址
- CPU 必须处于 EL2 模式(推荐，可以访问虚拟化扩展)或非安全 EL1 模式下

!!! question "为什么必须关闭数据缓存"

    数据缓存有可能缓存了 bootloader 的数据，如果不清除，可能导致内核访问错误的数据。而 bootloader 的指令与内核指令无关，所以可以不关闭指令缓存。

更详细的 booting protocol 请参考<Documentation/arm64/booting.rst\>文档。

## 启动代码分析

ARM64 的启动代码位于<arch/arm64/kernel/head.S\>文件中，代码如下：

```C
SYM_CODE_START(primary_entry)
	bl	preserve_boot_args
	bl	init_kernel_el			// w0=cpu_boot_mode
	adrp	x23, __PHYS_OFFSET
	and	x23, x23, MIN_KIMG_ALIGN - 1	// KASLR offset, defaults to 0
	bl	set_cpu_boot_mode_flag
	bl	__create_page_tables
	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
	bl	__cpu_setup			// initialise processor
	b	__primary_switch
SYM_CODE_END(primary_entry)
```

启动的入口函数为`primary_entry`，该函数又调用了一系列初始化函数，我们逐个分析。

### 保存参数

```C
SYM_CODE_START_LOCAL(preserve_boot_args)
	mov	x21, x0				// 保存设备树的地址至x21寄存器

	adr_l	x0, boot_args			// 将boot_args变量的地址保存至x0
	stp	x21, x1, [x0]			// 保存x0和x1的值到boot_args[0]和boot_args[1]
	stp	x2, x3, [x0, #16]		//保存x2和x3的值到boot_args[2]和boot_args[3]

	dmb	sy				// 数据内存屏障指令，sy表示全系统共享域

	add	x1, x0, #0x20			// 4 x 8 bytes
	b	dcache_inval_poc		// tail call
SYM_CODE_END(preserve_boot_args)
```

在启动前，MMU 和 D-cache 已经被关闭，因此存储指令略过了 cache，直接写入 RAM。但是为了安全起见，仍然需要清除 cache ——调用`dcache_inval_poc`函数。

!!! question "x0~x3寄存器"

    为何要保留这四个寄存器的值？在 boot protocol 中有解释：x0 是 dtb 的地址，而x1~x3 必须为0，用来保留使用。执行函数`setup_arch()`时，会校验 boot_args 的值。

### 初始化异常等级

我们期望系统从 EL2 模式下启动，如果不是，则需要先配置相关环境，在返回前，必须将 BOOT_CPU_MODE 的值保存在 w0 寄存器中：

```C
/*
 * Starting from EL2 or EL1, configure the CPU to execute at the highest
 * reachable EL supported by the kernel in a chosen default state. If dropping
 * from EL2 to EL1, configure EL2 before configuring EL1.
 *
 * Since we cannot always rely on ERET synchronizing writes to sysregs (e.g. if
 * SCTLR_ELx.EOS is clear), we place an ISB prior to ERET.
 *
 * Returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in w0 if
 * booted in EL1 or EL2 respectively.
 */
SYM_FUNC_START(init_kernel_el)
	mrs	x0, CurrentEL                   //读取当前异常等级至x0寄存器
	cmp	x0, #CurrentEL_EL2              //与EL2进行比较
	b.eq	init_el2                    //如果相等，跳转到init_el2函数
										//不相等则执行下面代码

SYM_INNER_LABEL(init_el1, SYM_L_LOCAL)
	mov_q	x0, INIT_SCTLR_EL1_MMU_OFF	
	msr	sctlr_el1, x0					
	isb									
	mov_q	x0, INIT_PSTATE_EL1	
	msr	spsr_el1, x0					
	msr	elr_el1, lr
	mov	w0, #BOOT_CPU_MODE_EL1
	eret								//ERET指令用于从异常中返回

SYM_INNER_LABEL(init_el2, SYM_L_LOCAL)
	mov_q	x0, HCR_HOST_NVHE_FLAGS
	msr	hcr_el2, x0
	isb

	init_el2_state

	/* Hypervisor stub */
	adr_l	x0, __hyp_stub_vectors
	msr	vbar_el2, x0					//设置向量基地址寄存器VBAR_EL2
	isb

	/*
	 * Fruity CPUs seem to have HCR_EL2.E2H set to RES1,
	 * making it impossible to start in nVHE mode. Is that
	 * compliant with the architecture? Absolutely not!
	 */
	mrs	x0, hcr_el2
	and	x0, x0, #HCR_E2H
	cbz	x0, 1f

	/* Switching to VHE requires a sane SCTLR_EL1 as a start */
	mov_q	x0, INIT_SCTLR_EL1_MMU_OFF
	msr_s	SYS_SCTLR_EL12, x0

	/*
	 * Force an eret into a helper "function", and let it return
	 * to our original caller... This makes sure that we have
	 * initialised the basic PSTATE state.
	 */
	mov	x0, #INIT_PSTATE_EL2
	msr	spsr_el1, x0
	adr	x0, __cpu_stick_to_vhe
	msr	elr_el1, x0
	eret

1:
	mov_q	x0, INIT_SCTLR_EL1_MMU_OFF
	msr	sctlr_el1, x0

	msr	elr_el2, lr
	mov	w0, #BOOT_CPU_MODE_EL2
	eret

__cpu_stick_to_vhe:
	mov	x0, #HVC_VHE_RESTART
	hvc	#0
	mov	x0, #BOOT_CPU_MODE_EL2
	ret
SYM_FUNC_END(init_kernel_el)
```

上面代码主要配置了一些系统寄存器的环境，然后从对应的异常等级返回。

### 设置CPU启动模式

进入该函数前，必须保证 w0 保存了 __boot_cpu_mode 的值，该值用于保存 CPU 启动时的 Exception level，它的定义如下：

```C
SYM_DATA_START(__boot_cpu_mode)
	.long	BOOT_CPU_MODE_EL2
	.long	BOOT_CPU_MODE_EL1
SYM_DATA_END(__boot_cpu_mode)
```

设置启动模式的代码如下：

```C
/*
 * Sets the __boot_cpu_mode flag depending on the CPU boot mode passed
 * in w0. See arch/arm64/include/asm/virt.h for more info.
 */
SYM_FUNC_START_LOCAL(set_cpu_boot_mode_flag)
	adr_l	x1, __boot_cpu_mode     //x1寄存器保存全局变量__boot_cpu_mode的地址
	cmp	w0, #BOOT_CPU_MODE_EL2      //比较当前启动模式是否是EL2
	b.ne	1f                      //如果不相等，则跳转至1
	add	x1, x1, #4
1:	str	w0, [x1]			//保存CPU启动模式
	dmb	sy                  //保证str指令执行完成
	dc	ivac, x1			//使高速缓存失效
	ret
SYM_FUNC_END(set_cpu_boot_mode_flag)
```

我们期望所有 CPU 都在同一模式下启动，如果都在 EL2 模式下启动，则说明系统支持虚拟化，kvm 模块才可以顺利启动。

### 建立页表

为了提高性能，加快初始化速度，必须在某个阶段打开 MMU，而在打开 MMU 之前，必须要先设定好页表。

```C
SYM_FUNC_START_LOCAL(__create_page_tables)
	mov	x28, lr

	/*
	 * Invalidate the init page tables to avoid potential dirty cache lines
	 * being evicted. Other page tables are allocated in rodata as part of
	 * the kernel image, and thus are clean to the PoC per the boot
	 * protocol.
	 */
	adrp	x0, init_pg_dir
	adrp	x1, init_pg_end
	bl	dcache_inval_poc

	/*
	 * Clear the init page tables.
	 */
	adrp	x0, init_pg_dir
	adrp	x1, init_pg_end
	sub	x1, x1, x0
1:	stp	xzr, xzr, [x0], #16
	stp	xzr, xzr, [x0], #16
	stp	xzr, xzr, [x0], #16
	stp	xzr, xzr, [x0], #16
	subs	x1, x1, #64
	b.ne	1b

	mov_q	x7, SWAPPER_MM_MMUFLAGS

	/*
	 * Create the identity mapping.
	 */
	adrp	x0, idmap_pg_dir
	adrp	x3, __idmap_text_start		// __pa(__idmap_text_start)

#ifdef CONFIG_ARM64_VA_BITS_52
	mrs_s	x6, SYS_ID_AA64MMFR2_EL1
	and	x6, x6, #(0xf << ID_AA64MMFR2_LVA_SHIFT)
	mov	x5, #52
	cbnz	x6, 1f
#endif
	mov	x5, #VA_BITS_MIN
1:
	adr_l	x6, vabits_actual
	str	x5, [x6]
	dmb	sy
	dc	ivac, x6		// Invalidate potentially stale cache line

	/*
	 * VA_BITS may be too small to allow for an ID mapping to be created
	 * that covers system RAM if that is located sufficiently high in the
	 * physical address space. So for the ID map, use an extended virtual
	 * range in that case, and configure an additional translation level
	 * if needed.
	 *
	 * Calculate the maximum allowed value for TCR_EL1.T0SZ so that the
	 * entire ID map region can be mapped. As T0SZ == (64 - #bits used),
	 * this number conveniently equals the number of leading zeroes in
	 * the physical address of __idmap_text_end.
	 */
	adrp	x5, __idmap_text_end
	clz	x5, x5
	cmp	x5, TCR_T0SZ(VA_BITS_MIN) // default T0SZ small enough?
	b.ge	1f			// .. then skip VA range extension

	adr_l	x6, idmap_t0sz
	str	x5, [x6]
	dmb	sy
	dc	ivac, x6		// Invalidate potentially stale cache line

#if (VA_BITS < 48)
#define EXTRA_SHIFT	(PGDIR_SHIFT + PAGE_SHIFT - 3)
#define EXTRA_PTRS	(1 << (PHYS_MASK_SHIFT - EXTRA_SHIFT))

	/*
	 * If VA_BITS < 48, we have to configure an additional table level.
	 * First, we have to verify our assumption that the current value of
	 * VA_BITS was chosen such that all translation levels are fully
	 * utilised, and that lowering T0SZ will always result in an additional
	 * translation level to be configured.
	 */
#if VA_BITS != EXTRA_SHIFT
#error "Mismatch between VA_BITS and page size/number of translation levels"
#endif

	mov	x4, EXTRA_PTRS
	create_table_entry x0, x3, EXTRA_SHIFT, x4, x5, x6
#else
	/*
	 * If VA_BITS == 48, we don't have to configure an additional
	 * translation level, but the top-level table has more entries.
	 */
	mov	x4, #1 << (PHYS_MASK_SHIFT - PGDIR_SHIFT)
	str_l	x4, idmap_ptrs_per_pgd, x5
#endif
1:
	ldr_l	x4, idmap_ptrs_per_pgd
	adr_l	x6, __idmap_text_end		// __pa(__idmap_text_end)

	map_memory x0, x1, x3, x6, x7, x3, x4, x10, x11, x12, x13, x14

	/*
	 * Map the kernel image (starting with PHYS_OFFSET).
	 */
	adrp	x0, init_pg_dir
	mov_q	x5, KIMAGE_VADDR		// compile time __va(_text)
	add	x5, x5, x23			// add KASLR displacement
	mov	x4, PTRS_PER_PGD
	adrp	x6, _end			// runtime __pa(_end)
	adrp	x3, _text			// runtime __pa(_text)
	sub	x6, x6, x3			// _end - _text
	add	x6, x6, x5			// runtime __va(_end)

	map_memory x0, x1, x5, x6, x7, x3, x4, x10, x11, x12, x13, x14

	/*
	 * Since the page tables have been populated with non-cacheable
	 * accesses (MMU disabled), invalidate those tables again to
	 * remove any speculatively loaded cache lines.
	 */
	dmb	sy

	adrp	x0, idmap_pg_dir
	adrp	x1, idmap_pg_end
	bl	dcache_inval_poc

	adrp	x0, init_pg_dir
	adrp	x1, init_pg_end
	bl	dcache_inval_poc

	ret	x28
SYM_FUNC_END(__create_page_tables)
```

### 初始化CPU

代码位于<arch/arm64/mm/proc.S\>中：
```C
/*
 *	__cpu_setup
 *
 *	Initialise the processor for turning the MMU on.
 *
 * Output:
 *	Return in x0 the value of the SCTLR_EL1 register.
 */
	.pushsection ".idmap.text", "awx"
SYM_FUNC_START(__cpu_setup)
	tlbi	vmalle1				// Invalidate local TLB
	dsb	nsh

	mov	x1, #3 << 20
	msr	cpacr_el1, x1			// Enable FP/ASIMD
	mov	x1, #1 << 12			// Reset mdscr_el1 and disable
	msr	mdscr_el1, x1			// access to the DCC from EL0
	isb					// Unmask debug exceptions now,
	enable_dbg				// since this is per-cpu
	reset_pmuserenr_el0 x1			// Disable PMU access from EL0
	reset_amuserenr_el0 x1			// Disable AMU access from EL0

	/*
	 * Default values for VMSA control registers. These will be adjusted
	 * below depending on detected CPU features.
	 */
	mair	.req	x17
	tcr	.req	x16
	mov_q	mair, MAIR_EL1_SET
	mov_q	tcr, TCR_TxSZ(VA_BITS) | TCR_CACHE_FLAGS | TCR_SMP_FLAGS | \
			TCR_TG_FLAGS | TCR_KASLR_FLAGS | TCR_ASID16 | \
			TCR_TBI0 | TCR_A1 | TCR_KASAN_SW_FLAGS | TCR_MTE_FLAGS

	tcr_clear_errata_bits tcr, x9, x5

#ifdef CONFIG_ARM64_VA_BITS_52
	ldr_l		x9, vabits_actual
	sub		x9, xzr, x9
	add		x9, x9, #64
	tcr_set_t1sz	tcr, x9
#else
	ldr_l		x9, idmap_t0sz
#endif
	tcr_set_t0sz	tcr, x9

	/*
	 * Set the IPS bits in TCR_EL1.
	 */
	tcr_compute_pa_size tcr, #TCR_IPS_SHIFT, x5, x6
#ifdef CONFIG_ARM64_HW_AFDBM
	/*
	 * Enable hardware update of the Access Flags bit.
	 * Hardware dirty bit management is enabled later,
	 * via capabilities.
	 */
	mrs	x9, ID_AA64MMFR1_EL1
	and	x9, x9, #0xf
	cbz	x9, 1f
	orr	tcr, tcr, #TCR_HA		// hardware Access flag update
1:
#endif	/* CONFIG_ARM64_HW_AFDBM */
	msr	mair_el1, mair
	msr	tcr_el1, tcr
	/*
	 * Prepare SCTLR
	 */
	mov_q	x0, INIT_SCTLR_EL1_MMU_ON
	ret					// return to head.S

	.unreq	mair
	.unreq	tcr
SYM_FUNC_END(__cpu_setup)
```

### 开启MMU

```C
SYM_FUNC_START_LOCAL(__primary_switch)
#ifdef CONFIG_RANDOMIZE_BASE
	mov	x19, x0				// preserve new SCTLR_EL1 value
	mrs	x20, sctlr_el1			// preserve old SCTLR_EL1 value
#endif

	adrp	x1, init_pg_dir
	bl	__enable_mmu
#ifdef CONFIG_RELOCATABLE
#ifdef CONFIG_RELR
	mov	x24, #0				// no RELR displacement yet
#endif
	bl	__relocate_kernel
#ifdef CONFIG_RANDOMIZE_BASE
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	blr	x8

	/*
	 * If we return here, we have a KASLR displacement in x23 which we need
	 * to take into account by discarding the current kernel mapping and
	 * creating a new one.
	 */
	pre_disable_mmu_workaround
	msr	sctlr_el1, x20			// disable the MMU
	isb
	bl	__create_page_tables		// recreate kernel mapping

	tlbi	vmalle1				// Remove any stale TLB entries
	dsb	nsh
	isb

	set_sctlr_el1	x19			// re-enable the MMU

	bl	__relocate_kernel
#endif
#endif
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	br	x8						//跳转至__primary_switched
SYM_FUNC_END(__primary_switch)
```

__enable_mmu的代码如下：

```C
/*
 * Enable the MMU.
 *
 *  x0  = SCTLR_EL1 value for turning on the MMU.
 *  x1  = TTBR1_EL1 value
 *
 * Returns to the caller via x30/lr. This requires the caller to be covered
 * by the .idmap.text section.
 *
 * Checks if the selected granule size is supported by the CPU.
 * If it isn't, park the CPU
 */
SYM_FUNC_START(__enable_mmu)
	mrs	x2, ID_AA64MMFR0_EL1
	ubfx	x2, x2, #ID_AA64MMFR0_TGRAN_SHIFT, 4
	cmp     x2, #ID_AA64MMFR0_TGRAN_SUPPORTED_MIN
	b.lt    __no_granule_support
	cmp     x2, #ID_AA64MMFR0_TGRAN_SUPPORTED_MAX
	b.gt    __no_granule_support
	update_early_cpu_boot_status 0, x2, x3
	adrp	x2, idmap_pg_dir
	phys_to_ttbr x1, x1
	phys_to_ttbr x2, x2
	msr	ttbr0_el1, x2			// load TTBR0
	offset_ttbr1 x1, x3
	msr	ttbr1_el1, x1			// load TTBR1
	isb

	set_sctlr_el1	x0

	ret
SYM_FUNC_END(__enable_mmu)
```

在开启 MMU 之后，使用 bx 命令跳转至 __primary_switched，进行最后的栈设置和异常向量表设置，然后进入 start_kernel：

```C
/*
 * The following fragment of code is executed with the MMU enabled.
 *
 *   x0 = __PHYS_OFFSET
 */
SYM_FUNC_START_LOCAL(__primary_switched)
	adr_l	x4, init_task
	init_cpu_task x4, x5, x6

	adr_l	x8, vectors			// load VBAR_EL1 with virtual
	msr	vbar_el1, x8			// vector table address
	isb

	stp	x29, x30, [sp, #-16]!
	mov	x29, sp

	str_l	x21, __fdt_pointer, x5		// Save FDT pointer

	ldr_l	x4, kimage_vaddr		// Save the offset between
	sub	x4, x4, x0			// the kernel virtual and
	str_l	x4, kimage_voffset, x5		// physical mappings

	// Clear BSS
	adr_l	x0, __bss_start
	mov	x1, xzr
	adr_l	x2, __bss_stop
	sub	x2, x2, x0
	bl	__pi_memset
	dsb	ishst				// Make zero page visible to PTW

#if defined(CONFIG_KASAN_GENERIC) || defined(CONFIG_KASAN_SW_TAGS)
	bl	kasan_early_init
#endif
	mov	x0, x21				// pass FDT address in x0
	bl	early_fdt_map			// Try mapping the FDT early
	bl	init_feature_override		// Parse cpu feature overrides
#ifdef CONFIG_RANDOMIZE_BASE
	tst	x23, ~(MIN_KIMG_ALIGN - 1)	// already running randomized?
	b.ne	0f
	bl	kaslr_early_init		// parse FDT for KASLR options
	cbz	x0, 0f				// KASLR disabled? just proceed
	orr	x23, x23, x0			// record KASLR offset
	ldp	x29, x30, [sp], #16		// we must enable KASLR, return
	ret					// to __primary_switch()
0:
#endif
	bl	switch_to_vhe			// Prefer VHE if possible
	ldp	x29, x30, [sp], #16
	bl	start_kernel			//jump to kernel
	ASM_BUG()
SYM_FUNC_END(__primary_switched)
```

## 从设备树获取信息


### 识别机器类型

内核通过设备树中的信息来识别机器类型。理想情况下，特定的平台不应该影响内核，因为所有的平台细节都会完美地被设备树以一致且可靠的方式描述。但是硬件并不完美，因此内核必须在启动早期识别出具体的机型，这样就有机会执行机型特有的硬件修复代码。大多数时候，机型并不重要，内核会根据 SoC 来选择执行相应的启动代码。以 ARM 为例，<arch/arm/kernel/setup.c\>中的`setup_arch()`函数会调用位于<arch/arm/kernel/devtree.c\>中的`setup_machine_fdt()`函数。该函数会查找machine_desc表并选择与设备树最为匹配的machine_desc。通过比较设备树根节点的compatible属性和定义在<arch/arm/include/asm/mach/arch.h\>中的machine_desc数据结构的dt_compat列表来选择最佳匹配。

compatible属性包含了以确切机器名开始的一组有序字符串。以ARM为例，对于每一个machine_desc，内核会检查其dt_compat列表中的条目是否出现在compatible属性中。如果有的话，对应的machine_desc就会作为驱动机器的一个候选。

当machine_descs表被全部检索后，`setup_machine_fdt()`函数返回一个最佳匹配的machine_desc——依据machine_desc与compatible属性的哪一个条目相匹配来确定。如果没有匹配的machine_desc，那么该函数返回NULL。在选择了machine_desc之后，`setup_machine_fdt()`还负责早期的设备树扫描。


### 运行时配置

大多数情况下，设备树将作为u-boot与内核传递数据的唯一方法。所以内核参数、initrd镜像的位置等运行时配置项也可以通过设备树传递。这些数据一般存放在/chosen节点下。

bootargs属性包含了内核参数，initrd-开头的属性则定义了initrd数据块的起始地址和大小。在启动阶段的早期，分页机制还没有开启，`setup_machine_fdt()`借助不同的辅助函数调用`of_scan_flat_dt()`对设备树的数据进行扫描。`of_scan_flat_dt()`扫描整个设备树，利用传入的辅助函数来提取启动阶段早期所需要的信息。辅助函数`early_init_dt_scan_chosen()`主要用来解析包含内核启动参数的chosen节点，`early_init_dt_scan_root()`则负责初始化设备树的地址空间模型，`early_init_dt_scan_memory()`的调用决定了可用内存的位置和大小。


### 设备填充

在机型识别完成并且解析完早期的配置信息之后，内核的初始化就可以按常规方式继续了。在这个过程中的某个时间点，`unflatten_device_tree()`函数负责将设备树的数据转化为更为有效的运行时描述方式。在ARM平台上，这里也是机型特有的启用钩子（比如`init_early()`、`init_irq()`和`init_machine()`）被调用的地方。这些函数的目的可以从名字看出来。

在设备树上下文中最有趣的钩子函数当属`init_machine()`，主要负责根据平台相关的信息填充Linux设备模型。设备列表可以通过解析设备树获取，然后动态地为这些设备分配device数据结构。

## 启动内核

内核启动的入口函数是`start_kernel()`：

```C title="init/main.c"
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
	char *command_line;
	char *after_dashes;

	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	debug_objects_early_init();
	init_vmlinux_build_id();

	cgroup_init_early();

	local_irq_disable();
	early_boot_irqs_disabled = true;

	/*
	 * Interrupts are still disabled. Do necessary setups, then
	 * enable them.
	 */
	boot_cpu_init();
	page_address_init();
	pr_notice("%s", linux_banner);
	early_security_init();
	setup_arch(&command_line);
	setup_boot_config();
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */
	boot_cpu_hotplug_init();

	build_all_zonelists(NULL);
	page_alloc_init();

	pr_notice("Kernel command line: %s\n", saved_command_line);
	/* parameters may set static keys */
	jump_label_init();
	parse_early_param();
	after_dashes = parse_args("Booting kernel",
				  static_command_line, __start___param,
				  __stop___param - __start___param,
				  -1, -1, NULL, &unknown_bootoption);
	print_unknown_bootoptions();
	if (!IS_ERR_OR_NULL(after_dashes))
		parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
			   NULL, set_init_arg);
	if (extra_init_args)
		parse_args("Setting extra init args", extra_init_args,
			   NULL, 0, -1, -1, NULL, set_init_arg);

	/*
	 * These use large bootmem allocations and must precede
	 * kmem_cache_init()
	 */
	setup_log_buf(0);
	vfs_caches_init_early();
	sort_main_extable();
	trap_init();
	mm_init();

	ftrace_init();

	/* trace_printk can be enabled here */
	early_trace_init();

	/*
	 * Set up the scheduler prior starting any interrupts (such as the
	 * timer interrupt). Full topology setup happens at smp_init()
	 * time - but meanwhile we still have a functioning scheduler.
	 */
	sched_init();

	if (WARN(!irqs_disabled(),
		 "Interrupts were enabled *very* early, fixing it\n"))
		local_irq_disable();
	radix_tree_init();

	/*
	 * Set up housekeeping before setting up workqueues to allow the unbound
	 * workqueue to take non-housekeeping into account.
	 */
	housekeeping_init();

	/*
	 * Allow workqueue creation and work item queueing/cancelling
	 * early.  Work item execution depends on kthreads and starts after
	 * workqueue_init().
	 */
	workqueue_init_early();

	rcu_init();

	/* Trace events are available after this */
	trace_init();

	if (initcall_debug)
		initcall_debug_enable();

	context_tracking_init();
	/* init some links before init_ISA_irqs() */
	early_irq_init();
	init_IRQ();
	tick_init();
	rcu_init_nohz();
	init_timers();
	srcu_init();
	hrtimers_init();
	softirq_init();
	timekeeping_init();
	kfence_init();
	time_init();

	/*
	 * For best initial stack canary entropy, prepare it after:
	 * - setup_arch() for any UEFI RNG entropy and boot cmdline access
	 * - timekeeping_init() for ktime entropy used in random_init()
	 * - time_init() for making random_get_entropy() work on some platforms
	 * - random_init() to initialize the RNG from from early entropy sources
	 */
	random_init(command_line);
	boot_init_stack_canary();

	perf_event_init();
	profile_init();
	call_function_init();
	WARN(!irqs_disabled(), "Interrupts were enabled early\n");

	early_boot_irqs_disabled = false;
	local_irq_enable();

	kmem_cache_init_late();

	/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
	console_init();
	if (panic_later)
		panic("Too many boot %s vars at `%s'", panic_later,
		      panic_param);

	lockdep_init();

	/*
	 * Need to run this when irqs are enabled, because it wants
	 * to self-test [hard/soft]-irqs on/off lock inversion bugs
	 * too:
	 */
	locking_selftest();

	/*
	 * This needs to be called before any devices perform DMA
	 * operations that might use the SWIOTLB bounce buffers. It will
	 * mark the bounce buffers as decrypted so that their usage will
	 * not cause "plain-text" data to be decrypted when accessed.
	 */
	mem_encrypt_init();

#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start && !initrd_below_start_ok &&
	    page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
		pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
		    page_to_pfn(virt_to_page((void *)initrd_start)),
		    min_low_pfn);
		initrd_start = 0;
	}
#endif
	setup_per_cpu_pageset();
	numa_policy_init();
	acpi_early_init();
	if (late_time_init)
		late_time_init();
	sched_clock_init();
	calibrate_delay();
	pid_idr_init();
	anon_vma_init();

	thread_stack_cache_init();
	cred_init();
	fork_init();
	proc_caches_init();
	uts_ns_init();
	key_init();
	security_init();
	dbg_late_init();
	net_ns_init();
	vfs_caches_init();
	pagecache_init();
	signals_init();
	seq_file_init();
	proc_root_init();
	nsfs_init();
	cpuset_init();
	cgroup_init();
	taskstats_init_early();
	delayacct_init();

	poking_init();
	check_bugs();

	acpi_subsystem_init();
	arch_post_acpi_subsys_init();
	kcsan_init();

	/* Do the rest non-__init'ed, we're now alive */
	arch_call_rest_init();

	prevent_tail_call_optimization();
}
```

### setup_arch

`setup_arch()`负责根据平台相关的信息填充Linux设备模型，其接受一个command_line参数，记录了uboot传递给内核的信息，大小不能超过4096。

`setup_machine_fdt(__fdt_pointer)`：参数是dtb位于内存的地址。

此时MMU已经开启，需要将dtb位于内存的物理地址映射成虚拟地址。







