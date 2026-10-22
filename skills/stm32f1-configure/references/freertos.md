# FreeRTOS 生成配置

仅在用户选择 FreeRTOS 或已有工程使用它时读取/修改。裸机请求不自动加 RTOS；给已有系统增加任务通常不需要重设内核、tick、堆或 API 层。

## 识别并选择接口

先查 `FreeRTOSConfig.h`、`cmsis_os.h`/`cmsis_os2.h`、启动调用和 CMake 源文件。已有接口优先保留。新建 RTOS 工程根据需求选择 CMSIS v1/v2；原生 xTask/queue API 可与选定适配层共同使用，但内核只启动一次。

当前 CubeMX 选择形式分别为：

```text
set mode FREERTOS CMSIS_V1
```

或：

```text
set mode FREERTOS CMSIS_V2
```

两者二选一，不连续执行。`get mode available FREERTOS` 可能 KO；`set mode FREERTOS Native` 不能据此假设可用。需要纯原生工程时沿用其 CMake/FreeRTOS 初始化，或在生成的兼容结构中明确内核所有者，不靠虚构的生成器模式。

## 查询内核参数

```text
get ip parameters FREERTOS configTOTAL_HEAP_SIZE
get ip parameters FREERTOS configTICK_RATE_HZ
get ip parameters FREERTOS configUSE_PREEMPTION
get ip parameters FREERTOS Tasks01
```

字段以实际数据库为准；Tasks01 是结构化序列字段，不能用不明顺序的 CSV 覆盖已有全部任务。无法可靠编辑复杂任务字段时，保留生成配置，在用户维护的 freertos 用户区/应用文件创建所需任务。

只有需求和预算要求调整堆时，才用 `set ip parameters FREERTOS configTOTAL_HEAP_SIZE <实际字节数>` 并检查生成配置。堆要容纳所选 heap_x 的任务栈/对象及碎片预算，不能以 C8 有 20 KB SRAM 就把全部空间分给 RTOS。

## HAL 时间基准与中断

若 SysTick 被 RTOS 使用，按当前适配层选择独立 HAL 时间基准。先检查 TIM 占用；只有 TIM4 空闲且适合本工程时，`set mode SYS TIM4` 是选择形式，不是所有工程默认。

生成后核对 HAL timebase 源文件、实际 timer IRQ、PeriodElapsed 中 HAL_IncTick 条件，以及 FreeRTOS 的 SysTick/PendSV/SVC 向量。避免应用覆盖回调后使 HAL tick 停止，或多个 timebase 源同时编译。

调用 FromISR 的 IRQ 按实际优先级阈值和分组设置，见[DMA/NVIC](dma-irq.md)。CMSIS API 也有 ISR 限制，不因包装层就允许任意优先级调用。

## 完成检查

保存重载，生成并检查：内核真实版本、唯一 `portable/GCC/ARM_CM3` 端口、唯一 heap 实现、唯一 CMSIS 适配层、任务创建与 scheduler 启动、HAL/OS 时间基准。数据库 `v8.0.0_Cube` 字样不是内核版本。

构建后验证任务是否创建成功、运行、阻塞/唤醒及 ISR 交接；API 与栈单位见[FreeRTOS 开发](../../stm32f1-firmware/references/freertos.md)。
