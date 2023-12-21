# 简明入门
大多数的操作系统都能同时执行多个程序，这被称为多任务处理。实际上，每个处理器核心在给定时间片内只能运行单个任务。CPU通过快速切换时间片造成多个任务同时执行的假象。
实时操作系统(RTOS)中的任务调度器旨在提供可预测的执行模式。这种方式对嵌入式系统意义重大，因为嵌入式系统往往有实时性的要求，比如我要求在3秒内发出警报声。

使用STM32CubeMX创建的FreeRTOS工程中，源码主要涉及两个目录：

- Core
    - Inc目录下的FreeRTOSConfig.h是配置文件
    - Src目录下的freertos.c是STM32CubeMX创建的默认任务
- Middlewares/Thirt_Party/FreeRTOS/Source
    - 根目录下是核心文件，这些文件是通用的
    - portable目录下是移植时需要的文件：port.c和portmacro.h
        - 目录名为：[compiler]/[architecture]
        - 比如：RVDS/ARM-CM3，这表示cortexM3架构在RVDS工具上的移植文件

| FreeRTOS/Srouce/ | 作用 |
| ---- | ---- |
| tasks.c | 必须，任务操作 |
| list.c | 必须， 列表 |
| queue.c | 基本必须，提供队列操作、信号量 |
| timer.c | 可选，提供定时器 |
| event_groups.c | 可选，提供event group功能 |
| croutine.c | 可选，已过时 |


头文件目录需要包含：

- FreeRTOS自身头文件：Middlewares/Third_Party/FreeRTOS/Source/include

- 配置文件：Core/Inc/FreeRTOSConfig.h

数据类型：

在portmacro.h头文件里，定义了两个数据类型：

- TickType_t：
    - FreeRTOS配置了一个周期性的时钟中断：Tick Interrupt
    - 每发生一次中断，中断次数累加，这被称为tick count
    - tick count这个变量的类型就是TickType_t
    - TickType_t可以是16位的，也可以是32位的
    - FreeRTOSConfig.h中定义configUSE_16_BIT_TICKS时，TickType_t就是uint16_t
    - 否则TickType_t就是uint32_t
    - 对于32位架构，建议把TickType_t配置为uint32_t
- BaseType_t：
    - 这是该架构最高效的数据类型
    - 32位架构中，它就是uint32_t
    - 16位架构中，它就是uint16_t
    - 8位架构中，它就是uint8_t
    - BaseType_t通常用作简单的返回值的类型，还有逻辑值，比如pdTRUE/pdFALSE

命名规范：

| 变量名前缀 | 含义 |
| --- | --- |
| ul | uint32_t |
| us | uint16_t |
| uc | uint8_t |
| x | 非stdint类型，比如BaseType_t和TickType_t，或者size_t |
| ux | UBaseType_t |
| e | enum |
| p | 指针 |
| c | char |

| 函数名前缀 | 含义 |
| --- | --- |
| prv | 私有函数(file scope) |
| v | 返回值void |
| task | 定义在task.c |

| 宏前缀 | 含义 |
| --- | --- |
| config | 定义在FreeRTOSConfig.h |

## 内存管理

每次创建任务、队列、互斥锁、软件定时器、信号量或事件组时，内核都需要RAM，RAM可以由RTOS堆动态分配，也可以由编写者手动提供。

尽管C库提供了*malloc()*和*free()*函数，但是：
1. 它们并不总是适用于嵌入式系统
2. 它们耗费时间比较长
3. 它们不是线程安全的
4. 运行结果不确定

因此freertos实现了自己的内存分配接口函数。

freertos中有五种堆内存，文件定义在Middlewares/Third_Party/FreeRTOS/Source/portable/MemMang下。其中最常用的是heap_4.c，它可以将相邻的空间内存块合并，解决了内存碎片问题。而heap_5.c允许多个不连续的内存区域。需要用到以下结构体进行不同内存块的初始化：

```C
typedef struct HeapRegion
{
    uint8_t *pucStartAddress;   //起始地址
    size_t xSizeInBytes;       //大小
}HeapRegion_t;
```

当指定多块内存时，需要用到HeapRegion_t类型的数据，每个数组元素都是一个HeapRegion_t元素。这个数组中，低地址在前，高地址在后。

```C
HeapRegion_t xHeapRegions[] = 
{
    {(uint8_t *)0x80000000UL, 0x10000}, //起始地址和大小
    {(uint8_t *)0x90000000UL, 0x10000}, 
    {NULL, 0}  //表示数组结束
};
```

定义了内存块数组还需要对其进行初始化：

```C
void vPortDefineHeapRegions(const HeapRegion_t *const pxHeapRegions);
```

Heap的分配和释放用到以下函数：

```C
void *pvPortMalloc(size_t xWantedSize);
void vPortFree(void *pv);
```

获取当前空闲内存空间：

```C
size_t xPortGetFreeHeapSize(void);
```

获取程序运行时空闲内存的最小值：

```C
size_t xPortGetMinimumEverFreeHeapSize(void);
```

## 任务管理

一个任务最基本的元素有：

- 任务状态：比如阻塞，就绪，挂起等
- 优先级：每个任务都分配了从0到configMAX_PRIORITIES-1的优先级
- 栈：用来保存任务的局部数据
- 事件：表示任务做了什么事情

### 创建任务

在FreeRTOS中，任务函数的定义如下：

```C
void vATaskFunction(void *pvParameters)
{
    for(;;)
    {
        //RTOS推荐采用事件驱动型任务
        if(WaitForEvent(EventObject, Timeout) == pdPASS)
        {
            //处理事件
        }
        else
        {
            
        }
    }

    vTaskDelete(NULL);  //执行完毕删除自己
}
```
动态创建任务使用的函数如下：

```C
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode,
                        const char *const pcName, 
                        const configSTACK_DEPTH_TYPE usStackDepth,
                        void *const pvParameters,
                        UBaseType_t uxPriority,
                        TaskHandle_t *const pxCreatedTask);
```

参数说明：

| 参数 | 描述 |
| --- | --- |
| pvTaskCode | 任务函数 |
| pcName | 任务名称 |
| usStackDepth | 任务栈大小，单位word |
| pvParameters | 传递给任务函数的参数 |
| uxPriority | 任务优先级 |
| pxCreatedTask | 任务句柄，用来操作任务函数 |
| 返回值 | 成功返回pdPass，失败通常是因为内存不足 |

静态创建任务使用的函数如下：

```C
TaskHandle_t xTaskCreateStatic(TaskFunction_t pxTaskCode,
                                const char *const pcName,
                                const uint32_t ulStackDepth,
                                void *const pvParameters,
                                UBaseType_t uxPriority,
                                StackType_t *const puxStackBuffer,
                                StaticTask_t *const pxTaskBuffer);
```

相对于动态分配，有两个参数不同：

| 参数 | 描述 |
| --- | --- |
| puxStackBuffer | 栈数组，索引必须不小于ulStackDepth |
| pxTaskBuffer | 指向StaticTask_t的指针，用来保存任务结构体 |

每个任务都需要RAM来保存任务状态，并由任务用作其堆栈。如果使用动态创建，则会在程序运行时从FreeRTOS堆中自动分配RAM。如果是静态创建，RAM由编写者手动提供，但在编译时就已确定。

### Tick

任务进入睡眠后需要指定唤醒的时间，FreeRTOS通过Tick变量测量时间。定时器中断以严格的时间精度增加Tick count。每次Tick增加时，内核必须检查现在是否需要解除阻塞或者唤醒任务，高优先级的任务先被唤醒。任务通过调用*vTaskDelay()*函数来主动等待一定的时间。宏*pdMS_TO_TICKS()*将毫秒转换为Tick count。

需要注意的是，由于内核本身的任务切换需要时间，当指定N个Tick的延迟后，实际延迟时间将在(N-1)~(N)个Tick之间。

### 任务状态

任务可以有以下状态：

- 运行：单核只能同时运行一个任务

- 就绪：能够执行但目前没有执行

- 阻塞：正在等待时间或者外部事件

- 挂起：只有明确调用*vTaskSuspend()*和*vTaskResume()*时才会进入或退出挂起状态。

![tskstate](../../images/freertos/tskstate.gif)

### 调度算法

两个核心概念：**抢占**和**轮转**。通过配置configUSE_PREEMPTION使能抢占，即高优先级的任务可以抢占低优先级的任务。通过配置configUSE_TIME_SLICING使能轮转，即相同优先级的任务轮流执行。

### 空闲任务和钩子函数

在没有其他任务运行的时候，执行空闲任务。*vTaskStartScheduler()*函数内部会创建空闲任务，它的优先级为0，永远低于用户任务，且要么处于就绪态，要么处于运行态。

我们可以在空闲任务内添加一个钩子函数，这样空闲任务每执行一次，都会调用一次钩子函数。钩子函数的作用是：

- 执行一些后台任务
- 测量系统空闲时间
- 使系统进入省电模式

要注意的是，钩子函数不能调用任何使其导致阻塞的函数。空闲钩子函数的定义如下：

```C
void vApplicationIdleHook(void);
```

## 任务同步与通信

### 事件组

事件组用一个整数表示，其中高8位留给内核，用其他的位来表示事件。每一位事件的含义由编写者决定，比如用bit0表示灯泡亮灭，bit1表示按键是否按下。这些位，值为1表示事件发生，值为0表示事件未发生。多个任务可以读写位，单个任务也可以等待多个位。

事件组的操作流程如下：

- 创建一个事件组
- 任务A、B控制事件中的某些位
- 任务C、D等待事件中的位，并且决定发生事件后的操作

这里对两个比较重要的函数说明一下：

等待事件函数：

```C
EventBits_t xEventGroupWaitBits(EventGroupsHandle_t xEventGroup,
                                const EvetnBits_t uxBitsToWaitFor,
                                const BaseType_t xClearOnExit,
                                const BaseType_t xWaitForAllBits,
                                TickType_t xTickToWait);
```

| 参数 | 说明 |
| --- | --- |
| xEventGroup | 等待的事件组 |
| uxBitsToWaitFor | 等待的位 |
| xClearOnExit | pdTRUE:清除位 |
| xWaitForAllBits | pdTRUE：等待的位全部为1；pdFALSE：某一位为1 |
| xTicksToWait | 阻塞时间 |

举例说明：

| 事件组的值 | uxBitsToWaitFor | xWaitForAllBits | 说明 |
| --- | --- | --- | --- |
| 0100 | 0101 | pdTrue | 期望bit0,bit2都为1，不满足进入阻塞 |
| 0100 | 0110| pdFALSE | 期望bit0, bit2某位为1，满足退出 |

同步事件函数：

```C
EventBits_t xEventGroupSync(EventGroupHandle_t xEventGroup,
                            const EventBits_t uxBitsToSet,
                            const EventBits_t uxBitsToWaitFor,
                            TickType_t xTicksToWait);
```

该函数用来协同多个任务，当期望的事件一起发生后，才会成功返回。

### 任务通知

当我们使用队列、信号量、事件组等同步方法时，并不知道对方是谁。使用任务通知，可以明确指定通知哪个任务。




