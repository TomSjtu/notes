# 标准库容器

顺序容器：

- vector
- deque
- list
- forward_list
- array

容器适配器：

- queue
- priority_queue
- stack
  
有序关联容器：

- map
- multimap
- set
- multiset

无序关联容器或哈希表：

- unordered_map
- unordered_multimap
- unordered_set
- unordered_multiset

可变类型容器：

- variat
- optional
- any

## 顺序容器

### vector

vector 提供动态增长的数组，插入或删除为O(n)，随机访问为O(1)。为了提供动态增长的需求，当 vector 内部空间不够时，会分配一块更大的内存空间，然后将原内容复制到新的内存空间中。这个过程非常耗时，因此 vector 在分配内存时，会尽量一次性分配足够多的内存，避免频繁的内存分配。

但是 vector 永远不会释放任何内存，除非 vector 被销毁。从 vector 中删除会员会减少其大小，但是不会释放空间。回收内存的一个技巧是让它与一个空的 vector 进行交换：

```CPP
vector<int>values;
vector<int>().swap(values);
```

上述代码创建了一个临时的空 vector，并与其交换，语句结束后临时变量就会被销毁，这样就达到了释放内存的目的。

初始化：

```CPP
vector<int>intVector1(10, 100);  //10个100
vector<int>intVector2 = {1, 2, 3, 4, 5};
```

复制和赋值：

```CPP
intVector1.assign(5, 100);  //删除并填充5个100
intVector2.swap(intVector1);//交换元素
```

迭代器：

`begin()`指向第一个元素，`end()`指向最后一个元素的下一个位置。同时还有`cbegin()`和`cend()`，返回常量迭代器。`rbegin()`和`rend()`分别返回反向迭代器。

添加和删除元素：

`push_back()`和`pop_back()`用于向后添加和删除元素，`insert()`和`erase()`用于任意位置的插入和删除，`clear()`用于清空。

从 C++20 开始，`std::erase()`和`std::erase_if()`为所有标准库容器提供了删除操作：

```CPP
vector values{1,2,3,2,1};
erase(values, 2);       //删除所有等于2的元素
```

`erase_if()`接收一个谓词，对于应该删除的元素返回true，保留的元素返回false。

大小和容量：

`size()`返回当前元素个数，`capacity()`返回当前分配的内存空间大小。`reserve()`预分配空间。

访问数据：

`data()`方法获取指向这块内存的指针，`operator[]`和`at()`方法用于访问元素。


### deque

deque 是双端队列，与 vector 的区别是：

- 不要求元素保存在连续内存中
- 首尾插入和删除为常量时间
- 支持`push_fron()`和`pop_front()`


### list

list 是一种双向链表，支持链表中任意位置常量时间的插入和删除操作，但是访问单独元素的速度为线性时间，list不支持随机访问元素。

访问元素：

`front()`和`back()`分别访问第一个和最后一个元素。对其他所有元素的访问必须通过迭代器进行。

添加和删除元素：

`push_back()`、`pop_back()`、`emplace()`、`emplace_back()`，5种形式的`insert()`和2种形式的`erase()`和`clear()`，除此之外，还提供`push_front()`、`emplace_front()`和`pop_front()`。

大小和容量：

list 支持`size()`、`empty()`、`resize()`，但不支持`reserve()`和`capacitiy()`。

list 以方法的形式提供的标准库算法有：

| 方法 | 说明 |
| ---- | ---- |
| remove()/remove_if() | 从list中删除指定元素 |
| unique() | 删除连续的重复元素 |
| merge() | 合并两个已排序的list |
| sort() | 排序 |
| reverse() | 反转顺序 |


### array

和 vector 一样，array 支持随机访问迭代器，元素都保存在连续内存中，区别在于 array 的大小固定。

### span

span 是 C++20 引入的轻量级非拥有（non-owning）视图，用于表示连续内存区域中的对象序列。它类似于指针和长度的组合，但提供了更安全和更高级的接口。

它有以下几个特点：

1. 非拥有：`std::span` 不拥有它所指向的内存，因此它不会负责分配或释放内存。
2. 连续内存：`std::span` 只能用于表示连续内存区域中的对象序列。
3. 轻量级：`std::span` 通常只包含一个指针和一个大小，因此它的开销非常小。
4. 可变与不可变：`std::span` 可以表示可变或不可变的数据序列，具体取决于模板参数。

span 的复制成本很低，以下是一个示例：

```CPP
#include <iostream>
#include <span>
#include <vector>

void print(std::span<int> s) {
    for (int i : s) {
        std::cout << i << " ";
    }
    std::cout << std::endl;
}

int main() {
    int arr[] = {1, 2, 3, 4, 5};
    std::vector<int> vec = {6, 7, 8, 9, 10};

    // 使用数组初始化 span
    std::span<int> span1(arr);
    print(span1);  // 输出: 1 2 3 4 5

    // 使用 vector 初始化 span
    std::span<int> span2(vec);
    print(span2);  // 输出: 6 7 8 9 10

    // 使用部分数组初始化 span
    std::span<int> span3(arr, 3);
    print(span3);  // 输出: 1 2 3

    return 0;
}
```

## 容器适配器

queue、priority_queue、stack 都是容器适配器，是对顺序容器的包装，目的是为了简化接口，只提供特定的功能。

### queue

queue 是一种 FIFO（先进先出）容器适配器，它只支持8个方法：

- `push()`和`emplace()`：将元素添加到队列尾部
- `pop()`：移除队列头部的元素
- `front()`和`back()`：返回尾部和头部元素的引用
- `size()`：返回队列中元素的数量
- `empty()`：判断队列是否为空
- `swap()`：交换元素

### priority_queue

priority_queue 是一种优先队列，它不保证严格的 FIFO 顺序，而是保证队列头部元素拥有最高的优先级。它提供的方法更少：

- `push()`和`emplace()`：插入元素
- `pop()`：删除元素
- `top()`：返回队列头部元素
- `size()`、`empty()`和`swap()`：与queue相同

### stack

stack 提供先入后出，提供的方法类似。

## 有序关联容器

与顺序容器不同，有序关联容器将键映射到值，元素保存在类似树的结构中。


### pair

在学习有序关联容器之前，先要了解下 pair 类，它定义在<utility\>头文件中，可以将两个不相关的类型组合起来，并通过 first 和 second 访问。

```CPP
pair<string, int>myPair{"hello", 5};
auto pair2 { make_pair(5, 10.0)};
```

### map

map 保存键值对，插入、查找和删除都是基于键的，复杂度为对数时间。

`operator[]`用于插入元素，它总是会成功：如果键不存在，则会创建；如果键存在，则会替换值。

查找元素：

如果已知元素存在，则通过`operator[]`查找最为简单。`find()`方法可以返回指向键的迭代器，`count()`方法返回给定键的元素个数，对于map来说，这个结果不是0就是1，因为map不允许重复键。

删除元素：

`erase()`方法接受两个输入：指向删除元素的迭代器或者是键值。

### set

如果信息没有显示的键，且希望进行排序以便快速第执行插入、查找和删除，则可以考虑使用 set 容器。

set 容器提供与 map 几乎相同的接口。

multimap 和 multiset 不常用，不在此介绍。

## 无序关联容器

无序版本的关联容器，其元素没有排序，且不提供基于键的快速查找。

请查看《C++20高级编程下册》P561页与有序版本的区别。

## 可变类型容器

### variat

### optional

### any

## 其他容器

### string

> `string`实际上是`basic_string<char>`的别名。

!!! note "推导规则"

    ```CPP
    auto string1 { "Hello world"};   //string为const char *
    auto string2 { "Hello world"s};  //string为std::string

    ```

如果要推导为`std::string`，需要添加后缀`s`。

获取C风格字符串:

C++17 之后，`c_str()`返回const char *，`data()`返回char *。

数值转换:

- 数值转换为字符串：`string to_string(T val)`
- 字符串转换为数值：`T stoi(const string& str, size_t* idx = 0, int base = 10)`

### string_view

在过去，接收只读字符串一直是一件比较头疼的事情，`string_view`的出现解决了接口不一致的问题。它用来提供对字符串的只读访问，不拥有字符串的所有权，因此不能用来保存临时字符串。一般用在函数参数中按值传递。

### bitset

`bitset`是标准库提供的用于存储和操作位的数据结构。可通过`set()`、`reset()`、`flip()`方法改变单个位的值，通过重载的`operator[]`运算符访问和设置单个位。`test()`方法用于测试位是否为1。

### tuple

`std::tuple`是 C++11 引入的新容器，可以用来存储不同类型的值。它类似与结构体，但是不需要显示定义类型，并且可以通过索引来访问元素。

```CPP
tuple<string, int, double> t1{"Shark", 123, 3.14};
string fish = get<0>(t1);
int count = get<1>(t1);
double weight = get<2>(t1);
```

`tuple`的元素从零开始编号，通过`get<N>`方法访问第 N 个元素，N 必须为常量。`tuple`中具有唯一类型的元素可以通过其类型被获取。
