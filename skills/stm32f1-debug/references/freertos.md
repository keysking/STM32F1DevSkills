# FreeRTOS 运行问题

先识别原生/CMSIS v1/v2、内核版本、端口、heap、实际 FreeRTOSConfig.h 和固件。API 单位见[FreeRTOS 开发](../../stm32f1-firmware/references/freertos.md)，不要为排查新增另一套内核或适配层。

## 不进入任务或任务停止

检查创建返回值和调度器启动次数。若启动后返回，核对 idle/timer 任务分配、heap 和端口。确认 SVC、PendSV、SysTick 实际向量进入唯一正确处理函数，没有被其他源的同名实现覆盖。

任务“没有运行”要区分 Ready、Blocked、Suspended、Deleted、被高优先级任务饿死以及 CPU 进入 Fault。记录当前 PC、任务句柄和等待对象；高优先级循环没有阻塞点时，不用随意提高其他任务优先级掩盖问题。

## 栈、堆与断言

核对创建栈的单位，v1 当前适配层、v2 字节和原生栈字不同。按项目允许方式观察 `uxTaskGetStackHighWaterMark()`、`xPortGetFreeHeapSize()` 及所选 heap 支持的最低空闲指标；高水位通常以栈字计，未覆盖峰值路径时不能当作绝对安全证明。

启用已有或合理新增的 configASSERT、栈溢出和 malloc 失败钩子，记录触发位置而非在钩子里复杂 printf。task PSP、异常 MSP、FreeRTOS 堆与 C 库堆分开检查。栈溢出可先破坏对象再于别处 HardFault，结合[异常帧](startup-faults.md)定位。

## ISR、队列与锁

断言发生在 FromISR 时优先查 IRQ 数值优先级和 syscall 阈值、分组及 API 是否 ISR-safe。普通任务 API 不能替代 FromISR；互斥锁不能从 ISR 获取。确认唤醒标志和请求调度流程。

任务卡在队列/信号量时记录对象、生产者、消费者、超时单位和所有权。传指针的队列检查生命周期；DMA 回调交出的缓冲不能立即被 DMA 重用而消费者尚未读取。队列满/空时要有明确策略，不把所有错误无限重试。

使用 CubeF1 CMSIS v1 时另核对[实际适配层行为](../../stm32f1-firmware/references/freertos-cmsis-v1.md)：当前版本 MessagePut 的 0 超时在任务路径仍转为1 tick；SemaphoreWait 成功返回 osOK（0），动态/静态二值信号量的初始状态也不同。不能直接按通用 CMSIS 描述判断失败或重复补发事件。

死锁排查锁顺序、递归需求、持锁等待外设/其他任务以及异常退出后的释放。优先级继承仅对适当互斥机制生效，不能解决循环锁依赖。

## 时间基准与调试影响

核对 RTOS tick 与 HAL tick，尤其应用覆盖 PeriodElapsed 回调后是否遗漏 HAL_IncTick。v1 osDelay 毫秒、v2 osDelay tick、原生 vTaskDelay tick，不能按名字认定相同时间。低功耗恢复需同步 HAL、OS 和实际时钟。

调试器暂停可能触发外设超时或看门狗；支持 RTOS 线程视图需要客户端相应能力，不能把普通 GDB 的线程列表自动当任务列表。必要时用内核允许的任务状态 API 生成有限快照，避免在运行中随意遍历不断变化的内核链表。

修复后验证创建、调度、阻塞/唤醒、ISR 交接、负载峰值及恢复路径。仅链接通过不能证明调度或任务栈预算正确。
