# GNU汇编

## gcc编译流程

以一个简单的c语言程序为例：

```C title="test.c"
#include <stdio.h>

int data = 10;
int main(void)
{
    printf("%d\n",data);
    return 0;
}
```

gcc编译流程包括以下四个步骤：

1.预处理：gcc - E test.c -o test.i
2.编译：gcc -S test.i -o test.s
3.汇编：as test.s -o test.o
4.链接：ld test.o -o test -lc

汇编阶段生成的.o可重定位文件以及链接阶段生成的可执行文件都是ELF文件格式的一种。

ELF文件主要包含：

- ELF header：描述整个文件的基本属性 
- program header table：描述如何创建一个进程的内存映像
- 各个段
- section header table：描述段的信息

`readelf`命令可以用来查看一个ELF文件的组成。比如`readelf -h`用来读取ELF header，`readelf -S`用来读取section header table。

## 链接器

在现代软件工程中，一个大的项目往往由多个源文件组成，这些源文件经过预处理、编译、汇编后生成.o文件，然后通过链接器将这些.o文件链接成一个可执行文件。而链接脚本就告诉了链接器如何将目标文件组合起来。

以下是一段链接脚本的示例：

```C title="u-boot.lds"
#include <config.h>
#include <asm/psci.h>

OUTPUT_FORMAT("elf64-littleaarch64", "elf64-littleaarch64", "elf64-littleaarch64")
OUTPUT_ARCH(aarch64)
ENTRY(_start)
SECTIONS
{
	. = 0x00000000;

	. = ALIGN(8);
	.text :
	{
		*(.__image_copy_start)
		CPUDIR/start.o (.text*)
	}

	/* This needs to come before *(.text*) */
	.efi_runtime : {
                __efi_runtime_start = .;
		*(.text.efi_runtime*)
		*(.rodata.efi_runtime*)
		*(.data.efi_runtime*)
                __efi_runtime_stop = .;
	}

	.text_rest :
	{
		*(.text*)
	}

	. = ALIGN(8);
	.rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }

	. = ALIGN(8);
	.data : {
		*(.data*)
	}

	. = ALIGN(8);

	. = .;

	. = ALIGN(8);
	__u_boot_list : {
		KEEP(*(SORT(__u_boot_list*)));
	}

	. = ALIGN(8);

	.efi_runtime_rel : {
                __efi_runtime_rel_start = .;
		*(.rel*.efi_runtime)
		*(.rel*.efi_runtime.*)
                __efi_runtime_rel_stop = .;
	}

	. = ALIGN(8);

	.image_copy_end :
	{
		*(.__image_copy_end)
	}

	. = ALIGN(8);

	.rel_dyn_start :
	{
		*(.__rel_dyn_start)
	}

	.rela.dyn : {
		*(.rela*)
	}

	.rel_dyn_end :
	{
		*(.__rel_dyn_end)
	}

	_end = .;

	. = ALIGN(8);

	.bss_start : {
		KEEP(*(.__bss_start));
	}

	.bss : {
		*(.bss*)
		 . = ALIGN(8);
	}

	.bss_end : {
		KEEP(*(.__bss_end));
	}

	/DISCARD/ : { *(.dynsym) }
	/DISCARD/ : { *(.dynstr*) }
	/DISCARD/ : { *(.dynamic*) }
	/DISCARD/ : { *(.plt*) }
	/DISCARD/ : { *(.interp*) }
	/DISCARD/ : { *(.gnu*) }
}
```

该链接脚本主要做了几件事：

1. 指明输出二进制文件的格式和架构
2. 指明程序的入口点为_start
3. 设置程序的各个段，指明对齐方式
4. 丢弃一些不必要的段

## 伪指令

伪指令在汇编期间由汇编器处理，可以实现多种功能。

一些常用的伪指令比如：

- 对齐伪指令：.align
- 数据定义伪指令：.byte，.word
- 函数控制伪指令：.global，.ifeq expression
- 段伪指令：.section
- 宏伪指令：.macro



