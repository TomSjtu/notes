# GCC工具链

通常所说的GCC是GUN Compiler Collection的简称，是Linux系统上常用的编译工具。GCC工具链软件包括GNU C Compiler、Binutils、C运行库等。

Binutils是一组二进制处理工具，用于开发和调试，分别有如下工具：

- ar：用于创建静态库和动态库
- as：汇编器
- ld：链接器
- ldd：显示程序依赖的共享库
- objcopy：转换目标文件格式
- objdump：反汇编
- readelf：读取ELF文件信息
- size：显示符号表信息

GCC编译过程主要包含四个步骤：

1. 预处理：处理宏定义、条件编译指令，展开头文件，删除注释等
2. 编译：词法分析、语法分析、语义分析、生成汇编代码
3. 汇编：生成机器指令，生成.o目标文件
4. 链接：将目标文件链接成最后的可执行文件

## GNU C扩展语法

Linux内核使用GNU C编译器，它对标准C进行了一系列扩展和修改。

