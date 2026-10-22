# I2C 与 SPI

按实际总线、从设备协议、接线和速度选择章节。I2C1/SPI2 和数值只表达语法，不是固定外设分配。先加载真实 IOC；未知参数按[操作基础](operations.md)查询。

## I2C 查询与配置

```text
get mode available I2C1
get mode active I2C1
get mode_param_list I2C1 I2C
get ip parameters I2C1 ClockSpeed
get functions PB6
get functions PB7
```

需要新建 I2C1 时使用 `set mode I2C1 I2C`，再根据接线设定 SCL/SDA，检查默认或 remap。仅当目标速率是 100 kHz 时：

```text
set ip parameters I2C1 ClockSpeed 100000
get ip parameters I2C1 ClockSpeed
```

F1 用 ClockSpeed/DutyCycle 等字段，不照搬具有 TIMINGR 的系列。地址模式、OwnAddress、双地址、general call、时钟延展等按角色和协议查询后修改，不能把主机地址和从机地址混为一项。

生成后检查 I2C 时钟、AFIO、开漏引脚及外部上拉。内部上拉不能无条件替代符合总线电容/速度要求的外部上拉。SDA/SCL 被拉低时先确认电路、供电和设备状态，不能只改超时。

## I2C API

适用于已初始化的传统 HAL。`hi2c1` 来自实际生成代码；以下是调用形式，不是顺序操作：

| 需求 | 调用形式 |
| --- | --- |
| 主机发送 | `HAL_I2C_Master_Transmit(&hi2c1, addr7 << 1, buf, n, timeout_ms)` |
| 主机接收 | `HAL_I2C_Master_Receive(&hi2c1, addr7 << 1, buf, n, timeout_ms)` |
| 寄存器读 | `HAL_I2C_Mem_Read(&hi2c1, addr7 << 1, reg, I2C_MEMADD_SIZE_8BIT, buf, n, timeout_ms)` |

这里只描述 7 位地址：addr7 是未移位的实际设备地址，调用前左移一次；若上层已经存储移位地址就不再移位。寄存器地址宽度按设备协议选择，不一律 8 位。Size 是数据字节数，timeout_ms 按 HAL tick。10 位地址另查实际 API。

只有需要异步时使用相应 `_IT`/`_DMA`；按目标查询并启用 I2C EV/ER IRQ 和所需 DMA channel，检查 handle 关联及回调。不要把 I2C 当成 UART，只启用一个 IRQ 就假定错误链完整。处理 HAL_BUSY、NACK/AF、BERR、ARLO 等实际状态；重试次数和恢复动作由总线协议决定。

## SPI 查询与配置

```text
get mode available SPI2
get mode active SPI2
get mode_param_list SPI2 Full_Duplex_Master
get ip parameters SPI2 CLKPolarity
get mode_param_possvalue SPI2 Full_Duplex_Master BaudRatePrescaler
```

只有需要 SPI2 全双工主机时 `set mode SPI2 Full_Duplex_Master`。按接线分配实际 SCK/MISO/MOSI，确认自动引脚结果。以下字段独立选择：

| 参数 | 依据与候选示意 |
| --- | --- |
| CLKPolarity | 空闲低/高，对应 SPI_POLARITY_LOW/HIGH |
| CLKPhase | 第一/第二边沿，对应 SPI_PHASE_1EDGE/2EDGE |
| DataSize | 8/16 位，影响 Size 与内存对齐 |
| BaudRatePrescaler | SPI_BAUDRATEPRESCALER_2…256；按实际 PCLK 和从设备上限计算 |
| FirstBit | MSB/LSB，与对端协议一致 |
| NSS | 软件/硬件、主从方向及片选时序，按当前模式查询 |

若需求确为低空闲、第一边沿、16 分频，命令形式为：

```text
set ip parameters SPI2 CLKPolarity SPI_POLARITY_LOW
set ip parameters SPI2 CLKPhase SPI_PHASE_1EDGE
set ip parameters SPI2 BaudRatePrescaler SPI_BAUDRATEPRESCALER_16
```

这三项不是通用默认组合。F103 的 SPI1 与 SPI2 总线时钟不同，不能都用 SYSCLK 算速率。回读、保存重载、生成，核对 hspi、GPIO 和片选默认电平。

## SPI API 与片选

按需求使用 `HAL_SPI_Transmit`、`HAL_SPI_Receive` 或 `HAL_SPI_TransmitReceive`，参数为实际 hspi、持久缓冲区、元素数量和超时。16 位 DataSize 时 Size 是 16 位元素数，即使形参为 uint8_t 指针也不自动变成字节数。

软件 NSS 不会自动帮应用切换任意 CS GPIO。应用按设备时序拉有效、传输、确认结束、释放；IT/DMA 不能在 Start 返回时就释放 CS 或复用缓冲区。主机接收需要时钟/发送占位数据，半双工和只接收模式按 HAL 实现处理。

DMA 前核对固定通道冲突，特别是 C8 SPI2_RX 与 USART1_TX。完成需真实波形/协议响应；构建和 HAL_OK 不能证明从设备已接受数据。

接口依据：[HAL I2C](https://github.com/STMicroelectronics/stm32f1xx-hal-driver/blob/master/Src/stm32f1xx_hal_i2c.c)、[HAL SPI](https://github.com/STMicroelectronics/stm32f1xx-hal-driver/blob/master/Src/stm32f1xx_hal_spi.c)。
