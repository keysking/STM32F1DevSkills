# 外设、DMA 与中断诊断

先固定实际 ELF、硬件连接、输入和现象。对照链路逐层取证：需求 → IOC → 生成初始化 → 启动调用 → 寄存器/IRQ → 缓冲与应用 → 外部信号。某层正确不能代替后续层验证。

## 共用检查

- 修改未生效：确认加载/保存的是实际 IOC，执行独立重载查询；核对生成输出目录、CMake 源路径、本次 ELF 和板上版本。
- 无中断：检查外设使能位、状态标志、NVIC enable/priority、PRIMASK/BASEPRI、向量符号及实际 HAL handler。shared IRQ 必须服务需要的来源。
- 无 DMA：核对固定通道映射、冲突、DMA 时钟顺序、方向、对齐、递增、计数和 HAL link；观察 CNDTR/状态标志，区分未启动、已完成和错误。
- 无回调：初始化不等于启动；检查 API 返回值、IRQ 路由、回调签名和实例分派。循环 DMA 不应每次回调重启。

## 按外设定位

| 现象 | 重点证据与相应参考 |
| --- | --- |
| GPIO 电平/EXTI 不符 | CRL/CRH、ODR/IDR、外部负载、AFIO/EXTI 映射、共享线和边沿；[GPIO](../../stm32f1-configure/references/gpio-exti.md) |
| 波特率/PWM 频率不符 | RCC 实际时钟、APB 分频、定时器倍频、PSC/ARR/BRR；[时钟](../../stm32f1-configure/references/clocks.md) |
| UART 丢帧/只收一次 | ORE/FE/NE、IRQ、接收挂接状态、IDLE/HT/TC 位置、处理速度；[UART](../../stm32f1-configure/references/uart.md) |
| I2C BUSY/NACK | 实际地址及左移规则、上拉、电压、SDA/SCL 波形、对端状态；[I2C/SPI](../../stm32f1-configure/references/i2c-spi.md) |
| SPI 数据错位 | CPOL/CPHA、位宽/顺序、CS 时序、收发缓冲和 OVR；[I2C/SPI](../../stm32f1-configure/references/i2c-spi.md) |
| TIM 无输出/捕获 | GPIO 复用、MSP PostInit、通道 Start、CCR、共享 PSC/ARR、实际信号；[TIM](../../stm32f1-configure/references/timers.md) |
| ADC 值异常/不更新 | 模拟引脚、参考电压、ADC 时钟、校准、采样时间、触发/序列/DMA；F1 regular 多 rank 中断只得到末次转换，不能在回调循环读 DR 恢复全部数据；[ADC](../../stm32f1-configure/references/adc.md) |
| CAN 无 ACK/Bus-off | 收发器、端接、位时序、过滤、启动/通知、错误计数；[CAN](../../stm32f1-configure/references/can-usb.md) |
| USB 不枚举/BUSY | 48 MHz 与精度、D+/D−、上拉、IRQ、类注册、配置状态与缓冲所有权；[USB](../../stm32f1-configure/references/can-usb.md) |
| RTC 重启丢时间 | VBAT、备份复位、无条件 SetTime/SetDate、软件日期状态；[RTC](../../stm32f1-configure/references/rtc.md) |
| STOP 后卡住 | HSI/PLL 恢复、HAL 超时基准、OS tick 和外设时钟；[低功耗](../../stm32f1-configure/references/watchdogs-power.md) |

I2C 总线恢复、GPIO 手动翻转、外设 reset 会改变线路和对端状态，只在确认总线所有权及恢复策略后操作。故障不明时不直接用频繁 DeInit/Init 掩盖状态问题。

软件与硬件链路均核对后仍出现特定异常，应按完整型号和芯片修订查询勘误。例如 F101/F102/F103 中容量使用 [ES096](https://www.st.com/resource/en/errata_sheet/es096-stm32f101x8b-stm32f102x8b-and-stm32f103x8b-mediumdensity-device-limitations-stmicroelectronics.pdf)。逐项核对受影响版本、触发条件和官方规避方法，不把某个 I2C workaround 无条件加入所有 F1 工程。

断点会改变串口/DMA 时序、看门狗和 USB 主机超时；取证方法按问题选择。统计计数、有限事件缓冲、GPIO 示波标记可能比逐指令单步更适合，新增观测也要评估资源和时序影响。

完成须在相同负载和边界条件下复测，并报告真实波形/数据或可重复观测。仅读取寄存器或构建通过不能证明外部通信正常。
