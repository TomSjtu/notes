# 内存管理

使用`new`关键字动态分配的内存始终在堆区上分配。在C++中，`new`关键字不仅分配内存，还会调用响应的构造函数以构建对象。例如有一个Simple类，当调用`new Simple[4]`时，Simple的构造函数就会被调用4次。

通过原始指针分配的`new`对象必须与`delete`关键字配对(`new []`与`delete[]`配对)，以释放内存并调用相应的析构函数。在释放完指针之后，应将其指向`nullptr`，以避免访问悬空指针。

## 智能指针

C++11引入了智能指针的概念，智能指针是一种能够自动管理指针所指向的内存的类。智能指针是一种模板类，可以指向任何类型的对象。

智能指针的实现原理是利用RAII（Resource Acquisition Is Initialization）技术，在构造函数中申请内存，在析构函数中释放内存。这样，当智能指针超出作用域时，会自动调用析构函数，释放内存。

### unique_ptr

`unique_ptr`是一种独占式智能指针，它只能指向一个对象。如果对象的所有者唯一，应使用该指针。`unique_ptr`的优点是，即便是在抛出异常时，也能正确释放内存和资源。

!!! warning "内存泄漏的例子"

    ```CPP
    void couldLeak()
    {
        Simple *mySimplePtr{new Simple{}};
        mySimplePtr->go();
        delete mySimplePtr;
    }
    ```

在上面代码中，如果`go()`方法抛出异常，则delete永远不会被调用，这会导致内存泄漏。

应始终使用`make_unique()`来创建`unique_ptr`：`auto mySimpleSmartPtr{make_unique<Simple>()}`;

`get()`方法可以直接获得底层指针，`reset()`方法可以释放底层指针，`release()`方法返回底层指针，然后将`unique_ptr`设置为nullptr。

`unique_ptr`存储C风格数组的方式是：`auto array{make_unique<int[]>(10)}`。

### shared_ptr

`unique_ptr`拥有对资源的唯一使用权，因此无法复制。如果我们需要共享所有权，则必须使用`shared_ptr`。共享指针内部维护了引用计数用来统计资源的使用情况。

`shared_ptr`用`make_shared()`来创建。

`shared_ptr`支持`get()`和`reset()`方法，区别是`reset()`在最后一个`shared_ptr`销毁时，才释放底层指针。`shared_ptr`不支持`release()`。`use_count()`用来检索共享资源的实例数量。
