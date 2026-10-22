# DMA 与 NVIC

仅在外设需求需要中断/DMA 时操作。本页提供共同判断，串口等完整路径见对应外设参考。F1 DMA 使用固定请求映射的 channel，不能套用 DMAMUX、GPDMA 或其他系列的 stream 选择方法。

## 发现请求与分配

```text
dma
dma available_periph_requests USART1
dma list_all
dma get_param USART1_RX Mode
NVIC get_ip_interrupts USART1
NVIC get_enable_state USART1_IRQn
NVIC get_priority USART1_IRQn
```

用真实实例/请求/IRQ 替换。`dma` 裸命令用于读 Usage，某些子查询可能不支持；结合器件 DMA 映射和当前分配表。USART1_TX 与 SPI2_RX 在 C8 上共用 DMA1_Channel4，不能通过任选空闲 channel 消除冲突；需调整运行方式、时序或外设选择。

`available_periph_requests` 可能只列出尚未分配的候选。已有 USART1_RX 时只显示 TX 不表示不支持 RX，需结合 `dma list_all` 判断。

## 添加与设置

只有新需求确实需要且尚未分配时 `dma add USART1_RX`。模式、方向、宽度、递增和优先级按返回的参数/候选及外设数据设置，写法为 `dma set_param <请求> <参数> <内部值>`，然后用 get_param/list_all 回读。

通道 IRQ 与外设 IRQ 职责不同。例如 UART IDLE/最终发送完成在 USART IRQ，ADC/TIM 还需对应触发或外设状态。不得因用了 DMA 就一律关闭外设 IRQ。

## NVIC 优先级

仅当本任务需要开启或修改优先级时执行对应命令。例如已确认 USART1 中断需设置为库级优先级 5：

```text
NVIC enable USART1_IRQn
NVIC set_priority USART1_IRQn 5
NVIC get_priority USART1_IRQn
```

5 不是通用推荐值。数值越小越紧急；区分抢占/子优先级、分组和寄存器左移后的值。调用 FreeRTOS FromISR 或 CMSIS ISR 可用接口的中断，必须满足实际 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 阈值及分组；不要把移位后的 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 直接传给 HAL 的逻辑优先级参数。

## 生成和运行检查

1. 对应 DMA 控制器时钟先启用；源/目标地址所在区域可访问。
2. MSP 配置实际 channel，`__HAL_LINKDMA` 方向和 handle 正确。
3. 向量表中的 IRQ 名与 handler 一致；handler 调用相应 HAL_DMA_IRQHandler/外设 handler，没有重复定义。
4. 应用确实发起传输，长度单位、缓冲区对齐和生命周期正确，返回值已处理。
5. 检查 CNDTR、请求使能、通道 EN、完成/错误标志和回调；配置存在不是传输开始的证据。

共享 IRQ 处理所有属于该应用的来源。只释放已确认不再使用的请求/中断，不为了排错重建全部 DMA。LL 模式需要应用自己处理 flags 和回调桥接，HAL handler 不会自动替 LL 实现状态机。
