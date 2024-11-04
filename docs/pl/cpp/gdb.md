# GDB

GDB 是强大的代码调试工具，编译时需要设置`-g`选项，使得编译器生成调试信息。可以直接使用 gdb a.out 来调试一个可执行文件，也可以通过 gdb attach pid 来将 GDB 附加到正在运行的程序上。

## 常用命令

GDB 的命令有很多，这里列出一些常用的，足以应付一般的调试任务：

- run：运行程序，直到遇到断点
- list：显示源代码
- break：设置断点
- next：单步执行(跳过函数)
- step：单步执行(进入函数)
- backtrace：显示函数调用栈
- set：设置变量的值
- frame：切换函数调用栈
- info：显示局部变量的值
- finish：结束当前函数
- until：退出循环体
- delete：删除断点
- continue：继续执行到下一个断点处或程序结束
- print：打印变量的值
- watch：监视某个变量
- quit：退出 GDB