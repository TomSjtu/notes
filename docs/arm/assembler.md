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

## 伪指令

伪指令在汇编期间由汇编器处理，可以实现多种功能。

一些常用的伪指令比如：

- 对齐伪指令：.align
- 数据定义伪指令：.byte，.word
- 函数控制伪指令：.global，.ifeq expression
- 段伪指令：.section
- 宏伪指令：.macro



