# 多线程编程

## this_thread

`std::this_thread`是一个关于线程的命名空间，提供了四个公共的成员函数，用于对当前线程进行操作。

- `get_id`：获取当前线程的 ID
- `sleep_for`：使当前线程休眠一段时间
- `sleep_until`：使当前线程休眠到某个时间点(timepoint)
- `yield`：使当前线程让出 CPU 资源

## 线程库

<thread\> 库中定义的线程库可以使用多种方式创建线程，包括：

- 通过函数指针创建线程：可以传入多个参数
- 通过 lambda 表达式创建线程
- 通过类成员函数创建线程：需要同时传递类方法和类实例

调用`join()`会阻塞主线程，直到线程函数执行完毕。而调用`detach()`会使线程与主线程分离，在线程函数执行完毕后由操作系统回收。需要注意的是，主进程退出后，分离的线程也会被回收。如果它仍在运行，那么程序的行为是未定义的。

C++11 提供了一些辅助函数用于更精细化地操作线程：

```CPP
std::thread t(func);
std::cout<<"当前线程个数："<< t.get_id() <<std::endl;
std::cout<< "当前cpu个数："<< std::thread::hardware_concurrency() <<std::endl;
std::this_thread::sleep_for(std::chrono::seconds(10));
```

### 局部变量

在线程中使用局部变量可能会遇到局部变量已经被释放，然而线程还在使用的问题。为了避免这个问题，可以使用以下几个措施：

- 通过智能指针传递参数，保证局部变量在使用期间不会被释放。
- 将局部变量的值作为参数传递，需要有拷贝赋值的功能，且拷贝会浪费空间和效率。
- 将线程运行方式改为 join，保证局部变量被释放前线程已经运行结束，但这么做会改变运行逻辑。

### 引用参数

当线程要调用的回调函数参数为引用类型时，需要将参数显示转换为引用对象后再传递给线程的构造函数，否则会触发一个编译错误：

```CPP

void change_param(int& param) {
    param++;
}

void ref_oops(int some_param) {
    std::cout << "before change , param is " << some_param << std::endl;
    //需使用引用显示转换
    std::thread  t2(change_param, std::ref(some_param));
    t2.join();
    std::cout << "after change , param is " << some_param << std::endl;
}
```

### jthread

`std::jthread`是 C++20 引入的线程库，它是`std::thread`的扩展，提供了更加安全的线程管理机制。它支持自动`join()`，当`jthread`对象被销毁时，确保线程安全退出。还提供了`stop_token`参数，用于控制线程的退出。

```CPP
#include <iostream>
#include <thread>
#include <chrono>

void threadFunction(std::stop_token stopToken) {
    while (!stopToken.stop_requested()) {
        std::cout << "Thread is running..." << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
    std::cout << "Thread is interrupted." << std::endl;
}

int main() {
    std::jthread t(threadFunction); // 创建线程
    std::this_thread::sleep_for(std::chrono::seconds(3)); // 主线程等待 3 秒
    t.request_stop(); // 请求中断线程
    // 线程 t 在作用域结束时自动加入
    std::cout << "Main function is done." << std::endl;
    return 0;
}
```

## 锁

`std::mutex`是独占的互斥锁，用于保护临界区代码，提供了基本的锁定和解锁操作。然而直接使用`std::mutex`容易忘记解锁或异常情况未正确解锁的问题，因此不建议直接使用`std::mutex`。而使用辅助类`std::lock_guard`或者`std::unique_lock`。

`std::lock_guard`是一种 RAII 风格的互斥锁，在构造时自动加锁，在析构时自动解锁，有效避免了因异常等原因导致的死锁或资源泄露问题，适用于一些简单的加锁逻辑。

`std::unique_lock`的用法和`std::lock_guard`类似，但是可以延迟加锁和手动解锁，这方便了我们控制临界区代码的粒度，可以和条件变量很好的配合使用。

`std::scoped_lock`是 C++17 引入的 RAII 风格锁管理工具，用于管理多个互斥量的锁定和解锁。它能够避免死锁，并确保在作用域结束时自动释放锁。

`std::shared_mutex`是一种读写锁，允许多个线程同时读临界区代码，但是只允许一个线程写临界区代码。提供了 `lock()`, `try_lock()`, 和 `try_lock_for()` 以及 `try_lock_until()` 函数，这些函数都可以用于获取互斥锁。还提供了 `try_lock_shared()` 和 `lock_shared()` 函数，这些函数可以用于获取共享锁。

下面是一个综合示例：

```CPP
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <chrono>

std::mutex mtx;
int shared_resource = 0;

void increment(int id) {
    for (int i = 0; i < 5; ++i) {
        {
            // 使用 lock_guard 简单地锁定和解锁
            std::lock_guard<std::mutex> lock(mtx);
            ++shared_resource;
            std::cout << "Thread " << id << " increments resource to " << shared_resource << '\n';
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

void complex_operation(int id) {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock); // 延迟锁定
    // 执行一些非临界区的操作...
    
    // 现在开始临界区操作
    lock.lock();
    ++shared_resource;
    std::cout << "Thread " << id << " performs a complex operation and increments resource to " << shared_resource << '\n';
    lock.unlock();
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(increment, i);
        threads.emplace_back(complex_operation, i);
    }

    for (auto& th : threads) {
        th.join();
    }

    return 0;
}
```

## 线程本地存储

线程本地存储(Thread-Local Storage, TLS)是一种机制，允许每个线程拥有独立的变量副本，各个线程之间互不影响。在 C++11 之后，可以通过`thread_local`关键字来声明一个线程本地存储的变量，可以用于全局变量、静态成员变量和局部变量。它有以下特点：

- 初始化：对于全局的`thread_local`变量，在第一次被访问时初始化。对于局部的`thread_local`变量，在线程第一次进入函数时初始化。
- 作用域：`thread_local`变量的作用域与普通变量相同。
- 生命周期：与线程相同。
- 内存管理：由编译器自动管理，不需要手动分配和释放。

```CPP
#include <iostream>
#include <thread>

// 全局 thread_local 变量
thread_local int global_tls_variable = 0;

void print_tls() {
    // 函数内的 static thread_local 变量
    static thread_local std::string thread_name = "Thread-" + std::to_string(++global_tls_variable);
    
    std::cout << "Hello from " << thread_name << std::endl;
}

int main() {
    std::thread t1(print_tls);  //打印Thread-1
    std::thread t2(print_tls);  //打印Thread-1

    t1.join();
    t2.join();

    return 0;
}
```

## 原子变量

原子变量提供了一种在多线程环境中安全操作数据的方式，从 C++17 开始，所有原子类型都包含一个静态常量表达式成员变量`is_always_lock_free`，用于表示在任意给定的目标硬件上，该原子类型是否以无锁结构形式实现。

头文件 <atomic\> 定义了 C++ 中的原子类型，标准库为所有基本类型都定义了命名的原子类型，比如`atomic_int`、`atomic_bool`等。

### atomic_flag

`std::atomic_flag`是一个特殊的原子类型，仅支持两个操作：

- `test_and_set()`：检查并设置标志。如果标志未被设置，则设置标志并返回 false；如果标志已被设置，则返回 true。
- `clear()`：清除标志，设置为 false。

### atomic<T\>

对于`std::atomic<T\>`，支持以下常用操作：

- `load()`：原子地加载并返回当前值
- `store()`：原子地存储一个新值
- `exchange()`：原子地交换当前值和给定值
- `compare_exchange_weak()`：原子地比较并交换当前值和给定值
- `compare_exchange_strong()`：原子地比较并交换当前值和给定值

### 内存次序

原子操作可以指定内存序，以控制操作的顺序和可见性，包括：

| 内存序 | 语义 | 适用场景 |
|--------- |--------------|-------------|
| `std::memory_order_relaxed`   | 无同步或顺序约束               | 计数器、统计信息             |
| `std::memory_order_acquire`   | 加载操作，防止后续操作重排序   | 读取共享数据                 |
| `std::memory_order_consume`   | 加载操作，防止依赖操作重排序   | 依赖链（如指针解引用）       |
| `std::memory_order_release`   | 存储操作，防止之前操作重排序   | 写入共享数据                 |
| `std::memory_order_acq_rel`   | 加载-存储操作，同时获取和释放  | 自旋锁、复杂同步             |
| `std::memory_order_seq_cst`   | 顺序一致性，最强约束           | 需要强一致性的场景           |

## call_once

c++11 提供了`std::call_once`来保证某一函数在多线程环境中只调用一次，它需要配合`std::once_flag`使用：

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

`std::condition_variable`是 c++11 引入的一种同步机制，它可以阻塞一个线程或者个线程，直到有线程通知或者超时才会唤醒正在阻塞的线程，条件变量需要和锁配合使用，这里的锁就是上面介绍的`std::unique_lock`。

## 异步

future、promise 和 async 是 C++11 引入的用于异步编程的三个组件。

`std::future`是一个类模板，代表一个未来可用的结果。可以用来获取`std::async`和`std::promise`启动任务的结果。可以用`get()`和`wait()`方法来获取结果：

- `get()`：阻塞当前线程，直到异步任务完成，然后返回结果。
- `wait()`：阻塞当前线程，不返回任务结果，只等待异步任务完成。

`std::async`是一个用于创建异步函数的模板函数，它返回一个`std::future`对象，该对象用于获取函数的返回值。

```CPP
std::string fetchData(std::string query){
    // 模拟数据库请求
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return "Data: " + query;
}

int main() {
    std::future<std::string> resFromDB = std::async(std::launch::async, fetchData, "query");
    std::cout << resFromDB.get() << std::endl;
    return 0;
}
```

`std::promise`也是一个类模板，用于设置异步任务的结果，由`std::future`获取。

```CPP
#include <iostream>
#include <thread>
#include <future>

void set_value(std::promise<int> prom) {
    // 设置 promise 的值
    prom.set_value(10);
}

int main() {
    // 创建一个 promise 对象
    std::promise<int> prom;
    // 获取与 promise 相关联的 future 对象
    std::future<int> fut = prom.get_future();
    // 在新线程中设置 promise 的值
    std::thread t(set_value, std::move(prom));
    // 在主线程中获取 future 的值
    std::cout << "Waiting for the thread to set the value...\n";
    std::cout << "Value set by the thread: " << fut.get() << '\n';
    t.join();
    return 0;
}
```

