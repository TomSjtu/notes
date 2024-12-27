# 标准库

## fmt

fmt 实现了格式化 I/O，用于控制输入输出。

### 向外输出

#### Print

`Print`系列函数用于将内容输出到系统的标准输出，`Printf`支持格式化输出，`Println`输出内容后自动换行：

```GO
func Print(a ...interface{})(n int, err error)
func Printf(format string, a ...interface{})(n int, err error)
func Println(a ...interface{})(n int, err error)
```

#### Fprint

`Fprint`系列函数将内容输出到一个 `io.Writer` 变量中：

```GO
func Fprint(w io.Writer, a ...interface{})(n int, err error)
func Fprintf(w io.Writer, format string, a ...interface{})(n int, err error)
func Fprintln(w io.Writer, a ...interface{})(n int, err error)
```

#### Sprint

`Sprint`系列函数将传入的数据存储在一个字符串中并返回：

```GO
func Sprint(a ...interface{})string
func Sprintf(format string, a ...interface{})string
func Sprintln(a ...interface{})string
```

#### Errorf

`Errorf`函数根据 format 参数生成格式化字符串并返回一个包含该字符串的错误：

```GO
func Errorf(format string, a ...interface{}) error
```

### 格式化占位符

常用的一些占位符如下：

| 占位符 | 说明 |
| --- | --- |
| %v | 值的默认格式 |
| %+v | 输出结构体时包括字段名 |
| %#v | 值的 Go 语法表示 |
| %T | 值的类型 |
| %% | 百分号 |
| %t | 布尔值 |
| %d | 十进制整型 |
| %x | 十六进制整型（小写字母） |
| %e | 科学计数法 |
| %f | 浮点数，默认 6 位 |
| %g | 自动选择 %e 或 %f |
| %s | 字符串 |
| %p | 十六进制地址 |

### 获取输入

#### Scan

`Scan`从标准输入扫描文本，读取由空白符分隔的值保存到传递给本函数的参数中，换行符视为空白符：

```GO
func Scan(a ...interface{})(n int, err error)
```

#### Scanf

`Scanf`从标准输入扫描文本，根据 format 参数指定的格式读取值并保存到传递给本函数的参数中：

```GO
func Scanf(format string, a ...interface{})(n int, err error)
```

#### Scanln

`Scanln`类似于`Scan`，但遇到换行时才停止扫描：

```GO
func Scanln(a ...interface{})(n int, err error)
```

#### Fscan

`Fscan`从一个`io.Reader`变量中读取数据：

```GO
func Fscan(r io.Reader, a ...interface{})(n int, err error)
func Fscanln(r io.Reader, a ...interface{})(n int, err error)
func Fscanf(r io.Reader, format string, a ...interface{})(n int, err error)
```

#### Sscan

`Sscan`从一个字符串中读取数据：

```GO
func Sscan(str string, a ...interface{})(n int, err error)
func Sscanln(str string, a ...interface{})(n int, err error)
func Sscanf(str string, format string, a ...interface{})(n int, err error)
```

## time

time 包提供了时间和日期相关的功能。

Go 语言中用`time.Time`类型来表示时间，可以通过`time.Now()`来获取当前时间：

```GO
// timeDemo 时间对象的年月日时分秒
func timeDemo() {
	now := time.Now() // 获取当前时间
	fmt.Printf("current time:%v\n", now)

	year := now.Year()     // 年
	month := now.Month()   // 月
	day := now.Day()       // 日
	hour := now.Hour()     // 小时
	minute := now.Minute() // 分钟
	second := now.Second() // 秒
	fmt.Println(year, month, day, hour, minute, second)
}
```

### 时区

时区信息可以通过`time.LoadLocation`函数加载：

```GO
loc, err := time.LoadLocation("Asia/Shanghai")
```

在获取了时区之后，就可以创建一个带有该时区信息的`time.Time`对象：

```GO
now := time.Now().In(loc)
```

### Unix Time

Unix Time 是从 1970-01-01 00:00:00 UTC 经过的秒数，可以通过`time.Time.Unix()`方法获取时间戳：

```GO
now := time.Now()
unixTime := now.Unix()
```

### 时间间隔

`time.Duration`类型表示时间间隔，它代表两个时间点的差值，以纳秒为单位。time 包中定义的时间间隔类型常量有：

```GO
const (
    Nanosecond  Duration = 1
    Microsecond = 1000 * Nanosecond
    Millisecond = 1000 * Microsecond
    Second      = 1000 * Millisecond
    Minute      = 60 * Second
    Hour        = 60 * Minute
)
```

### 时间操作

time 包中提供了一些时间操作函数，如：

```GO
func (t Time) Add(d Duration) Time
func (t Time) Sub(u Time) Duration
func (t Time) Equal(u Time) bool
func (t Time) Before(u Time) bool
func (t Time) After(u Time) bool
```

### 定时器

time.Timer 是一个定时器，它会在指定的时间后发送一个信号到其通道。如果定时器被重置，它会停止当前的计时并重新开始一个新的计时周期。

```GO
timer := time.NewTimer(time.Second * 5) // 5秒后发送信号
<- timer.C // 等待信号
// do something here
```

time.Ticker 用于定期发送事件，它每隔一定时间就会向其通道发送当前时间：

```GO
ticker := time.NewTicker(time.Second * 5) // 每隔5秒发送一次时间
```

## flag

flag 包实现了对命令行参数的解析，支持的参数类型有`bool`、`int`、`int64`、`uint`、`uint64`、`float`、`float64`、`string`、`duration`。

有以下两种常用的定义命令行参数的方式：

1. `flag.Type(flag名, 默认值, 帮助信息)`

   ```GO
   name := flag.String("name", "defaultName", "your name")
   ```

2. `flag.TypeVar(flag.Value, flag名, 默认值, 帮助信息)`

   ```GO
   var name string
   flag.StringVar(&name, "name", "defaultName", "your name")
   ```

通过以上两种方式定义好命令行参数后，就可以通过调用`flag.Parse()`来解析。支持的命令行参数有：

- -flag xxx
- --flag xxx
- -flag=xxx
- --flag=xxx

其中，布尔类型的参数必须使用等号方式指定。

flag 包还提供了一些辅助函数，如：

```GO
flag.Args()  ////返回命令行参数后的其他参数，以[]string类型
flag.NArg()  //返回命令行参数后的其他参数个数
flag.NFlag() //返回使用的命令行参数个数
```

## log

Go 语言内置的 log 包实现了简单的日志服务。直接使用 Print 系列、Fatal 系列、Panic 系列函数就可以输出日志至标准 logger。标准 logger 默认只会提供日志的时间信息，但可以通过`log.SetFlags`函数设置日志的输出格式。flag 选项有：

```GO
const (
    Ldate         = 1 << iota     // 日期：2024/01/01
    Ltime                         // 时间：12:00:00
    Lmicroseconds                 // 微秒：12:00:00.123123
    Llongfile                     // 文件名和行号：/home/user/main.go:123
    Lshortfile                    // 文件名：main.go:123
    LUTC                         // 使用 UTC 时间
    LstdFlags = Ldate | Ltime    // 标准日志格式
)
```

除此之外，`log.SetPrefix()`函数用于设置日志的前缀，`log.SetOutput()`函数用于设置日志的输出目的地，`log.New()`函数用于创建一个新的 logger 对象。

## 文件

Go 语言中使用`os.Open()`函数打开一个文件，返回一个`*os.File`类型的文件对象。文件对象提供了一些方法，如`Read`、`Write`、`Close`等。

### 读取文件

`file.Read()`函数用于从文件中读取数据，返回读取的字节数和错误信息。该函数是操作系统级别的读取操作。它直接从文件描述符读取数据到一个缓冲区中。这个方法是阻塞式的，直到它读到了数据或遇到错误（包括到达文件末尾）。这是最基础的读取方式，提供了对读取行为的精细控制，但使用起来可能比较繁琐，尤其是在处理文本文件时：

```GO title="循环读取"
func main() {
	// 只读方式打开当前目录下的main.go文件
	file, err := os.Open("./main.go")
	if err != nil {
		fmt.Println("open file failed!, err:", err)
		return
	}
	defer file.Close()
	// 循环读取文件
	var content []byte
	var tmp = make([]byte, 128)
	for {
		n, err := file.Read(tmp)
		if err == io.EOF {
			fmt.Println("文件读完了")
			break
		}
		if err != nil {
			fmt.Println("read file failed, err:", err)
			return
		}
		content = append(content, tmp[:n]...)
	}
	fmt.Println(string(content))
}
```

bufio 包基于 file 包实现了带缓冲机制的读取操作，可以减少系统调用的次数，提高读取效率。

### 写入文件

`os.OpenFile()`函数用于打开一个文件，模式有以下几种：

| 模式 | 含义 |
| ---- | ---- |
| os.O_WRONLY | 只写 |
| os.O_CREATE | 创建文件 |
| os.O_RDONLY | 只读 |
| os.O_RDWR | 读写 |
| os.O_APPEND | 追加 |
| os.O_TRUNC | 清空 |

## context

contex 包提供了用于管理上下文和取消操作的机制，它定义了 Context 类型，专门用来简化对于处理单个请求的多个 goroutine 之间与请求域的数据、取消信号、截止时间等信息的传递、处理和取消操作。

对于服务器传入的请求应该创建上下文，而传出请求应该接受上下文。当一个上下文被取消时，它派生的所有上下文也应该一并被取消。

### context接口

`context.Context`是一个接口，定了四个需要实现的方法：

```GO
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

- `Deadline()`：返回当前上下文的截止时间，如果未设置截止时间，则返回 ok 为 false。
- `Done()`：返回一个 channel，当上下文被取消时，该 channel 会被关闭。
- `Err()`：返回取消上下文的原因。
- `Value()`：返回与上下文关联的值。

Go 内置的`Background()`是一个不可取消，没有设置截止时间，没有携带任何值的 Context，通常作为根 Context 衍生出更多的子上下文对象。

### with系列函数

- `WithCancel(parent Context) (ctx Context, cancel CancelFunc)`：返回一个可以手动取消的 Context 和一个 CancelFunc，调用 CancelFunc 会取消 Context。

	```GO
	package main

	import (
		"context"
		"fmt"
		"time"
	)

	func main() {
		// 创建一个父上下文
		parentCtx := context.Background()

		// 使用 WithCancel 创建一个可取消的上下文
		ctx, cancel := context.WithCancel(parentCtx)

		// 启动一个 goroutine 来执行任务
		go func(ctx context.Context) {
			for {
				select {
				case <-ctx.Done(): // 监听上下文的取消信号
					fmt.Println("任务被取消:", ctx.Err())
					return
				default:
					fmt.Println("任务正在执行...")
					time.Sleep(500 * time.Millisecond)
				}
			}
		}(ctx)

		// 模拟任务执行一段时间后取消
		time.Sleep(2 * time.Second)
		cancel() // 手动取消上下文
		time.Sleep(1 * time.Second) // 等待 goroutine 结束
	}
	```

- `WithDeadline(parent Context, deadline time.Time) (ctx Context, cancelCancelFunc)`：返回一个带有截止时间的 Context，当截止时间到达时，该 Context 会被取消。

	```GO
	package main

	import (
		"context"
		"fmt"
		"time"
	)

	func main() {
		// 创建一个父上下文
		parentCtx := context.Background()

		// 使用 WithDeadline 创建一个带有截止时间的上下文
		deadline := time.Now().Add(2 * time.Second) // 2秒后截止
		ctx, cancel := context.WithDeadline(parentCtx, deadline)
		defer cancel() // 确保在函数结束时调用 cancel

		// 启动一个 goroutine 来执行任务
		go func(ctx context.Context) {
			for {
				select {
				case <-ctx.Done(): // 监听上下文的取消信号
					fmt.Println("任务被取消:", ctx.Err())
					return
				default:
					fmt.Println("任务正在执行...")
					time.Sleep(500 * time.Millisecond)
				}
			}
		}(ctx)

		// 等待任务完成或超时
		time.Sleep(3 * time.Second)
	}
	```

- `WithTimeout(parent Context, timeout time.Duration) (ctx Context, cancel CancelFunc)`：返回一个带有超时时间的 Context，当超时时间到达时，该 Context 会被取消。

	```GO
	package main

	import (
		"context"
		"fmt"
		"time"
	)

	func main() {
		// 创建一个父上下文
		parentCtx := context.Background()

		// 使用 WithTimeout 创建一个带有超时时间的上下文
		ctx, cancel := context.WithTimeout(parentCtx, 2*time.Second)
		defer cancel() // 确保在函数结束时调用 cancel

		// 启动一个 goroutine 来执行任务
		go func(ctx context.Context) {
			for {
				select {
				case <-ctx.Done(): // 监听上下文的取消信号
					fmt.Println("任务被取消:", ctx.Err())
					return
				default:
					fmt.Println("任务正在执行...")
					time.Sleep(500 * time.Millisecond)
				}
			}
		}(ctx)

		// 等待任务完成或超时
		time.Sleep(3 * time.Second)
	}
	```

- `WithValue(parent Context, key, val interface{})`：返回一个带有键值对的 Context，可以通过 Context 的 Value 方法获取该键值对。

	```GO
	package main

	import (
		"context"
		"fmt"
	)

	func main() {
		// 创建一个父上下文
		parentCtx := context.Background()

		// 使用 WithValue 创建一个带有键值对的上下文
		ctx := context.WithValue(parentCtx, "userID", 12345)

		// 在函数中获取上下文中的值
		userID := ctx.Value("userID").(int)
		fmt.Println("用户ID:", userID)

		// 如果键不存在，返回 nil
		unknownValue := ctx.Value("unknownKey")
		fmt.Println("未知键的值:", unknownValue)
	}
	```