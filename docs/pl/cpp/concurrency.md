# 多线程编程

## this_thread

`std::this_thread`是一个关于线程的命名空间，提供了四个公共的成员函数，用于对当前线程进行操作。

- `get_id`：获取当前线程的ID
- `sleep_for`：使当前线程休眠一段时间
- `sleep_until`：使当前线程休眠到某个时间点(timepoint)
- `yield`：使当前线程让出CPU资源


## 线程库

<thread\>库中定义的线程库可以使用多种方式创建线程，包括：

- 通过函数指针创建线程：可以传入多个参数
- 通过lambda表达式创建线程
- 通过类成员函数创建线程：需要同时传递类方法和类实例

调用`join()`会阻塞主线程，直到线程函数执行完毕。而调用`detach()`会使线程与主线程分离，在线程函数执行完毕后由操作系统回收。

C++11提供了一些辅助函数用于更精细化地操作线程：

```CPP
std::thread t(func);
std::cout<<"当前线程个数："<< t.get_id() <<std::cendl;
std::cout<< "当前cpu个数："<< std::thread::hardware_concurrency() <<std::cendl;
std::this_thread::sleep_for(std::chrono::seconds(10));
```

## 锁

`std::mutex`是一个独占的互斥锁，`std::lock_guard`和`std::unique_lock`是对互斥锁的RAII实现，可以动态地释放锁资源。它们的主要区别在于灵活性和性能开销上。

`lock_guard`是一个非常简单的锁管理类，它在构造时自动加锁，在析构时自动解锁。这意味着它适用于那些简单的同步需求，例如在一个函数作用域内保证一段代码的互斥执行。`lock_guard `不能手动解锁，也不能尝试锁（try-lock），它只有最基本的锁定功能。因此，`lock_guard`的性能开销相对较小。

`unique_lock`则提供了更多的功能，它提供了与`lock_guard`相同的基本功能，并且还可以手动加锁和解锁，可以尝试锁（try-lock），还可以设定锁的拥有权转移，以及与条件变量一起使用时能够自动解锁和重新加锁。`unique_lock`的这些额外功能使得它的性能开销相对较大，但在需要更灵活的锁管理时非常有用。

## 线程本地存储

`thread_local`可以将任何变量标记为线程本地数据，每个线程都有一个独立的副本。

## 原子变量

头文件<atomic\>定义了C++中的原子类型，标准库为所有基本类型都定义了命名的原子类型，比如`atomic_int`、`atomic_bool`等。


## call_once

c++11提供了`std::call_once`来保证某一函数在多线程环境中只调用一次，它需要配合`std::once_flag`使用：

```CPP
std::once_flag onceflag;

void CallOnce() {
   std::call_once(onceflag, []() {
       cout << "call once" << endl;
  });
}

int main() {
   std::thread threads[5];
   for (int i = 0; i < 5; ++i) {
       threads[i] = std::thread(CallOnce);
  }
   for (auto& th : threads) {
       th.join();
  }
   return 0;
}
```

## 条件变量

`std::condition_variable`是c++11引入的一种同步机制，它可以阻塞一个线程或者个线程，直到有线程通知或者超时才会唤醒正在阻塞的线程，条件变量需要和锁配合使用，这里的锁就是上面介绍的`std::unique_lock`。

## 异步

`async`可以直接创建异步任务，返回的结果会保存在`std::future`中。

```CPP
int func(int num){return num + 1;}

auto res = std::async(func, 5);
std::cout<< res.get() <<std::endl;
```

