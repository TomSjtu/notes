# CMSIS标准

Cortex系列的芯片内核是相同的，但是不同的厂商在生产的时候外设会有区别，这些差异导致软件在同内核，不同外设的芯片上移植困难。为了解决不同芯片厂商生产的兼容性问题，ARM与各芯片厂商建议了CMSIS标准。

所谓的CMSIS标准，其实就是新建了一个软件抽象层，将不同的RTOS的接口统一起来，提供CMSIS的接口供使用者使用。

本文只简单列举了部分CMSIS标准下的函数接口，其他函数及其实现细节请参考：[CMSIS-RTOS2](https://arm-software.github.io/CMSIS_5/develop/RTOS2/html/rtos_api2.html)

## 任务管理

```C
osThreadId_t osThreadGetId(void); //返回当前线程ID
osStatus_t osThreadSuspend(osThreadId_t thread_id); //挂起任务
osStatus_t osThreadResume(osThreadId_t thread_id); //恢复任务
void osThreadExit(void) //终止当前任务
osStatus_t osThreadTerminate(osThreadId_t thread_id); //终止其他任务
osStatus_t osDelay(uint32_t ticks); //延时ticks时钟周期

```