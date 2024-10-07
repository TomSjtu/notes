# 函数

Go 语言中函数的特点有：

- 无需声明原型
- 支持不定参数
- 支持多返回值
- 支持命名返回参数
- 支持匿名函数和闭包
- 不支持嵌套、重载、默认参数

符合规范的函数一般写成如下的形式：

```GO
func functionName(parameter_list) (return_value_list) {
    //body
}
```

其中：

- parameter_list的形式为(param1 type1, param2 type2, …)
- return_value_list的形式为(return1 type1, return2 type2, …)

只有当某个函数需要被外部包调用的时候才使用大写字母开头，并遵循 Pascal 命名法；否则就遵循骆驼命名法，即第一个单词的首字母小写，其余单词的首字母大写。

## 返回值

在GO中，函数可以返回多个返回值:

- 匿名方式返回

```GO
func foo(a string, b int)(int, int){
    return 111, 222
}
```

- 命名方式返回

```GO
func foo(a string, b int)(r1 int, r2 int){
    r1 = 111
    r2 = 222
    return
}
```

## 内置函数

Go 语言提供了一些内置函数，可以直接使用：

- `append()`：追加元素到数组、切片中
- `close()`：关闭 channel
- `delete()`：从 map 中删除 key 对应的 value
- `panic()`：触发异常，终止代码执行
- `recover()`：捕获`panic()`异常
- `imag()`：获取复数的虚部
- `real()`：获取复数的实部
- `make()`：创建切片、map、channel
- `new()`：分配值类型的内存
- `cap()`：获取切片、map 的容量
- `copy()`：复制切片
- `len()`：获取字符串、数组、切片、map、channel 的长度
- `println()`：打印到控制台

## 高级用法

函数可以作为参数：

```GO
func add(x, y int) int {
    return x + y
}

func calc(x, y int, op func(int, int) int) int {
    return op(x, y)
}


ret := calc(1, 2, add)
fmt.Println(ret)   // 3
```

函数也可以作为返回值：

```GO
func do(s string) (func(int, int) int, error) {
	switch s {
	case "+":
		return add, nil
	case "-":
		return sub, nil
	default:
		err := errors.New("无法识别的操作符")
		return nil, err
	}
}
```

## 匿名函数

匿名函数就是没有函数名的函数：

```GO
func(parameter_list)(return_value_list){
    //body
}
```

因为没有函数名，所以没法像普通函数那样调用，所以匿名函数需要立刻保存到某个变量中，或者作为立即执行的函数：

```GO
add := func(x, y int) int {
    return x + y
}

add(1, 2)   // 3
```

匿名函数也可以直接执行：

```GO
func(x, y int){
    fmt.Println(x + y)
}(1, 2)
```

## 闭包

所谓闭包就是一个函数，它可以访问外部作用域中的变量：

```GO
func calc(base int) (func(int) int, func(int) int){
    add := func(i int) int{
        base += i
        return base
    }

    sub := func(i int) int{
        base -= i
        return base
    }

    return add, sub
}

func main(){
    f1, f2 := calc(10)
    fmt.Println(f1(1))   // 11
    fmt.Println(f2(2))   // 8
}
```

在上述代码中，`calc()`函数返回两个匿名函数，分别用于加和减`base`变量，当我们用两个变量去接收返回值时，可以实现不同的行为。这就是闭包的魅力。

## 延迟调用

`defer`关键字用于延迟执行一些代码，直到包含`defer`语句的函数执行完毕，这常被用于释放资源等操作：

```GO
func main(){
    file, err := os.Open("file.txt")
    if err!= nil {
        log.Fatalln(err)
    }
    defer file.Close()
    // do something with file
}
```

在上述代码中，无论`main()`函数是否执行成功，`file.Close()`都会被调用，以确保文件被关闭。

`defer`语句可以有多个，按照栈的顺序执行：

```GO
defer func1()
defer func2()
defer func3()
```

则执行顺序为：`func3()` -> `func2()` -> `func1()`。

## 异常处理

Go 语言通过`panic()`抛出异常，然后在`defer`中通过`recover()`捕获异常：

```GO
func test(){
    defer func(){
        if err := recover(); err!= nil {
            println(err.(string))
        }
    }()

    panic("error")
}
```

