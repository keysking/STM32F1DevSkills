# FreeRTOS 应用 API 与资源

只在项目需要 FreeRTOS 时使用。已有工程保留原生 API、CMSIS-RTOS v1 或 v2；不要为了新增一个任务更换内核/适配层。生成配置见[FreeRTOS 配置](../../stm32f1-configure/references/freertos.md)。

## API 对照与单位

| 操作 | 原生 FreeRTOS | CMSIS v1 | CMSIS v2 |
| --- | --- | --- | --- |
| 创建任务 | `xTaskCreate` / `xTaskCreateStatic` | `osThreadDef` + `osThreadCreate` | `osThreadNew` + `osThreadAttr_t` |
| 延时 | `vTaskDelay`，单位 tick | `osDelay`，参数毫秒 | `osDelay`，单位内核 tick |
| 队列 | `xQueueCreate/Send/Receive` | `osMessageQDef/Create/Put/Get` | `osMessageQueueNew/Put/Get` |
| 互斥 | `xSemaphoreCreateMutex/Take/Give` | `osMutexCreate/Wait/Release` | `osMutexNew/Acquire/Release` |
| ISR 通知 | 对应 `...FromISR` | 仅允许的 ISR API | 仅允许的 ISR API，超时通常必须为 0 |

FreeRTOS 原生任务栈深度以 StackType_t 元素计，F1 ARM_CM3 通常每项 4 字节。当前 CubeF1 CMSIS v1 适配层把任务定义栈大小直接传给 xTaskCreate，因此实际按栈字使用；不能只按通用 CMSIS 文档推断。CMSIS v2 `stack_size` 是字节。升级适配层时重新检查实现，生成器任务字段也要核对转换。

`pdMS_TO_TICKS(ms)` 用于原生毫秒转换，转换后可能为 0；周期任务考虑 `vTaskDelayUntil`。v1 队列包装有消息宽度/指针约定，不能把任意结构体按 v2 `msg_size` 的用法直接传入；先查适配层，需要传结构体时选择原生队列或明确生命周期的对象指针。

## 选择性调用形式

需要新建原生任务时，形态如下，变量均由应用定义：

```c
BaseType_t rc = xTaskCreate(worker, name, stack_words, context,
                           priority, &task_handle);
if (rc != pdPASS) { /* 应用处理内存或创建失败 */ }
```

需要 CMSIS v2 任务时，形态为：

```c
const osThreadAttr_t attr = {
    .name = name,
    .stack_size = stack_bytes,
    .priority = osPriorityNormal
};
osThreadId_t id = osThreadNew(worker, context, &attr);
```

worker 的签名匹配所选 API。两段是不同创建方式，不创建两份相同任务；优先级和栈预算来自实际工作量。检查创建结果，不让自动存储期参数在任务开始后失效。调度器只启动一次，不能同时调用 `osKernelStart()` 和 `vTaskStartScheduler()`。

## 中断、锁与时间基准

原生 ISR 路径使用 `BaseType_t higher_priority_task_woken = pdFALSE`，调用允许的 FromISR API，再按当前端口执行 `portYIELD_FROM_ISR(...)`。IRQ 数值优先级必须满足 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`，数值更小代表更高紧迫度；检查优先级分组与未移位/已移位阈值的区别。

互斥锁不用于 ISR；ISR 只做有限处理并通知任务。任务间避免持锁执行无界外设等待，必要时设置超时与恢复所有者。忙等高优先级任务会饿死低优先级任务，不能靠加大堆解决。

核对 SysTick、PendSV、SVC 所有者及独立 HAL timebase。HAL 回调若兼顾时间基准，必须保留 HAL_IncTick 条件；任务中优先选择 OS 延时，不用 HAL_Delay 形成 CPU 忙等。

## 内存与验证

仅编译一个 `portable/GCC/ARM_CM3` 端口和一个 heap 实现。heap_4 的 ucHeap、C 库 malloc 堆、任务栈和 MSP 不同；map 的静态占用不能证明各任务栈足够。启用项目允许的断言、malloc 失败及栈溢出钩子，检查创建返回值、空闲堆和栈高水位，并给峰值路径留余量。

实机检查任务运行/阻塞、队列边界、锁竞争、ISR 交接和故障恢复。诊断见[任务故障](../../stm32f1-debug/references/freertos.md)。依据为项目 `FreeRTOSConfig.h`、kernel/task.h 和实际 CMSIS 适配层源码。
