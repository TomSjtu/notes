# C++杂项

## static

1. 修饰成员变量：使得所有对象只保存一份拷贝
2. 修饰成员函数：使得所有对象共享同一个函数，static成员函数只能访问static成员变量

## const

1. 修饰成员函数：使得成员函数不能修改成员变量
2. 修饰成员变量：使得成员变量不能被修改
3. 修饰函数参数：使得函数参数不能被修改

不修改对象的方法应声明为const，const对象只能调用const方法。

## 类型转换

- `const_cast`：用于取消变量的const属性
- `static_cast`：用于执行基本数据类型之间的转换，或者在继承结构中向下转换，它在编译时进行类型检查
- `dynamict_cast`：用于执行基类和派生类之间的转换，它是在运行时进行类型检查
- `reinterpret_cast`：用于低级转换，可以将指针类型转换为任何其他指针类型，即使这种转换不合法，因此使用时必须非常小心

类型转换小结：

| 场景 | 类型转换 |
|------|----------|
| 移除const属性 | const_cast |
| 语言支持的显示强制转换 | static_cast |
| 同一继承层次结构中，一个类的指针/引用转换为另一个类的指针/引用 | dynamic_cast |
| 任何指针类型转换为任何其他指针类型 | reinterpret_cast |
| 一种类型的引用转换为其他不相关类型的引用 | reinterpret_cast |

## 友元

C++允许某个类将其他类、其他类的成员函数或非成员函数声明为友元，友元可以访问类的protected、private数据成员和方法。