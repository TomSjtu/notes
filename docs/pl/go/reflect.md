# 反射

反射是指在程序运行期间对程序本身进行访问和修改的能力。Go 语言通过 reflect 包访问程序的反射信息。

在 Go 语言的反射机制中，任何接口值都是由一个具体类型和类型的值两部分组成的。reflect 包提供了`reflect.TypeOf`来获取接口的类型，`reflect.ValueOf`来获取类型的值。

```GO
func reflectType(x interface{}){
    v := reflect.TypeOf(x)
    fmt.Println("Type:", v)
}

func main(){
    var a float32 = 3.14
    reflectType(a) // Type: float32
    var b int = 10
    reflectType(b) // Type: int
}
```

## name和kind

在 Go 语言中，我们可以用`type`关键字定义自定义类型，而`kind`就是用来描述底层的类型。Go 语言的反射中像数组、切片、map、指针等类型的变量，它们的`.Name()`都是返回空。


```GO
package main

import (
	"fmt"
	"reflect"
)

type myInt int64

func reflectType(x interface{}) {
	t := reflect.TypeOf(x)
	fmt.Printf("type:%v kind:%v\n", t.Name(), t.Kind())
}

func main() {
	var a *float32 // 指针
	var b myInt    // 自定义类型
	var c rune     // 类型别名
	reflectType(a) // type: kind:ptr
	reflectType(b) // type:myInt kind:int64
	reflectType(c) // type:int32 kind:int32

	type person struct {
		name string
		age  int
	}
	type book struct{ title string }
	var d = person{
		name: "沙河小王子",
		age:  18,
	}
	var e = book{title: "《跟小王子学Go语言》"}
	reflectType(d) // type:person kind:struct
	reflectType(e) // type:book kind:struct
}
```

reflect 包中定义的`kind`常量如下：

```GO
type Kind uint
const (
    Invalid Kind = iota  // 非法类型
    Bool                 // 布尔型
    Int                  // 有符号整型
    Int8                 // 有符号8位整型
    Int16                // 有符号16位整型
    Int32                // 有符号32位整型
    Int64                // 有符号64位整型
    Uint                 // 无符号整型
    Uint8                // 无符号8位整型
    Uint16               // 无符号16位整型
    Uint32               // 无符号32位整型
    Uint64               // 无符号64位整型
    Uintptr              // 指针
    Float32              // 单精度浮点数
    Float64              // 双精度浮点数
    Complex64            // 64位复数类型
    Complex128           // 128位复数类型
    Array                // 数组
    Chan                 // 通道
    Func                 // 函数
    Interface            // 接口
    Map                  // 映射
    Ptr                  // 指针
    Slice                // 切片
    String               // 字符串
    Struct               // 结构体
    UnsafePointer        // 底层指针
)
```

## ValueOf

`reflect.Valueof()`返回的是`reflect.Value`类型，其中包含了原始值的信息，可以相互转换。`reflect.Value`类型提供了很多方法来操作原始值：

- `Bool()`：返回`bool`类型的值
- `Int()`：返回`int64`类型的值
- `Uint()`：返回`uint64`类型的值
- `Float()`：返回`float64`类型的值
- `Bytes()`：返回`[]byte`类型的值
- `String()`：返回`string`类型的值
- `Interface()`：返回`interface{}`类型的值

### 修改值

在反射中修改值，需要先获取`reflect.Value`类型的值，然后调用`Elem()`方法获取底层指针，然后才可以修改：


```GO
func SetValue(x interface{}){
    v := reflect.ValueOf(x)
    //获取底层指针
    if v.Elem().Kind() == reflect.Int64 {
        v.Elem().SetInt(100)
    }
}

func main(){
    var a int64 = 10
    //必须传递变量地址
    SetValue(&a)
}
```

### IsNil和IsValid

`IsNil()`常用于判断指针是否为空，`IsValid()`常用于判断返回值是否有效。


## 结构体反射

如果`reflect.TypeOf()`获取的类型是结构体，可以通过一些方法来获取结构体的详细信息：

| 方法 | 说明 |
| --- | --- |
| Field(i int) StructField | 根据索引，返回索引对应的结构体字段的信息。 |
| NumField() int | 返回结构体成员字段数量。 |
| FieldByName(name string) (StructField, bool) | 根据给定字符串返回字符串对应的结构体字段的信息。 |
| FieldByIndex(index []int) StructField | 多层成员访问时，根据 []int 提供的每个结构体的字段索引，返回字段的信息。 |
| FieldByNameFunc(match func(string) bool) (StructField,bool) | 根据传入的匹配函数匹配需要的字段。 |
| NumMethod() int | 返回该类型的方法集中方法的数目。 |
| Method(int) Method | 返回该类型方法集中的第i个方法。 |
| MethodByName(string)(Method, bool) | 根据方法名返回该类型方法集中的方法。 |

`StructField`类型用来描述结构体中的一个字段的信息：

```GO
type StructField struct {
    // Name是字段的名字。PkgPath是非导出字段的包路径，对导出字段该字段为""。
    // 参见http://golang.org/ref/spec#Uniqueness_of_identifiers
    Name    string
    PkgPath string
    Type      Type      // 字段的类型
    Tag       StructTag // 字段的标签
    Offset    uintptr   // 字段在结构体中的字节偏移量
    Index     []int     // 用于Type.FieldByIndex时的索引切片
    Anonymous bool      // 是否匿名字段
}
```



