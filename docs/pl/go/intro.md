# 快速入门

Go语言版的"Hello World"程序如下所示：

```GO
package main

import "fmt"

func main(){
    /*第一个程序*/
    fmt.Println("Hello, World!")
}
```

使用`go run`执行.go程序，使用`go build`编译.go程序。

!!! warning "花括号的使用"

    Go语言中，{不能单独放一行，如果写成如下形式，则会报错：
    ```GO
    func main()
    {
        fmt.Println("Hello, World!")
    }
    ```


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





