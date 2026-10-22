---
name: stm32f1-debug
description: 用 GDB、ST-Link 和板端证据诊断 STM32F1 CMake 固件的启动失败、HardFault、中断/DMA、外设和 FreeRTOS 运行异常。用于运行故障定位、寄存器取证与修复验证。
license: MIT
---

# STM32F1 调试

先记录可复现现象、实际板卡与接线、完整型号、固件版本和本次 ELF。保留正在运行的状态；决定是附加取证还是从复位重现，不先执行 `load`、reset 或擦除来试运气。

按问题读取：

| 问题 | 资料 |
| --- | --- |
| 建立 GDB/ST-Link 会话、寄存器和内存 | [GDB 操作](references/gdb.md) |
| 不进 main、HardFault、反复复位 | [启动与异常](references/startup-faults.md) |
| GPIO、时钟、IRQ、DMA、UART/I2C/SPI、TIM/ADC、CAN/USB | [外设诊断](references/peripherals.md) |
| 任务不运行、死锁、堆栈、优先级 | [FreeRTOS 诊断](references/freertos.md) |

F1 为 Cortex-M3。异常分析按基本异常栈帧、MSP/PSP 和 SCB 状态，不能照搬浮点扩展栈帧或安全域信息。寄存器读取也有副作用，FIFO/状态寄存器按手册读取，不连续扫描整个外设空间。

用证据缩小到配置、启动链、驱动状态或应用所有权后再修复；配置源归 CubeMX 的改动同时维护 IOC，防止重生成回退。完成时给出根因证据、最小修复、构建结果及同一故障条件下的复测结果。没有板端证据时明确尚不能确认的部分。
