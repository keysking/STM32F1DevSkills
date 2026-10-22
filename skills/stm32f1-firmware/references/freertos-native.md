# FreeRTOS 原生任务与通信

仅当工程使用原生 API，或已明确允许在 CMSIS 适配层下调用原生接口时读取。本页各节独立，变量由真实应用提供；不为演示创建额外任务/队列。先读 `FreeRTOSConfig.h`、`task.h`、`queue.h`、`semphr.h`，确认动态/静态分配及所用功能已启用。

## 创建与周期执行

需要动态任务时，worker 签名为 `void worker(void *context)`，只能循环处理或 `vTaskDelete(NULL)`，不能普通 return。context 指向的数据须活到任务不再使用它。

```c
TaskHandle_t task = NULL;
BaseType_t rc = xTaskCreate(worker, name, stack_words, context, priority, &task);
```

`rc == pdPASS` 才能使用 task；stack_words 是 StackType_t 元素数，ARM_CM3 每项 4 字节，priority 小于 `configMAX_PRIORITIES`。静态分配使用 `xTaskCreateStatic()`，应用提供长期有效的栈数组及 StaticTask_t，不额外增加动态堆。沿用现有唯一调度器启动入口。

需要固定周期、且 period_ms 已由业务确定时：

```c
TickType_t period = pdMS_TO_TICKS(period_ms);
configASSERT(period > 0);
TickType_t last = xTaskGetTickCount();
/* 放在任务循环中，每轮完成有限工作后调用： */
vTaskDelayUntil(&last, period);
```

周期小于 tick 分辨率时重新设计时间源，不把 0 tick 当作延时。超期执行会影响调度，核对最坏执行时间。

## 队列复制数据

Message 是应用定义的消息类型。需要队列时，先创建，再在所有生产者/消费者中使用同一个有效句柄：

```c
QueueHandle_t queue = xQueueCreate(depth, sizeof(Message));
```

返回 NULL 表示未创建。任务上下文分别选择发送或接收，wait_ticks 是允许阻塞的 tick 数，0 为不等待：

```c
BaseType_t sent = xQueueSend(queue, &tx_message, wait_ticks);
BaseType_t received = xQueueReceive(queue, &rx_message, wait_ticks);
```

成功为 pdPASS；队列复制 `sizeof(Message)` 字节。若 Message 含指针，仅复制指针，不复制所指对象；缓冲区释放/复用必须等消费者完成。明确队列满时丢弃、背压或计数策略，不能默认消息一定进入。

只有在已获准调用 RTOS 的 ISR 中发送时，替换任务发送调用为：

```c
BaseType_t woken = pdFALSE;
BaseType_t sent = xQueueSendFromISR(queue, &tx_message, &woken);
portYIELD_FROM_ISR(woken);
```

IRQ 优先级必须满足项目 syscall 阈值；FromISR 无阻塞等待，队列满也要处理。不要在同一事件中再发送一遍任务版 API。

## 互斥与事件计数

保护任务共享资源时使用 `xSemaphoreCreateMutex()` → `xSemaphoreTake(mutex, wait_ticks)` → 执行有界临界操作 → `xSemaphoreGive(mutex)`。只有 Take 返回 pdTRUE 后才访问并释放；普通 mutex 不可递归获取，不在 ISR 获取/释放。需要递归语义时使用对应 Recursive 系列并核对配置。

单次完成事件可用 `xSemaphoreCreateBinary()`，初始为空；任务等待 `xSemaphoreTake(event, wait_ticks)`，ISR 调用 `xSemaphoreGiveFromISR(event, &woken)` 后按需 yield。它至多保留一个 token，多次事件会合并。每次事件都需计数时改用 counting semaphore/队列并考虑上限。

只有单个目标任务、事件无需携带队列数据时，可用任务通知降低开销。ISR 通知形式：

```c
BaseType_t woken = pdFALSE;
vTaskNotifyGiveFromISR(task, &woken);
portYIELD_FROM_ISR(woken);
```

目标任务以 `ulTaskNotifyTake(pdTRUE, wait_ticks)` 读取累计计数并清零；需要每次只减一时选 pdFALSE。0 表示超时；检查计数溢出及事件合并需求。CubeF1 1.8.7 的内核使用单个通知槽，不套用新版本 indexed API；CMSIS v1 signal/v2 thread flags 也占用它，同一任务不能独立混用这些协议。

验证创建失败、超时、队列满、突发 ISR 与退出清理；对象销毁前停止所有使用者。构建只能确认类型/符号，调度和竞态需运行验证。依据：[FreeRTOS Kernel](https://github.com/FreeRTOS/FreeRTOS-Kernel)，实际工程内核版本优先。
