# CubeF1 CMSIS-RTOS v1

仅用于包含 `cmsis_os.h`、使用 `osThreadDef/osThreadCreate` 的工程。保持已有生成任务和唯一 osKernelStart；不同时加入 v2 适配层。本页依据 CubeF1 1.8.7 的 `CMSIS_RTOS/cmsis_os.c`，其若干行为与通用 CMSIS 描述不同，升级或替换适配层后重新检查。

## 任务

只有需要新增 worker 任务时，定义其入口及描述；名字、优先级和栈值由实际业务选择。下列 256 是栈字数的语法实例，不是推荐预算：

```c
void worker(void const *argument);
osThreadDef(worker, worker, osPriorityNormal, 0, 256);
```

初始化阶段创建，并保存长期使用的句柄：

```c
osThreadId id = osThreadCreate(osThread(worker), context);
```

NULL 表示失败。这个 ST 适配层直接把 stacksize 传给 xTaskCreate，F1 上 256 对应 1024 字节；不能按通用文档把它认成 256 字节。worker 参数数据须长期有效，任务不能普通 return。v1 `osDelay(delay_ms)` 和等待参数单位为毫秒，不是 v2 的 tick。

## 32 位消息队列

需要传递 32 位值时，在定义区声明队列；16 只是深度语法实例，按容量/内存预算替换：

```c
osMessageQDef(events, 16, uint32_t);
```

创建与任务收发形态分别为：

```c
osMessageQId queue = osMessageCreate(osMessageQ(events), NULL);
```

```c
osStatus sent = osMessagePut(queue, value, timeout_ms);
osEvent event = osMessageGet(queue, timeout_ms);
if (event.status == osEventMessage) {
    uint32_t received_value = event.value.v;
    /* 使用 received_value；其他状态按超时/错误处理。 */
}
```

只有创建非 NULL 后才收发；sent==osOK 表示入队。不要把任意结构体替换 uint32_t：本适配层 Put 的参数是 uint32_t，实际从该值地址复制队列 item_size 字节，结构体大小会越界。需要结构体时选原生复制队列；传对象指针时 F1 指针宽 32 位，但仍须管理对象所有权和释放时机。

**本版本任务路径 `osMessagePut(..., 0)` 仍会等待至少一个 tick**，因为实现把 0 向上调整为 1。严格不阻塞时，可在确认此适配层句柄就是 FreeRTOS QueueHandle_t 后使用 `xQueueSend((QueueHandle_t)queue, &value, 0)`；引入 queue.h，且仍按同一个 uint32_t 消息协议。ISR 路径用支持 ISR 的 osMessagePut 且 timeout=0；适配层内部调用 FromISR 和 yield，不再额外调用任务发送。优先级仍须符合 FreeRTOS 阈值。

## Mutex 与 semaphore

任务互斥：定义 `osMutexDef(bus)`，创建 `osMutexCreate(osMutex(bus))`，成功后 `osMutexWait(mutex, timeout_ms)` → 访问资源 → `osMutexRelease(mutex)`。Wait==osOK 才能进入；不能在 ISR 使用 mutex。

事件信号量：定义 `osSemaphoreDef(done)`，按需求调用 `osSemaphoreCreate(osSemaphore(done), 1)`。此 SDK 动态路径使用 vSemaphoreCreateBinary，初始有一个 token；静态路径使用 xSemaphoreCreateBinaryStatic，初始为空。需要“首次事件前必须等待”时，在启用生产者/IRQ 之前用 `osSemaphoreWait(done, 0)` 消耗可能已有的 token，之后再开始接收真实事件。

此 SDK 的 `osSemaphoreWait()` **成功返回 osOK（0）**，失败返回 osError*，不是剩余 token 数；不能用 `>0` 或 `>=0` 判断成功。运行期任务等待成功后处理事件；ISR 以 `osSemaphoreRelease(done)` 通知，由包装层进行 FromISR/yield。二值信号量会合并多次事件，需要完整计数时选择计数信号量或队列。

## Signal

需要目标任务事件位时用 `osSignalSet(id, mask)`，任务用 `osSignalWait(mask, timeout_ms)` 并检查 event.status==osEventSignal 和 event.value.signals。mask 使用有效用户位，不能占用错误标志位。Signal 基于原生 task notification；同一任务不另用原生通知做独立计数。ISR 只调用适配层明确支持的 SignalSet 等接口，等待只放在任务中。

按实际工程检查生成对象是否已经存在，避免重复创建；重点验证包装层超时/返回值与初始化状态。源码依据为项目 CubeF1 的 CMSIS_RTOS 目录及[官方 CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)。
