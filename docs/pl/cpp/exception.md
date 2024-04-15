# 异常处理

异常处理的基本形式如下：

```CPP
try {
    // 可能会抛出异常的代码
} catch (ExceptionType1 e1) {
    // 处理ExceptionType1类型的异常
} catch(ExceptionType2 e2){
    // 处理ExceptionType2类型的异常
}
//剩余代码
```

它的处理流程如下：

- 如果没有抛出异常：catch块的语句不会被执行，程序继续执行剩余代码。

- 如果抛出了异常：根据不同的异常找到对应的catch块，一直执行到throw语句，之后的语句不会被执行。

一个最简单的除0异常如下：

```CPP
double SafeDivide(double num1, double num2)
{
    if(num2 == 0){
        throw invalid_argument("Divide by zero");
    }
    return num1 / num2;
}

int main()
{
    try{
        cout<<SafeDivide(5, 2)<<endl;
        cout<<SafeDivide(10,0)<<endl;
    }catch(const invalid_argument& e){
        cout<<"Caught exception:"<<e.what()<<endl;
    }
}
```

`e.what()`函数返回一个`const char *`的字符串。

