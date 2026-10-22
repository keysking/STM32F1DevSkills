---
name: stm32f1-firmware
description: 为 STM32F1 CMake 工程实现应用、移植驱动或维护 FreeRTOS 任务，按实际传统 HAL、LL、原生 FreeRTOS 或 CMSIS-RTOS 接口开发。用于功能实现、异步通信、应用集成与代码维护。
license: MIT
---

# STM32F1 固件

先识别准确型号、HAL/LL 或旧库、裸机/FreeRTOS、初始化所有者和 CMake 源码目标。F1 是 Cortex-M3，传统 HAL 使用 `UART_HandleTypeDef` 等句柄；不套用 HAL2 API、FPU 或 TrustZone 模型。

按当前任务读取：

- HAL/LL、初始化、回调与异步缓冲区：[驱动与生命周期](references/hal-ll.md)。
- 已选 LL，需要 GPIO、USART、IRQ/DMA API：[LL 操作](references/ll-operations.md)。
- 应用模块、第三方驱动、SPL/寄存器代码接入：[移植与集成](references/porting.md)。
- 任务、队列、同步、原生/CMSIS v1/v2：[FreeRTOS API](references/freertos.md)。
- 片内 Flash、参数存储、bootloader 分区：[存储与启动布局](references/storage.md)。

外设配置确实需要变更时，读取[配置技能](../stm32f1-configure/SKILL.md)中该外设的单份参考；已有初始化满足需求时直接实现应用。用户代码放在自有源文件或对应 USER CODE 区，通过[用户维护的 CMake 目标](../stm32f1-build/references/cmake.md)加入构建。

按需求选择轮询、IT 或 DMA，不并列启动互斥的传输。检查返回值、所有权、超时和错误恢复；ISR 不加入阻塞延时。增加 RTOS 任务不自动修改已有 tick、堆、API 层或调度架构。

完成至少检查编译链接、调用入口与关键错误路径；需要功能证明时列出并执行任务范围内可用的板端检查。未运行的功能明确标为构建检查，不以接口调用存在证明运行成功。
