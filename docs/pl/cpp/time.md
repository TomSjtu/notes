# 日期和时间

C++标准库中的与时间相关的功能都放在std::chrono名称空间中，定义在头文件<chrono\>。

这些库包含以下组件：

- 时间间隔(Duration)
- 时钟(Clock)
- 时间点(Time point)
- 日期(Date) C++20
- 时区(Time zone) C++20

## Duration

Duration表示一个时间间隔，它由一个数量和单位组成：

`template<class Rep, class Period = std::ratio<1>> class duration;`

Rep表示重复的次数，ratio的单位是秒。

标准库为了方便使用，就定义了一些常用的时间间隔：

```CPP
typedef duration <Rep, ratio<3600,1>> hours;
typedef duration <Rep, ratio<60,1>> minutes;
typedef duration <Rep, ratio<1,1>> seconds;
typedef duration <Rep, ratio<1,1000>> milliseconds;
typedef duration <Rep, ratio<1,1000000>> microseconds;
typedef duration <Rep, ratio<1,1000000000>> nanoseconds;
```

通过定义这些常用的时间间隔类型，我们能方便的使用它们，比如线程的休眠：

```CPP
std::this_thread::sleep_for(std::chrono::seconds(3)); //休眠三秒`
```

`duration_cast`可以将当前的时钟周期转换为其他的时钟周期。


## Time point



## Clock

表示当前的系统时钟，它有三种：

- system_clock：系统时钟，会被修改
- steady_clock：不能被修改的时钟
- high_resolution_clock：高精度时钟

成员函数`now()`用来获取当前时间点。


