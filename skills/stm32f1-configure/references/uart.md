# UART 查询、配置与使用

USART1、PA9/PA10 和 57600 是语法对象；按接线、协议和当前工程替换。只选择轮询、IT、DMA 中实际需要的路径。启动会话后先加载真实 IOC，扩展参数见[操作基础](operations.md)。

## 查询和基础配置

```text
get mode available USART1
get mode active USART1
get functions PA9
get functions PA10
get ip parameters USART1 BaudRate
get mode_param_list USART1 Asynchronous
get mode_param_possvalue USART1 Asynchronous WordLength
```

需要新建异步串口且上述引脚符合需求时，启用 `set mode USART1 Asynchronous`，然后显式设定所需 TX/RX 信号并核对自动分配：

```text
set pin PA9 USART1_TX
set pin PA10 USART1_RX
```

已有串口仅改需求参数。例如目标波特率确为 57600 时 `set ip parameters USART1 BaudRate 57600`。WordLength、Parity、StopBits 等从当前模式枚举选择；CubeMX 内部值可能不同于 HAL 宏。

F1 字长配置包含校验位的影响。协议的有效数据位、校验与 HAL WordLength 联合核对，不能把“8 位加校验”直接解释成所有配置都选 WORDLENGTH_8B。9 位无校验的 HAL 数据元素为 uint16_t，Size 不是字节数。

USART1 改到 PB6/PB7 时会影响 AFIO 重映射并释放旧 TX/RX，先确认占用，再设新引脚，检查新旧引脚和生成的 remap 宏。修改后回读、保存重载、生成；核对 USART 时钟、GPIO、电气模式和 huart 初始化。

## 轮询通信

已有 `MX_USART1_UART_Init()` 成功，且应用允许阻塞时使用。`huart1` 取实际生成声明，buf/n/timeout_ms 由应用提供；timeout_ms 是 HAL tick 基准的毫秒超时：

```c
HAL_StatusTypeDef status = HAL_UART_Transmit(&huart1, buf, n, timeout_ms);
```

接收使用 `HAL_UART_Receive()`，按返回状态区分 OK、TIMEOUT、BUSY、ERROR。普通轮询不需要添加 DMA/NVIC。不要在无法推进 HAL tick 的 ISR 内调用带等待的接口。

## 中断通信

只有需要 IT 时启用实际 USART IRQ：

```text
NVIC enable USART1_IRQn
NVIC get_enable_state USART1_IRQn
NVIC get_priority USART1_IRQn
```

生成应有 `USART1_IRQHandler → HAL_UART_IRQHandler(&huart1)`。应用按需调用 `HAL_UART_Transmit_IT()` 或 `HAL_UART_Receive_IT()`，缓冲区保持有效至完成/错误；实现对应 TxCplt/RxCplt/Error 回调并判断句柄。普通定长 IT 接收完成后按需重新挂接，不能同时让轮询和 IT 竞争同一接收状态。

## 接收 DMA

需要 USART1_RX DMA 时先查通道和冲突；普通/循环模式按真实生命周期二选一：

```text
dma available_periph_requests USART1
dma list_all
dma add USART1_RX
dma get_param USART1_RX Mode
```

仅当需求为循环接收时：

```text
dma set_param USART1_RX Mode DMA_CIRCULAR
NVIC enable DMA1_Channel5_IRQn
NVIC enable USART1_IRQn
```

DMA1_Channel5 是 C8 USART1_RX 的映射，其他实例/器件先查询。普通 DMA 使用对应 DMA_NORMAL 候选；不要为阻塞发送添加接收 DMA。

检查 DMA 时钟在外设 DMA 初始化之前开启，MSP 中方向、数据宽度、递增、模式正确，`__HAL_LINKDMA(huart, hdmarx, ...)` 已关联，DMA IRQ 调用匹配 handle。HAL 发送普通 DMA 的最终收尾还需要 USART TC IRQ，不能只开 DMA IRQ。

应用选择 `HAL_UART_Receive_DMA()` 或当前版本支持的 `HAL_UARTEx_ReceiveToIdle_DMA()`。返回 HAL_OK 表示已启动，不表示收到数据。IDLE 依赖 USART IRQ；HT/TC 依赖 DMA IRQ。

## ReceiveToIdle 与循环缓冲

`HAL_UARTEx_RxEventCallback(huart, Size)` 的 Size 表示缓冲区中数据可用到的位置。用前次位置和 HT/TC/IDLE 事件计算新增区间，处理环回和重复位置；不能每次处理 `[0, Size)`。回调内用 `HAL_UARTEx_GetRxEventType()` 区分事件。

循环 DMA 的 HT、TC、IDLE 均不停止接收，不能每次回调重新 Start。普通模式的结束、错误和中止后重新启动分别处理。只有当前位置的模 N 差值不能区分完整一圈与无新增数据；按事件配置和处理时限设计，防止生产者覆盖消费者。

按当前接收位宽设置 DMA 对齐和 Size 单位。异步缓冲区不能是已退出函数的局部数组。RTOS 回调按优先级规则通知任务处理，避免 ISR 长时间复制/解析。

依据：[HAL UART 实现](https://github.com/STMicroelectronics/stm32f1xx-hal-driver/blob/master/Src/stm32f1xx_hal_uart.c)。完成验证分别检查配置链、构建、真实波特率、接收边界和错误恢复；没有板端数据时不声称收发正常。
