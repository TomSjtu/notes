# I/O流

C的`printf()`和`scanf()`不能处理错误，也不是类型安全的。C++通过流的机制提供了更精良的输入输出方法，流是一种灵活的、面向对象的I/O方法。

## 流的概念

流可以看成是数据的缓冲，`cout`和`cin`是C++在std名称空间预定义的流实例。缓冲流不会讲数据立即发送，而是先存储在缓冲区，然后以块的形式发送。非缓冲流则会立即将数据发送至目的地。缓冲的目的通常是为了提高性能。`flush()`方法可以强制刷新缓冲流的缓冲区。

| 流 | 说明 |
| --- | --- |
| cin | 标准输入流 |
| cout | 缓冲的标准输出流 |
| cerr | 非缓冲的错误输出流 |

这些流还支持宽字符版本，以w开头：wcin、wcout、wcerr。宽字符用于字符数多于英语的语言，比如中文。

!!! note

    GUI程序通常没有控制台，因此不要假定这些流存在，即便写入了数据，用户也无法看到。

## 文件流

`std::ofstream`和`std::ifstream`分别用于打开文件进行写入和读取。这两个类在<fstream\>头文件中定义。

文件流的参数有：

| 参数 | 说明 |
| --- | --- |
| ios_base::app | 在末尾追加 |
| ios_base::binary | 以二进制模式打开 |
| ios_base::in | 从开头开始读取 |
| ios_base::out | 从开头开始写入 |
| ios_base::trunc | 删除已有数据 |

`ifstream`自动包含`ios_base::in`，`ofstream`自动包含`ios_base::out`。且二者会自动关闭底层文件，因此不需要显示调用`close()`。

### 文件流的定位

文件流支持定位操作，`seekp()`和`seekg()`分别用于定位写入和读取的文件流。位置的类型为`std::streampos`，偏移量的类型为`std::streamoff`，两种类型都以字节计数。

预定义的3个位置如下图：

| 位置 | 说明 |
| --- | --- |
| ios_base::beg | 流开头 |
| ios_base::end | 流末尾 |
| ios_base::cur | 流当前位置 |

!!! example "定位流"

    ```CPP
    //定位到输出流的开头
    std::streampos begPos = outStream.seekp(ios_base::beg);

    //定位到输入流的末尾
    std::streampos endPos = inStream.seekg(ios_base::end);
    ```

`tellp()`和`tellg()`分别用于获取写入和读取的文件流的当前位置。



## filesystem库

C++17引入了`std::filesystem`库，用于处理文件系统。

### 路径

路径可以是绝对的，也可以是相对的：`filesystem::path p {"test.txt"}`。

可以使用`append()`函数或者`operator+=`将字符串拼接到路径中，与平台相关的路径分隔符会自动插入。而`concat()`函数或`operator+=`则不会插入分隔符。

基于范围的for循环可以遍历路径的不同组件，比如：

```CPP
filesystem::path p {R"(C:\Foo\Bar)"};
for(const auto &component: p){
    cout<<component<<endl;>>
}
```

Windows上的输出如下：

```
"C:"
"\\"
"Foo"
"Bar"
```

### 目录

`create_directory()`用于创建一个新目录。

