# F1 LL 常用操作

仅在工程已使用 LL，或任务已明确需要该外设由 LL 管理时读取；本页不要求切换驱动。以下实例名/引脚/通道用于说明 API，按实际配置替换。先确认时钟、GPIO/AFIO、外设初始化和中断所有者，头文件为 `stm32f1xx_ll_*.h`。

## GPIO 读写

已配置为输出的引脚，按需求选择一次写高、写低或翻转，不顺序执行全部操作：

```c
LL_GPIO_SetOutputPin(GPIOA, LL_GPIO_PIN_5);
```

```c
LL_GPIO_ResetOutputPin(GPIOA, LL_GPIO_PIN_5);
```

```c
LL_GPIO_TogglePin(GPIOA, LL_GPIO_PIN_5);
```

读取引脚实际输入电平用 `LL_GPIO_IsInputPinSet(GPIOA, LL_GPIO_PIN_5)`，读取输出锁存值用 `LL_GPIO_IsOutputPinSet()`；开漏释放后的实际高低还取决于上拉/外部负载。Set/Reset 利用位操作寄存器；Toggle 有读改写竞争，不让多个上下文无同步翻转同一端口。

## USART 轮询发送

仅适用于实际已配置为 8 位有效数据的发送路径，且没有 HAL、DMA 或 ISR 同时占用发送。deadline 和 byte 由应用提供；`HAL_GetTick()` 需要已运行的 HAL tick，纯 LL 工程改用自己的有界时基。函数调用形态为：

```c
static int uart_write_byte(USART_TypeDef *uart, uint8_t byte, uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    while (!LL_USART_IsActiveFlag_TXE(uart)) {
        if ((uint32_t)(HAL_GetTick() - start) >= timeout_ms) return 0;
    }
    LL_USART_TransmitData8(uart, byte);
    return 1;
}
```

TXE 表示 DR 可接受下一字节，不代表停止位已发完。需要切 RS485 方向、停时钟等操作时，还需有界等待 `LL_USART_IsActiveFlag_TC()`。9 位有效数据使用对应 Data9 接口；带校验时核对实际有效位数，不机械用 Data8。

## USART 接收中断

需要 RXNE 中断时，在接收缓冲与所有者准备好后启用 `LL_USART_EnableIT_RXNE(USART1)` 及实际 NVIC。IRQ 中先取得一次 SR 状态，按需求读取 DR；RXNE 表示有字节，PE/FE/NE/ORE 表示需要记录/丢弃或恢复。F1 清除这些接收错误和 IDLE 的序列是读 SR 再读 DR，不是给 SR 任意位写 0。

`LL_USART_ReceiveData8()` 会读 DR 并消费数据；`LL_USART_ClearFlag_ORE()` / `LL_USART_ClearFlag_IDLE()` 也包含 SR/DR 读序列，不能在已取出字节后再无条件清一次，或在 RX DMA 工作时偷读 DR。用一次状态快照协调数据与错误处理；只开业务需要且处理函数已实现的中断，TXE 中断在无待发数据时要关闭，否则可能持续进 IRQ。

## DMA 通道

F1 使用固定请求映射，没有新系列的 DMAMUX。先按实际 MCU 表确认 USART1_RX 等请求对应 channel，并检查其他外设占用。以下仅说明已经选定 DMA1 Channel5 的地址/长度设置；调用前该通道必须禁用，方向、递增、宽度、模式由初始化完成，rx_buffer 存储期覆盖传输：

```c
LL_DMA_SetPeriphAddress(DMA1, LL_DMA_CHANNEL_5,
                       LL_USART_DMA_GetRegAddr(USART1));
LL_DMA_SetMemoryAddress(DMA1, LL_DMA_CHANNEL_5, (uint32_t)rx_buffer);
LL_DMA_SetDataLength(DMA1, LL_DMA_CHANNEL_5, count);
```

count 是数据项数量，须在该 DMA 的计数范围内。按需求清理该 channel 旧标志，设置 HT/TC/TE 中断及 NVIC，再 `LL_DMA_EnableChannel(DMA1, LL_DMA_CHANNEL_5)` 与 `LL_USART_EnableDMAReq_RX(USART1)`。顺序须保证外设产生请求前缓冲已就绪；这不意味着所有 DMA 都必须开 HT/TC 中断。

IRQ 检查对应 `LL_DMA_IsActiveFlag_HT5/TC5/TE5`，清对应 `LL_DMA_ClearFlag_*5`；循环模式用 `count - LL_DMA_GetDataLength(...)` 求当前写入位置，处理回绕和消费者落后。需要重新装载地址/计数时，先停止请求并禁用通道，确认不再传输后再改；HAL 正管理的 DMA 不能按此另建状态机。

LL 常见寄存器函数为头文件内联实现；使用 `LL_*_Init()` 等完整驱动函数时，按实际声明检查 `USE_FULL_LL_DRIVER` 和相应 LL 源是否加入 CMake。完成条件是初始化/IRQ/数据所有权一致、编译链接通过，功能和时序另做板端验证。依据：[官方 F1 LL 头文件](https://github.com/STMicroelectronics/stm32f1xx-hal-driver/tree/master/Inc)。
