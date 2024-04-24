# 内存管理

使用`new`关键字动态分配的内存始终在堆区上分配。在C++中，`new`关键字不仅分配内存，还会调用响应的构造函数以构建对象。例如有一个Simple类，当调用`new Simple[4]`时，Simple的构造函数就会被调用4次。

通过原始指针分配的`new`对象必须与`delete`关键字配对(`new []`与`delete[]`配对)，以释放内存并调用相应的析构函数。在释放完指针之后，应将其指向`nullptr`，以避免访问悬空指针。

## 智能指针

C++11实现了三种智能指针：`std::unique_ptr`、`std::shared_ptr`和`std::weak_ptr`，能够自动管理指针所指向的内存的类。智能指针是一种模板类，可以指向任何类型的对象。

智能指针的实现原理是利用RAII（Resource Acquisition Is Initialization）技术，在构造函数中申请内存，在析构函数中释放内存。这样，当智能指针超出作用域时，会自动调用析构函数，释放内存。

### unique_ptr

`unique_ptr`是一个独占所有权的智能指针，其引用计数永远是1。

它的初始化方式如下：

```C
std::unique_ptr<int>sp = std::make_unique<int>(3);
```

`unique_ptr`禁止拷贝和复制操作，例外是可以通过函数返回值返回一个`unique_ptr`。可以使用`std::move()`将所有权转移。

默认情况下，智能指针对象在析构时只会释放其持有的堆内存，如果需要自定义资源析构函数，可以在定义时添加：`std::unique_ptr<T, Deletor>`。

### shared_ptr

`shared_ptr`是一个共享所有权的智能指针，它允许多个指针指向同一个对象。

`shared_ptr`的初始化方式如下：

```C
std::shared_ptr<int>sp = std::make_shared<int>(3);
```

`shared_ptr`的引用计数会随着对象的释放而减少，当最后一个对象析构时，将释放其所占有的资源。`shared_ptr`提供了一个`use_count()`方法来获取当前资源的引用计数。

### weak_ptr

`weak_ptr`是一种弱引用，引入它的目的是协助`shared_ptr`工作。

`weak_ptr`可以从一个`shared_ptr`或另一个`weak_ptr`对象构造。`lock()`方法用来获得对应的`shared_ptr`。`expired()`方法用来检测对应的`shared_ptr`是否已经释放。

`unique_ptr`的大小和原始指针一样，而`shard_ptr`和`weak_ptr`是原始指针的两倍。

记住，一旦使用智能指针接管了资源的所有权，所有对资源的操作都应该通过智能指针对象进行，而不要再使用原始指针。三种智能指针都可以通过`get()`方法获取原始指针。