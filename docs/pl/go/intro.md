# 快速入门

Go语言版的"Hello World"程序如下所示：

```GO
package main

import "fmt"

func main(){
    fmt.Println("Hello, World!")
}
```

使用`go run`执行.go程序，使用`go build`编译.go程序。

## 缺失特性

许多能够在大多数面向对象语言中使用的特性 Go 语言都没有支持，但其中的一部分可能会在未来被支持。

- 为了简化设计，不支持函数重载和操作符重载
- 为了避免在 C/C++ 开发中的一些 Bug 和混乱，不支持隐式转换
- Go 语言通过另一种途径实现面向对象设计来放弃类和类型的继承
- 尽管在接口的使用方面可以实现类似变体类型的功能，但本身不支持变体类型
- 不支持动态加载代码
- 不支持动态链接库
- 不支持泛型
- 通过 `recover()` 和 `panic()` 来替代异常机制
- 不支持静态变量


## 格式化字符串

Go使用`fmt.Sprintf()`来格式化字符串，格式：`fmt.Sprintf(格式化样式，参数列表...)`。

```GO
func main(){
    var stockcode = "000987"
    var enddate = "2020-12-31"
    var url = "Code=%s&endDate=%s"
    var target_url = fmt.Sprintf(url, stockcode, enddate)
    fmt.Println(target_url)
    //结果为Code=000987&endDate=2020-12-31
}
```

## 环境变量

- $GOROOT：Go的安装位置
- $GOARCH：目标机器架构
- $GOOS：目标及其的操作系统

为了区分本地机器和目标机器，你可以使用 $GOHOSTOS 和 $GOHOSTARCH 设置本地机器的操作系统名称和编译体系结构，这两个变量只有在进行交叉编译的时候才会用到，如果你不进行显示设置，他们的值会和本地机器（$GOOS 和 $GOARCH）一样。

## 包

每个程序都是一个包，必须在源文件中非注释的第一行指明这个文件属于哪个包，如：`package main`。`package main` 表示一个可独立执行的程序，每个Go应用程序都包含一个名为`main`的包。

一个应用程序可以包含不同的包，而且即使你只使用`main`包也不必把所有的代码都写在一个巨大的文件里：你可以用一些较小的文件，并且在每个文件非注释的第一行都使用`package main`来指明这些文件都属于`main`包。

属于同一个包的源文件必须全部被一起编译，一个包即是编译时的一个单元，因此根据惯例，每个目录都只包含一个包。

如果对一个包进行更改或重新编译，所有引用了这个包的客户端程序都必须全部重新编译。

GO程序的执行（程序启动）顺序如下：

- 按顺序导入所有被`main`包引用的其它包，然后在每个包中执行如下流程：
- 如果该包又导入了其它的包，则从第一步开始递归执行，但是每个包只会被导入一次。
- 然后以相反的顺序在每个包中初始化常量和变量，如果该包含有`init()`函数的话，则调用该函数。
- 在完成这一切之后，`main`也执行同样的过程，最后调用`main()`函数开始执行程序。

## 类型转换

GO语言不存在隐式类型转换，所有的转换都必须显式说明，比如：

```GO
a := 5.0
b := int(a)
```

但这只能在定义正确的情况下转换成功，例如从一个取值范围较小的类型转换到一个取值范围较大的类型。当编译器捕捉到非法的类型转换时会引发编译时错误，否则将引发运行时错误。

## 常量

常量用`const`表示，它的值必须在编译期被确定。

在 Go 语言中，你可以省略类型说明符 [type]，因为编译器可以根据变量的值来推断其类型：

- 显式类型定义： `const b string = "abc"`
- 隐式类型定义： `const b = "abc"`

`const`可以用来定义枚举类型：

```GO
const (
    BEIJING     // 0
    SHANGHAI    // 1
    SHENZHEN    // 2
)
```

在`const()`中声明`iota`，它默认开始值为0，const中每新增一行常量声明将使iota计数一次，如下：

```GO
const (
    i = iota + 1    // iota = 0， i = 1
    j = iota + 2    // iota = 1， j = 2
    k = iota * 2    // iota = 2， k = 4
)
```
