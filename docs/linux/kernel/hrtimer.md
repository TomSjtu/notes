# 高精度定时器

传统的定时器精度不足，只能支持Tick级别的精度。为了实现更高精度的定时器，内核独立设计了一个高精度定时器层(High Resolution Timer Layer)，可以提供纳秒级别的精度，以满足对精确时间有迫切需求的程序。

高精度定时器建立在Per CPU变量上，每个CPU都有一个独立的定时器。同时支持低精度模式和高精度模式。内核使用红黑树来组织各个高精度定时器，新的定时器按顺序被插入到红黑树中，树的最左边的节点就是最快到期的定时器。

创建一个高精度定时器：

```C
struct hrtimer hr_timer;
```

初始化定时器：

```C
void hrtimer_init(struct hrtimer *timer, clockid_t which_clock, 
                  enum hrtimer_mode mode);
```

定时器初始化时，需要指定定时模式和时钟类型。

定时模式有以下几种：

- HRTIMER_MODE_ABS：绝对定时模式。在这种模式下，定时器将在一个特定的未来时间点到期。
- HRTIMER_MODE_REL：相对定时模式。在这种模式下，定时器将在当前时间点之后的指定时间间隔到期。
- HRTIMER_MODE_PINNED：固定定时模式。这是一种特殊的绝对定时模式，用于实时任务，确保定时器在特定的CPU上执行，以减少调度延迟。
- HRTIMER_MODE_SOFT：软定时模式。定时器回调函数在软中断上下文中执行。
- HRTIMER_MODE_HARD：硬定时模式。定时器回调函数在硬中断上下文中执行。

时钟类型有以下几种：

- CLOCK_REALTIME：实时时钟
- CLOCK_MONOTONIC：单调时钟
- CLOCK_BOOTTIME：启动时钟
- CLOCK_RAW：原始时钟

启动一个高精度定时器：
```C
int hrtimer_start(struct hrtimer *timer, ktime_t time,
                  const enum hrtimer_mode mode);
```

取消一个正在运行的高精度定时器：
```C
int hrtimer_cancel(struct hrtimer *timer);
```

!!! example "高精度定时器的使用"
    ```C
    #include <linux/hrtimer.h>
    #include <linux/ktime.h>

    // 定义一个高精度定时器的回调函数
    static enum hrtimer_restart my_hrtimer_callback(struct hrtimer *timer_for_my_hrtimer)
    {
        // 定时器到期的处理逻辑
        // ...

        // 如果需要，重新启动定时器
        // hrtimer_start(timer_for_my_hrtimer, ktime_set(0, 1000000), HRTIMER_MODE_REL);

        return HRTIMER_NORESTART; // 或者 HRTIMER_RESTART
    }

    void my_hrtimer_function(void)
    {
        struct hrtimer hr_timer;
        ktime_t ktime;

        // 初始化定时器
        hrtimer_init(&hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);

        // 设置定时器的回调函数
        hr_timer.function = &my_hrtimer_callback;

        // 设置定时时间（例如，1毫秒后）
        ktime = ktime_set(0, 1000000); // 0秒，1000000纳秒

        // 启动定时器
        hrtimer_start(&hr_timer, ktime, HRTIMER_MODE_REL);
    }
    ```



