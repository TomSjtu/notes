# shell脚本

## 特殊字符

- \#：注释
- ;：命令分隔符
- ;;：case语句结束符
- '：单引号，原样输出
- "：双引号，可以输出变量名
- `：命令替换

## 字符串


## 数组

创建数组：

```SHELL
# 直接赋值
array=(value1 value2 value3 ...)

# 使用索引赋值
array[0]=value1
array[1]=value2
```

访问数组元素：

```SHELL
# 访问特定元素
echo ${array[2]}

# 访问所有元素
echo ${array[*]}

# 访问数组长度
echo ${#array[*]}
```

遍历数组：

```SHELL
for item in ${array[*]}; do
    echo $item
done
```

## 字典

创建字典：

```SHELL
# 声明关联数组
declare -A dictionary

# 赋值
dictionary[key1]=value1
dictionary[key2]=value2
```

访问字典：

```SHELL
# 访问特定键的值
echo ${dictionary[key1]}

# 访问所有键
echo ${!dictionary[*]}

# 访问所有值
echo ${dictionary[*]}

# 访问字典长度
echo ${#dictionary[*]}
```

遍历字典：

```SHELL
for key in ${!dictionary[*]}; do
    echo ${dictionary[$key]}
done
```
    
## 特殊参数

| 参数名 | 含义 |
| ---- | ---- |
| $0 | 脚本名称 |
| $1-$9 | 位置参数 |
| $# | 参数个数 |
| $@ | 所有参数 |
| $? | 上一条命令的退出状态 |
| $$ | 当前shell的pid |
| $_ | 上一条命令的最后一个参数 |

## 流程控制

if else：

```SHELL
if condition1
then
	command1
elif condition2
	command2
else
	commandN
fi
```

for while：

```SEHLL
for var in item1 item2 ... itemN
do
	command1
	command2
	...
	commandN
done
```

while：

```SHELL
while condition
do
	command1
	command2
	...
	commandN
done
```

case：

```SHELL
case "${opt}" in
	"Install-Puppet-Server" )
		install_master $1
		exit
	;;

	"Install-Puppet-Client" )
		install_client $1
		exit
	;;

	"Exit" )
		exit
	;;

	* ) echo "Bad option, please choose again"
esac
```

## 条件判断

整数条件：

- -eq：等于
- -ne：不等于
- -gt：大于
- -ge：大于等于
- -lt：小于
- -le：小于等于

字符串条件：

- -z：空字符串
- -n：非空
- ==：等于
- !=：不等于

文件条件：

- -e：存在
- -d：目录
- -f：普通文件
- -h：符号链接
- -r：可读
- -w：可写
- -x：可执行

## 函数

```SHELL
function function_name {
	command...
}
```

使用函数时需要注意以下几点：

1. 函数局部变量，在函数被调用之后，外部可见。
2. 调用函数时可以传参，函数体内部通过`$n`的形式来获取参数
3. 用return表示函数返回值，在函数调用后通过`$?`来获取返回值

## grep

grep是文本搜索工具，用于在文件中搜索符合正则表达式的内容，并打印出来。

基本格式：`grep [options] pattern [file...]`

1. pattern：过滤条件
2. file：要搜索的文件(一个或多个)

常用选项参数：

| 参数 | 说明 |
| ---- | ---- |
| -i | 忽略大小写 |
| -n | 显示匹配行号 |
| -r | 递归搜索目录 |
| -l | 只显示文件名 |
| -c | 只显示匹配行数 |
| -o | 只显示匹配的内容 |
| -w | 只匹配过滤到的单词 |
| -v | 反向选择，只显示不匹配的行 |
| -A num | 显示匹配行及之后的num行 |
| -B num | 显示匹配行及之前的num行 |
| -C num | 显示匹配行及上下各num行 |
| -E | 使用扩展正则(等同于egrep) |
| -F | 使用规定字符串模式(等同于fgrep)|
| -P | 使用perl正则表达式 |

## sed

sed是一个流编辑器，主要用于对文本进行非交互式编辑。它可以进行插入、删除、替换、提取等操作，是文本处理和转换的利器。

基本格式：`sed [options] 'script' file`

- script：sed内置的命令脚本，主要用于对文件进行增删改查等操作
	- a：追加
	- d：删除匹配行
	- i：插入文本
	- p：打印匹配行
	- s/正则/替换内容/g：匹配正则，然后替换，g表示全局替换

## awk

awk是一个强大的文本处理工具，主要用于对结构化数据进行模式扫描和处理，有着强大的文本格式化能力。它支持复杂的操作和脚本编写，能够进行模式匹配、文本处理、数据报告生成等。

基本格式：`awk [options] 'pattern {action}' file`

## 颜色

```SHELL
echo -e "\033[32m绿色文本\033[0m"
```

上面这段代码将输出绿色文本。注意，在设置文本输出后，一定要使用`\033[0m`来恢复默认颜色。

一些常用的颜色代码有：

| 颜色 | 前景色 | 背景色 |
| 黑色 | 30 | 40 |
| 红色 | 31 | 41 |
| 绿色 | 32 | 42 |
| 黄色 | 33 | 43 |
| 蓝色 | 34 | 44 |
| 紫色 | 35 | 45 |
| 青色 | 36 | 46 |
| 白色 | 37 | 47 |