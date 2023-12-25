# CMSIS标准

Cortex系列的芯片内核是相同的，但是不同的厂商在生产的时候外设会有区别，这些差异导致软件在同内核，不同外设的芯片上移植困难。为了解决不同芯片厂商生产的兼容性问题，ARM与各芯片厂商建议了CMSIS标准。

所谓的CMSIS标准，其实就是新建了一个软件抽象层，将不同的RTOS的接口统一起来，提供CMSIS的接口供使用者使用。

本文只简单列举了部分CMSIS标准下的函数接口，其他函数及其实现细节请参考：[CMSIS-RTOS2](https://arm-software.github.io/CMSIS_5/develop/RTOS2/html/rtos_api2.html)

## 任务管理

```C
osThreadId_t osThreadNew(osThreadFunc_t func, void *argument, const osThreadAttr_t *attr);    //创建任务
osThreadId_t osThreadGetId(void);    //返回当前任务ID
osStatus_t osThreadSuspend(osThreadId_t thread_id);    //挂起任务
osStatus_t osThreadResume(osThreadId_t thread_id);    //恢复任务
osStatus_t osThreadYield(void);    //主动让出资源
void osThreadExit(void)    //终止当前任务
osStatus_t osThreadTerminate(osThreadId_t thread_id);    //终止其他任务
osStatus_t osDelay(uint32_t ticks);    //延时ticks时钟周期

```

## 消息队列

定义：

```C
typedef struct {
    const char *name;    //名字
    uint32_t attr_bits;  //FreeRTOS未使用
    void *cb_mem;        //控制块（静态创建）地址
    uint32_t cb_size;    //控制块（静态创建）大小
    void *mq_mem;        //数据存储地址
    uint32_t mq_size;    //数据存储大小
}osMessageQueueAttr_t;
```

常用函数：

```C
osMessageQueueId_t osMessageQueueNew(uint32_t msg_count, uint32_t msg_size, const osMessageQueueAttr_t *attr);    //创建
osStatus_t osMessageQueueDelete(osMessageQueueId_t mq_id);    //删除
osStatus_t osMessageQueuePut(osMessageQueueId_t mq_id, const void *msg_ptr, uint8_t msg_prio, uint32_t timeout);    //发送消息
osStatus_t osMessageQueueGet(osMessageQueueId_t mq_id, void *msg_ptr, uint8_t *msg_prio, uint32_t timeout);    //获取消息
osStatus_t osMessageQueueReset(osMessageQueueId_t mq_id);    //重置消息队列
```

## 信号量

定义：

```C
typedef struct {
    const char *name;   //名字
    uint32_t attr_bits; //FreeRTOS未使用
    void *cb_mem;       //控制块（静态创建）地址
    uint32_t cb_size;   //控制块（静态创建）大小
}osSemaphoreAttr_t;
```

常用函数：

```C
osSemaphoreId_t osSemaphoreNew(uint32_t max_count, uint32_t initial_count, const osSemaphoreAttr_t *attr);    //创建
osStatus_t osSemaphoreAcquire(osSemaphoreId_t semaphore_id, uint32_t timeout);    //获取
osStatus_t osSemaphoreRelease(osSemaphoreId_t semaphore_id);    //释放
uint32_t osSemaphoreGetCount(osSemaphoreID_t semaphore_id);    //获取计数量
osStatus_t osSemaphoreDelete(osSemaphoreId_t semaphore_id);    //删除
```

## 互斥量

定义：

```C
typedef struct {
    const char *name;    //名字
    uint32_t attr_bits;  //互斥量类型
    void *cb_mem;        //控制块（静态创建）内存 
    uint32_t cb_size;    //控制块（静态创建）大小
}osMutexAttr_t;
```

常用函数：

```C
osMutexId_t osMutexNew(const osMutexAttr_t *attr);    //创建
osStatus_t osMutexAcquire(osMutexId_t mutex_id, uint32_t timeout);    //获取
osStatus_t osMutexRelease(osMutexId_t mutex_id);    //释放
osStatus_t osMutexDelete(osMutexId_t mutex_id);    //删除
```

## 事件

定义：

```C
typedef struct {
    const char *name;    //名字
    uint32_t attr_bits;  //FreeRTOS未使用
    void *cb_mem;        //控制块（静态创建）内存
    uint32_t cb_size;    //控制块（静态创建）大小
}osEventFlagsAttr_t;
```

常用函数：

```C
osEventFlagsId_t osEventFlagsNew(const osEventFlagsAttr_t *attr);    //创建
uint32_t osEventFlagsSet(osEventFlagsId_t ef_id, uint32_t flags);    //设置位
uint32_t osEventFlagsClear(osEventFlagsId_t ef_id, uint32_t flags);    //清空位
uint32_t osEventFlagsWait(osEventFlagsId_t ef_id, uint32_t flags, uint32_t options, uint32_t timeout);    //等待位
osStatus_t osEventFlagsDelete(osEventFlagsId_t ef_id);    //删除
```

## 任务通知

CMSIS标准没有提供任务通知的API，属于FreeRTOS的特性。

## 定时器

定义：

```C
typedef struct {
    const char *name;    //名字
    uint32_t attr_bits;  //FreeRTOS未使用
    void *cb_mem;        //控制块（静态创建）内存
    uint32_t cb_size;    //控制块（静态创建）大小
}osTimerAttr_t;
```

常用函数：

```C
osTimerId_t osTimerNew(osTimerFunc_t func, osTimerType_t type, void *argument, const osTimerAttr_t *attr);    //创建
osStatus_t osTimerStart(osTimerId_t timer_id, uint32_t ticks);    //启动
osStatus_t osTimerStop(osTimerId_t timer_id);   //停止
osStatus_t osTimerDelete(osTimerId_t timer_id);    //删除
```

## 内存管理

常用函数：

```C
osMemoryPoolId_t osMemoryPoolNew(uint32_t block_count, unit32_t block_size, const osMemoryPoolAttr_t *attr);    //创建
void *osMemoryPoolAlloc(osMemoryPoolId_t mp_id, unit32_t timeout);    //分配
osStatut_t osMemoryPoolFree(osMemoryPoolId_t mp_id, void *block);    //释放
osStatus_t osMemoryPoolDelete(osMemoryPoolId_t mp_id);    //删除
```

