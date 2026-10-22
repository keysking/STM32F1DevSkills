# CubeF1 CMSIS-RTOS v2

仅用于 `cmsis_os2.h` / `osThreadNew` 工程。各节按需求选择，不重建已经存在的对象。以下变量与 Message 类型由应用定义，包含的结构体/指针必须具有符合使用期的生命周期。

## 任务和时间单位

worker 签名为 `void worker(void *argument)`。需要新增动态任务时：

```c
const osThreadAttr_t attr = {
    .name = name,
    .stack_size = stack_bytes,
    .priority = osPriorityNormal
};
osThreadId_t id = osThreadNew(worker, context, &attr);
```

id 为 NULL 表示创建失败。stack_size 单位为字节；属性按实际栈/优先级预算选择。静态创建需按适配层规则提供 cb_mem/cb_size、stack_mem/stack_size，不把栈缓冲放在退出后失效的局部数组里。由已有初始化顺序调用 osKernelInitialize、创建对象、osKernelStart，不重复启动调度器。

v2 等待参数和 `osDelay()` 使用 tick。需要把非零毫秒向上换算为至少一个 tick 时，可用 `(uint64_t)ms * osKernelGetTickFreq()` 加 999 后除以 1000，并检查结果小于 osWaitForever 和目标类型范围；0 明确表示不等待，不一律改成 1。周期执行可用 osDelayUntil，处理超期返回。

## 复制消息

需要按值传结构体时：

```c
osMessageQueueId_t queue = osMessageQueueNew(depth, sizeof(Message), NULL);
```

确认非 NULL 后，生产者/消费者分别调用：

```c
osStatus_t sent = osMessageQueuePut(queue, &tx_message, 0, wait_ticks);
osStatus_t received = osMessageQueueGet(queue, &rx_message, NULL, wait_ticks);
```

osOK 表示成功；非阻塞无资源和等待超时分别检查 osErrorResource、osErrorTimeout 等。按所建 msg_size 复制数据；包含指针时不延长其目标对象的生命周期。此 FreeRTOS 适配层忽略 msg_prio，不用它实现消息优先级。

ISR 可用 Put/Get，但 timeout 必须为 0，IRQ 优先级须满足 FreeRTOS syscall 阈值；包装层自行完成 FromISR/yield。不能在 ISR 创建/删除对象或执行阻塞等待。

## Mutex、信号量与事件

需要任务互斥时用 `osMutexNew(&mutex_attr)`，按需求在 attr_bits 中设置 `osMutexPrioInherit`，需递归才加 `osMutexRecursive`；检查适配层是否支持所选属性。Acquire==osOK 后才访问资源并 Release。Mutex 不用于 ISR，不能无条件解锁未取得的 mutex。

事件计数需要信号量时：

```c
osSemaphoreId_t done = osSemaphoreNew(max_count, initial_count, NULL);
```

确认非 NULL 后任务用 `osSemaphoreAcquire(done, wait_ticks)`，生产者用 `osSemaphoreRelease(done)`。初值0适合等待首次事件；max_count由突发事件预算确定。Release返回osErrorResource表示已到上限，不能忽略事件丢失。ISR只调用允许的Release或timeout=0的Acquire。

单个任务事件位可用 `osThreadFlagsSet(id, mask)`，目标任务 `osThreadFlagsWait(mask, osFlagsWaitAny, wait_ticks)`。检查返回值是否含 osFlagsError，成功返回满足条件的标志；默认会清除已等待标志，需要保留才加 osFlagsNoClear。ISR 可 Set，不能 Wait。该实现使用 task notification，同一任务不混入另一套原生通知协议。

需要多个等待者或共享事件对象时，考虑 `osEventFlagsNew/Set/Wait`；核对内核 timer/daemon 配置、ISR Set 的延迟及队列容量，不保证它与 ThreadFlags 在 ISR 中有相同执行时序。多次相同位会合并，逐次计数采用队列/信号量。

验证内存不足、等待超时、事件合并、队列满及 ISR 边界。依据为工程 CMSIS_RTOS_V2/cmsis_os2.c 和[CMSIS-RTOS2 API](https://arm-software.github.io/CMSIS_6/latest/RTOS2/index.html)，包装层的实际支持范围优先。
