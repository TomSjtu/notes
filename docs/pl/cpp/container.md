# 标准库容器

## string

> `string`实际上是`basic_string<char>`的别名。

!!! note "推导规则"

    ```C
    auto string1 { "Hello world"};   //string为const char *
    auto string2 { "Hello world"s};  //string为std::string

    ```

如果要推导为`std::string`，需要添加后缀`s`。

### 获取C风格字符串

C++17之后，`c_str()`返回const char *，`data()`返回char *。

### 数值转换

- 数值转换为字符串：`string to_string(T val)`
- 字符串转换为数值：`T stoi(const string& str, size_t* idx = 0, int base = 10)`


## string_view

在过去，接收只读字符串一直是一件比较头疼的事情，`string_view`的出现解决了接口不一致的问题。它用来提供对字符串的只读访问，不拥有字符串的所有权，因此不能用来保存临时字符串。一般用在函数参数中按值传递。




