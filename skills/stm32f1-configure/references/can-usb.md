# CAN 与 USB Device

按实际器件、板卡及功能选读。以下参数是调用形式，不能据此给空工程添加 CAN/USB。先完成[通用查询与保存流程](operations.md)，时钟见[时钟配置](clocks.md)。

## CAN：查询与位时序

先查 `get mode available CAN`、`get mode active CAN` 和 `get ip parameters CAN Prescaler`，同时查 BS1、BS2、SJW、Mode。对支持 CAN 且需要普通收发的工程，模式形式是 `set mode CAN CAN_Activate`，实际模式名称先由当前工具确认。

只有实际 CAN 时钟为 36 MHz、需求为 500 kbit/s 且采样点 75% 合适时，参数可采用：

```text
set ip parameters CAN Prescaler 6
set ip parameters CAN BS1 CAN_BS1_8TQ
set ip parameters CAN BS2 CAN_BS2_3TQ
set ip parameters CAN SJW CAN_SJW_1TQ
set ip parameters CAN Mode CAN_MODE_NORMAL
```

位速率 = CAN 时钟 / Prescaler / (1 + BS1 + BS2)，采样点 = (1 + BS1)/(1 + BS1 + BS2)。CAN 取 APB1 时钟，不能套用定时器的倍频规则。`CalculateBaudRate` 可能未即时刷新或显示 499999 一类舍入值；保存重载后按生成的位时序计算，不能只看该字段。

按原理图选择默认 PA11/PA12 或器件支持的重映射引脚；先查询完整引脚信号名，再设置 CAN_RX/CAN_TX。C8T6 的 CAN 与 USB 共用专用 SRAM，不能同时工作；改引脚不能消除此限制。其他 F1 按其参考手册判断。

## CAN：启动与收发

`MX_CAN_Init()` 只初始化控制器。应用需要按协议配置 `HAL_CAN_ConfigFilter()`，检查 `HAL_CAN_Start()`，中断接收再调用 `HAL_CAN_ActivateNotification()`。过滤器 ID/掩码、标准/扩展帧、FIFO 由实际协议决定，不默认全收。

- 发送：检查邮箱余量，使用 `HAL_CAN_AddTxMessage()`，处理完成、仲裁和错误；返回成功表示提交，不等于对端已收到。
- 接收：轮询 FIFO 或在 `HAL_CAN_RxFifo0MsgPendingCallback()` 等回调中调用 `HAL_CAN_GetRxMessage()`，快速交给应用处理。
- 中断：按实际用途启用 `USB_HP_CAN1_TX_IRQn`、`USB_LP_CAN1_RX0_IRQn`、`CAN1_RX1_IRQn`、`CAN1_SCE_IRQn` 中相应向量，并确认进入 `HAL_CAN_IRQHandler()`。

外部总线需要收发器、正确电压、共地和端接。内部回环只验证控制器路径；不能替代收发器、波形和对端 ACK 验证。总线关闭时先定位波特率、接线、端接和 ACK，再选择恢复策略。

## USB Device：生成条件

确认该 F1 具备 USB Device、实际 D+/D− 引脚和电气连接。F103 USB 时钟要求 48 MHz；只有 PLL 输出 72 MHz 时才考虑 1.5 分频。HSI 配置不能凭“计算为 48 MHz”就认定满足 USB 精度。

仅在需求为 USB CDC 时，当前经典 CubeMX 的形式为：

```text
set mode USB Device
set mode USB_DEVICE CDC_FS
```

查询模式失败时按当前 XML/已保存 IOC 核对，不能把 CDC 名称猜成任意类。其他类选择对应模式，不执行 CDC 配置。检查 USB IRQ、时钟、描述符、VID/PID 来源及实际枚举需求。F103 板级 D+ 上拉等要求查 [AN4879](https://www.st.com/resource/en/application_note/an4879-usb-hardware-and-pcb-guidelines-using-stm32-mcus-stmicroelectronics.pdf)，不能套用其他系列的内部上拉假设。

## USB CDC：应用与验证

搜索生成文件名，不能假设固定存在 `USB_DEVICE/App`；CMake 工程也可能使用 Src/Inc。确认 `USBD_Init`、类注册、接口注册、`USBD_Start` 链路和 `MX_USB_DEVICE_Init()` 调用。

`CDC_Transmit_FS(buffer, length)` 是异步提交：只有设备配置完成、类数据有效时才能发送；`USBD_BUSY` 表示忙，不会自动排队。缓冲区保持有效且不被改写直到发送完成。主机未打开端口时的策略由应用决定，不在 ISR 忙等。

接收回调按当前生成接口保存/转交数据，并用实际生成的接收接口重新装载接收缓冲；处理缓冲区所有权，避免重装后覆盖尚未消费的数据。根据包长与吞吐评估缓冲区，不能把生成默认缓冲全部保留而忽略 C8 的 20 KB SRAM。

完成分为：代码生成和构建、主机枚举、类请求/端点传输、拔插和忙态恢复。没有主机及板卡证据时只能报告前两者中实际完成的阶段。

依据：[RM0008](https://www.st.com/resource/en/reference_manual/cd00171190.pdf)、项目中的 CAN HAL、USB Device 中间件与生成接口。
