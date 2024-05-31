# 函数

符合规范的函数一般写成如下的形式：

```GO
func functionName(parameter_list) (return_value_list) {
   
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

