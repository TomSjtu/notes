# 接口

接口(interface)定义了一个对象的行为规范，它的定义如下：

```GO
type interfaceName interface {
    method1(params) result1
    method2(params) result2
    ...
}
```

在 Go 语言中，接口只负责定义方法的签名，但不关心具体的实现。比如假设有一个接口定义如下：

```GO
type Shape interface {
    Area() float64
}
```

我们可以定义一个圆形的结构体，并实现`Area()`方法：

```GO
type Circle struct {
    radius float64
}

func (c Circle) Area() float64 {
    return 3.14 * c.radius * c.radius
}
```

现在，我们可以创建一个`Shape`接口的变量，并将`Circle`结构体赋值给它：

```GO
var s Shape = Circle{radius: 5}
```

然后，我们就可以调用`Area()`方法来获取圆的面积：

```GO
area := s.Area()
fmt.Println("The area of the circle is", area)
```

## error接口

Go 语言中把错误当成一种特殊的值来处理，用`error`接口来表示：

```GO
type error interface {
    Error() string
}
```

`error`接口之定义了一个`Error()`方法，返回一个字符串，表示错误信息。当一个函数或方法需要返回错误时，我们通常是把错误作为最后一个返回值，例如标准库中的`Open()`函数：

```GO
func Open(name string) (*File, error){
    return OpenFile(name, O_RDONLY, 0)
}
```

返回值`error`是一个接口类型，默认为`nil`。所以我们通常将调用函数返回的错误与`nil`进行比较，来判断是否有错误发生：

```GO
file, err := Open("test.txt")
if err!= nil {
    fmt.Println("打开文件失败，Error:", err)
} else {
    // do something with file
}
```

我们可以根据需求自定义`error`：

```GO
type MyError struct {
    message string
}

func (e MyError) Error() string {
    return e.message
}
```

然后，我们可以返回一个`MyError`类型的错误：

```GO
func myFunc() error {
    return MyError{message: "something went wrong"}
}
```

这样，调用者就可以通过`error`接口来处理错误：

```GO
if err := myFunc(); err != nil {
    fmt.Println(err.Error())
}
```

## 空接口

空接口是指没有定义任何方法的接口类型。因此任何类型都可以视为实现了空接口。也正是因为空接口类型的这个特性，空接口类型的变量可以存储任意类型的值。

使用空接口可以实现接收任意类型的函数参数：

```GO
func printValue(v interface{}) {
    fmt.Println(v)
}

printValue(123)
printValue("hello")
printValue(true)
```

既然空接口可以接收任意类型的值，那么我们如何判断具体传入的值呢？我们可以使用类型断言来进行判断：

```GO
func printValue(v interfaceP{}){
    a, ok := v.(int)
    if ok {
        fmt.Println("int value:", a)
    }else {
        fmt.Println("not an int value")
    }
}
```

