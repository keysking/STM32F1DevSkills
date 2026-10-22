---
name: stm32f1-configure
description: 用传统 CubeMX 查询和配置 STM32F1 的引脚、时钟、外设、DMA、NVIC 与 FreeRTOS，处理保存重载、生成和配置不生效。用于 F1 CMake 工程的硬件配置调整及现有 IOC 查询。
license: MIT
---

# STM32F1 配置

先确认准确料号、板级接线、时钟来源和实际 `.ioc`。命令是按场景选择的参考，不是整篇执行清单；引脚、频率、实例和参数值来自本次需求与实际工程。GPIO 输出不隐含 EXTI/LL，阻塞通信不隐含 DMA，查询不修改配置。

先读[操作基础](references/operations.md)，其中给出会话上下文、能力查询、参数枚举、失败判断和保存重载。工具不明时读[CubeMX 启动](../stm32f1-env/references/mx-cli.md)。

只读取当前任务对应的文件：

| 任务 | 资料 |
| --- | --- |
| 输入、输出、EXTI、SWD 与 AFIO | [GPIO/EXTI](references/gpio-exti.md) |
| HSI/HSE、PLL、总线、ADC/USB 时钟 | [时钟](references/clocks.md) |
| UART 轮询、IT、DMA、IDLE | [UART](references/uart.md) |
| I2C 或 SPI | [I2C/SPI](references/i2c-spi.md) |
| 定时、PWM、输入捕获、编码器 | [TIM](references/timers.md) |
| ADC、校准、多通道与 DMA | [ADC](references/adc.md) |
| DMA 通道分配和中断链 | [DMA/NVIC](references/dma-irq.md) |
| FreeRTOS 生成配置和 HAL 时间基准 | [FreeRTOS 配置](references/freertos.md) |
| bxCAN、USB Device/CDC | [CAN/USB](references/can-usb.md) |
| RTC、备份域、日期保持 | [RTC](references/rtc.md) |
| IWDG、WWDG、STOP 唤醒 | [看门狗与低功耗](references/watchdogs-power.md) |

修改只覆盖需求项及必要依赖。检查被改动的旧引脚、重映射组、共享 DMA 通道和 IRQ；不得为解决资源冲突默默关闭 SWD。修改后回读、保存、独立重载；需要同步固件时再[生成并检查差异](../stm32f1-project/references/regenerate.md)。退出码 0 或单条 OK 不能代替完整日志和生成结果。

无生成器 CMake 工程在其现有初始化代码中修改，使用对应 HAL/LL 头文件和本页的硬件约束，不强行创建 IOC。完成时分别说明配置、生成、构建和板端信号验证状态。
