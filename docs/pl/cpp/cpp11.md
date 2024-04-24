# C++11 

## 新字符类型

C++98中提供了`wchar_t`字符类型和前缀L，用以表示宽字符，但是在不同平台上宽字符的字节不同，为了解决这个问题，新标准提出了`char16_t`和`char32_t`字符类型，分别表示16位和32位的Unicode字符，并且提供了三种UTF编码前缀：

- UTF-8：u8
- UTF-16：u
- UTF-32：U

## auto占位符

`auto`关键字是一个非常有用的关键字，可以省去手写类型的烦恼。它一般使用在以下场景：

1. 一眼就能看出变量的初始化类型
2. 较为复杂的类型比如`lambda`表达式

## 强枚举类型

旧式的枚举类型存在名称冲突的问题，即在枚举中定义的变量会与父作用域中的名称产生冲突，因此旧式的枚举必须保证独一无二的枚举值，即便是在多文件中也是如此。且C编译器会为其赋值为int类型的变量，它甚至可以与int类型的变量直接进行比较：

```CPP
enum color {
    red,
    green,
    blue
};

int red = 1; //名称冲突，编译无法通过

if(red == 0){
    printf("red == 0\n");
}
```

使用强枚举类型可以避免以上问题：

```CPP
enum class color {
    red,
    green,
    blue
};
```

强枚举类型的成员变量作用域限定在名称空间内。同旧式枚举类型一样，编译器也会对强枚举类型成员赋值，区别是不会自动转换成整数，因此以下代码是非法的：`if(color::red == 0) {}`。


## 属性

gcc编译器支持属性`__attribute__`，可以用于修饰变量、函数、类型等。从C++11开始，使用双括号语法[[]]对属性进行标准化支持。

### [[nodiscard]]

`[[nodiscard]]`属性用于修饰函数，表示该函数的返回值不会被忽略，如果被忽略，编译器会发出警告。

### [[deprecated]]

`[[deprecated]]`属性表示该内容已经被弃用，不建议使用。

### [[maybe_unused]]

`[[maybe_unused]]`属性禁止编译器在未使用某些内容时发出警告。

### [[noreturn]]

`[[noreturn]]`属性用于修饰函数，表示该函数不会返回。

### [[likely]]

`[[likely]]`属性告诉编译器代码执行这个分支的可能性比较大。

## 结构化绑定

结构化绑定允许同时获取一个元组的多个值，例如假设有：

`array values{11, 22, 33};`

则可以使用`auto [a, b, c] = values;`来获取`a`、`b`、`c`的值。注意，结构化绑定必须使用`auto`关键字以自动推导类型。

使用结构化绑定声明的变量数量必须与右侧表达式中的值数量匹配。

通过使用`auto&`或`const auto &`，还可以创建一组非const或const引用。

## 初始化列表

初始化列表在<initializer_list\>中定义，它允许使用花括号初始化一个对象。列表中的所有元素必须为统一类型。

```CPP
int makeSum(initializer_list<int> values)
{
    int total{0};
    for(auto value: values)
    {
        total += value;
    }
    return total;
}

int a {makeSum({1, 2, 3, 4, 5})};
```

## 原始字面量

原始字面量是C++11引入的新特性，以`R"()"`的形式定义。它会将括号内的内容原样输出。

## 显式默认与删除

`default`和`delete`关键字用于修饰成员函数，告诉编译器使用默认版本或将其删除。

## explicit

`explicit`关键字用于修饰只有一个参数的构造函数，告诉编译器禁止隐式类型转换。

## static_assert

`static_assert`关键字用于在编译期进行断言，它接受一个常量表达式和提示字符串。如果第一个常量表达式的值为false，则会产生一条编译错误，第二个参数就是提示的字符串。

## constexpr

C++11中将变量的只读和常量分开，只读属性用`const`修饰，常量属性用`constexpr`修饰。

`constexpr`修饰的称为常量表达式，用于在编译期计算出结果。

`cosntexpr`还可以修饰函数，使得函数的返回值是一个常量表达式。

## 右值引用

在C++中，左值是可以获取其地址的一个变量，由于经常出现在赋值语句的左边，因此被称为左值。但并不是所有非左值就是右值，例如字面量、临时对象。考虑如下语句：

```CPP
int a = 4 * 2;
```

在这条语句中，a为左值，4 * 2的结果是右值，它是一个临时值，语句执行完就会被销毁。

右值引用就是用来表示右值的引用，通过`&&`符号来表示。它的目的是延长右值的生命周期，减少不必要的拷贝操作，提升系统的性能。右值引用只能用右值来初始化：

```CPP
//左值
int num = 9;

//左值引用
int &a = num;

//右值引用
int &&b = 8;
```

我们再来看这个例子：

```CPP
void handleMessage(string &message)
{
    cout<<"lvalue Message:"<<message<<endl;
}

void handleMessage(string &&message)
{
    cout<<"rvalue Message:"<<message<<endl;
}

int main()
{
    string a {"hello"};
    handleMessage(a);           //调用左值版本

    string b {"world"};
    handleMessage(a+b);         //调用右值版本

    handleMessage("world");     //调用右值版本
}
```

在上述例子中，handleMessage函数有两个版本，一个是接受左值参数，另一个接受右值参数。在调用handleMessage函数时，根据传入参数的类型，编译器会自动选择调用左值版本还是右值版本。我们可以使用`std::move()`来强制调用右值版本：`handleMessage(std::move(a))`。

使用右值引用的注意点有：

1. 左值和右值时独立于他们的类型的，右值引用类型可能是左值也可能是右值
2. 编译器会将已命名的右值引用视为左值，将未命名的右值引用视为右值
3. `auto &&`或者函数参数类型自动推导的`T &&`是一个未定的引用类型，它取决于初始化值的类型。

### 移动语义

移动语义的目的是减少类的拷贝。为了对类增加移动语义，需要手动实现移动构造函数和移动赋值运算符。这两个函数应使用`noexcept`关键字标记，告诉编译器不会抛出异常。

一个简单的定义如下：

```CPP
class Simple{
    public:
        Simple(Simple &&)noexcept;
        Simple & operator=(Simple &&)noexcept;
};
```

注意：在返回值语句中，只需要`return object`即可，不需要使用`std::move()`。

### std::move

使用`std::move()`函数可以将左值转换为右值，该函数不复制任何内容，只是转移对象的所有权。如果某个对象在后续不会被使用了，那么就可以考虑使用`move()`，转移该对象的所有权而避免任何复制操作。

### 完美转发

一个右值引用作为函数参数的形参时，在函数内部传递的时候，它会变成一个左值。如果要保持原来的类型，可以使用`std::forward()`函数，该函数实现的功能被称为完美转发。

## lambda表达式

std::function是一个多态函数包装器，可以用来创建指向任何可调用对象的类型，比如函数、函数对象或者lambda表达式。下面代码是一个使用的示例：

```CPP
void func(int num, string_view str)
{
    cout<<"using function"<<endl;
}

int main()
{
    function<void(int, string_view)>f1{func};
    f1(1,"hello");
}
```

可以使用`auto f1{func}`来定义f1，只不过编译器推导的类型为函数指针，而不是std::function。

lambda表达式的语法如下：

```CPP
[capture list](parameter list)mutable exception attribute -> return type {statement}
```

编译器自动将任何lambda表达式转换为函数对象。如果不指定返回类型，则编译器会自动进行推导。推导的结构会移除任何引用和const限定，除非使用`decltype(auto)`。


[ ]称为捕获块，lambda有三种捕获方式：

1. [ ]：不捕获任何变量
2. [ & ]：按引用方式捕获外部作用域中的所有变量
3. [ = ]：按值方式捕获外部作用域中的所有变量

下面是一些捕获块的示例：

- [&x]：引用捕获x
- [x]：按值捕获x
- [=, &x]：默认按值捕获，x按引用捕获
- [this]：捕获当前类对象，对象需保证存活
- [*this]：捕获当前对象的拷贝，对象可以不存活
- [=, this]：按值捕获所有内容，显示捕获this指针


!!! warning

    全局变量总是使用引用捕获，即便要求按值捕获！


