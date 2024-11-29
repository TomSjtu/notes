# 快速入门

Go 语言版的"Hello World"程序如下所示：

```GO
package main

import "fmt"

func main(){
    fmt.Println("Hello, World!")
}
```

使用`go run`执行 .go 程序，使用`go build`编译 .go 程序。

## 特性

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

## 环境变量

使用`go env`命令可以查看 Go 语言的环境变量，其中常用的有：

- GOARCH：目标系统的体系结构
- GOPATH：Go 工作目录
- GOPROXY：Go 安装模块的路径
- GOOS：目标系统的架构(用于交叉编译)

由于网络原因，为了正确安装模块，一般会使用`go env -w GOPROXY=https://goproxy.cn,direct`命令设置代理。

## 命令行工具

Go 语言的命令行工具有：

```SHELL
$ go
Go is a tool for managing Go source code.

Usage:

        go <command> [arguments]

The commands are:

        bug         start a bug report
        build       compile packages and dependencies
        clean       remove object files and cached files
        doc         show documentation for package or symbol
        env         print Go environment information
        fix         update packages to use new APIs
        fmt         gofmt (reformat) package sources
        generate    generate Go files by processing source
        get         add dependencies to current module and install them
        install     compile and install packages and dependencies
        list        list packages or modules
        mod         module maintenance
        work        workspace maintenance
        run         compile and run Go program
        test        test packages
        tool        run specified go tool
        version     print Go version
        vet         report likely mistakes in packages

Use "go help <command>" for more information about a command.

Additional help topics:

        buildconstraint build constraints
        buildmode       build modes
        c               calling between Go and C
        cache           build and test caching
        environment     environment variables
        filetype        file types
        go.mod          the go.mod file
        gopath          GOPATH environment variable
        goproxy         module proxy protocol
        importpath      import path syntax
        modules         modules, module versions, and more
        module-auth     module authentication using go.sum
        packages        package lists and patterns
        private         configuration for downloading non-public code
        testflag        testing flags
        testfunc        testing functions
        vcs             controlling version control with GOVCS

Use "go help <topic>" for more information about that topic.
```

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

在`const()`中声明`iota`，它默认开始值为0，const 中每新增一行常量声明将使`iota`计数一次，如下：

```GO
const (
    i = iota + 1    // iota = 0， i = 1
    j = iota + 2    // iota = 1， j = 2
    k = iota * 2    // iota = 2， k = 4
)
```

## 数据类型

Go 语言支持以下数据类型：

- bool
- int
- float
- complex
- string
- rune (Unicode 码点)
- byte (uint8 的别名)
- pointer
- array
- slice
- struct
- map
- function
- interface
- channel

注意：基本类型和数组属于值传递，map、slice、interface、channel 属于引用传递。

## 数组与切片

Go 语言中数组大小是确定的，用以下方式来声明：

```GO
var arrayName [size]dataType
```

初始化：

```GO
var numbers = [3]int{1, 2, 3}
```

切片为动态数组，引用类型，它可以从数组中生成：

```GO
a := [5]int{1, 2, 3, 4, 5}
s := a[1:4]
```

也可以动态创建：

```GO
slice := make([]dataType, len, cap)
```

`len()`函数获取切片的长度，`cap()`函数获取切片的容量。

## 字典

字典类型是无序的键值对集合，用以下方式声明：

```GO
dictName := make(map[keyType]valueType)
```

判断某个键是否存在：

```GO
value, ok := dictName[key]

if ok{
    // 键存在
}else{
    // 键不存在
}
```

遍历：

```GO
for key, value := range dictName{
    // 遍历字典
}
```

删除键值对：

```GO
delete(dictName, key)
```

## 结构体

Go 语言使用`type`关键字来定义结构体：

```GO
type structName struct {
    filed1 dataType1
    filed2 dataType2
   ...
}
```

结构体中字段大写开头表示可公开访问，小写表示私有（仅在定义当前结构体的包中可访问）。结构体初始化可以采用键值对初始化、列表初始化。

Go 语言中，在结构体中嵌套另一个结构体可以直接实现继承。如果该结构体与嵌套结构体的方法重名，则会覆盖嵌套结构体的方法：

```GO
type Animal struct {
	Name string
}

func (a *Animal) Speak() {
	println(a.Name, "says meow!")
}

func (a *Animal) Move() {
	println(a.Name, "is running!")
}

type Dog struct {
	Animal
}

func (d *Dog) Speak() {
	println(d.Name, "says woof!")
}

func main() {
	a := Animal{Name: "Bob"}
	d := Dog{Animal: a}
	d.Move()    // 输出: Bob is running!
	d.Speak()   // 输出: Bob says woof!
}
```

但是，如果处于同一层级的多个嵌入字段拥有同名的字段或方法，则会触发一个编译错误，因为编译器无法确定应该选择哪个成员。

Go 语言可以对结构体打标签(tag)，使得结构体可以被序列化和反序列化。

```GO
type Person struct {
    Name string `json:"name" xml:"name"`
    Age  int    `json:"age" xml:"age"`
    City string `json:"city" xml:"city"`
}
```


使用结构体标签进行编码：

```GO
import (
    "encoding/json"
    "fmt"
)

p := Person{Name: "Alice", Age: 30, City: "New York"}

// 将结构体编码为JSON
data, err := json.Marshal(p)
if err != nil {
    fmt.Println("Error encoding JSON:", err)
    return
}

fmt.Println(string(data)) // 输出: {"name":"Alice","age":30,"city":"New York"}
```

使用结构体标签进行解码：

```GO
import (
    "encoding/json"
    "fmt"
)

jsonStr := `{"name":"Bob","age":25,"city":"Los Angeles"}`

var p Person
err := json.Unmarshal([]byte(jsonStr), &p)
if err != nil {
    fmt.Println("Error decoding JSON:", err)
    return
}

fmt.Println(p) // 输出: {Bob 25 Los Angeles}
```

## 方法和接收者

Go 语言中的方法是一种作用于特定类型变量的函数，这种变量叫做接收者：

```GO
func (receiverVar dataType) funcName(parameterList) (returnList){
    // 函数体
}
```

举例说明：

```GO
type Person struct {
    name string
    age int
}

func(p Person) sayHello(){
    fmt.Println("Hello, my name is", p.name)
}
```

可以使用指针类型的接收者来修改结构体变量：

```GO
func (p *Person) setAge(age int){
    p.age = age
}
```

## 类型转换

GO 语言不存在隐式类型转换，所有的转换都必须显式说明，比如：

```GO
a := 5.0
b := int(a)
```

但这只能在定义正确的情况下转换成功，例如从一个取值范围较小的类型转换到一个取值范围较大的类型。当编译器捕捉到非法的类型转换时会引发编译时错误，否则将引发运行时错误。

## 流程控制

Go 语言中的流程控制语句如下：

1. 条件语句
    - if：基本的条件判断语句
    - switch：多条件判断语句，Go 语言中的 switch 不需要显示添加 break，每个 case 默认自带 break。
    ```GO
    if condition {

    } else if condition {

    } else {

    }

    swtich variable {
        case value1:

        case value2:
        
        default:
    }
    ```

2. 循环语句

    - for：唯一的循环语句

    ```GO
    for init; condition; post{

    }
    ```

3. 选择语句

    - select：用于处理通道
     
    ```GO
    select {
        case <-chan1:
            // 从chan1接收数据
        case chan2 <- data:
            // 向chan2发送数据
        default:
            // 如果没有任何case可以处理，则执行default语句
    }
    ```






