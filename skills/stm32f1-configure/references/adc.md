# ADC 配置与采样

按实际模拟信号、采样频率和精度需求选择章节。ADC1/IN4 仅为 C8 语法对象；先核对引脚、电压范围、源阻抗和参考电压。F1 ADC 固定 12 位，不创建其他系列的 Resolution 配置。

## 查询与通道配置

```text
get mode available ADC1
get mode active ADC1
get mode_param_list ADC1 IN4
get ip parameters ADC1 NbrOfConversion
get ip parameters RCC ADCFreqValue
```

新需求需要 ADC1_IN4 时 `set mode ADC1 IN4`；核对生成器分配到实际模拟引脚，未使用通道不顺便开启。参数枚举不足时读取目标对应 ADC Modes.xml 和 IOC。

当前 regular 序列的字段可能带序号与分组，例如：

| 字段形式 | 用途 |
| --- | --- |
| `Rank-0#ChannelRegularConversion` | 序列条目的 rank |
| `Channel-0#ChannelRegularConversion` | 该条目的 ADC_CHANNEL_* |
| `SamplingTime-0#ChannelRegularConversion` | ADC_SAMPLETIME_* |
| ScanConvMode / NbrOfConversion | 是否扫描、转换数量 |
| ContinuousConvMode / ExternalTrigConv | 连续转换或触发来源 |

序号 0 不是物理通道 0；按当前序列取得真实键。只有需求需要改变第一项采样时间、并且 55.5 周期满足信号条件时，写法为：

```text
set ip parameters ADC1 "SamplingTime-0#ChannelRegularConversion" ADC_SAMPLETIME_55CYCLES_5
get ip parameters ADC1 "SamplingTime-0#ChannelRegularConversion"
```

采样时间由源阻抗、ADC 时钟及精度要求决定，不默认最短。C8 ADC 时钟不超过 14 MHz；需结合分频、采样周期和转换时间计算吞吐。内部温度/Vrefint 的使能、建立时间和校准公式按器件资料，不能照搬带校准常数的新系列。

## 单次轮询

已生成初始化且 ADC 空闲后，按 HAL 顺序进行校准，再按需求采样。以下表示调用顺序，应用需检查每步返回值并提供真实句柄/超时：

```c
HAL_StatusTypeDef status = HAL_ADCEx_Calibration_Start(&hadc1);
/* 仅当校准成功且需要采样时，继续 Start → Poll → GetValue → Stop。 */
```

`HAL_ADC_Start`、`HAL_ADC_PollForConversion`、`HAL_ADC_GetValue`、`HAL_ADC_Stop` 分别负责启动、等待、读取和停止。不要在正在转换时反复校准；校准函数在 F1 中实际执行硬件操作。单次轮询无需添加 DMA。

## 中断、扫描与 DMA

单通道 regular 转换需要 IT 时，启用 ADC 对应 IRQ 并检查 `HAL_ADC_IRQHandler()`；应用使用 `HAL_ADC_Start_IT(&hadc1)`，在 `HAL_ADC_ConvCpltCallback()` 判断句柄后读取 `HAL_ADC_GetValue()`。

**F1 regular 多 rank 扫描不能用一次 EOC 中断逐项接收。** 扫描序列只在最后一次转换产生转换完成中断，DR 中之前的数据已被后续 rank 覆盖；加快回调或循环 GetValue 无法取回。连续完整多通道采集选择 DMA。轮询也不能按其他系列的每 rank EOC 模型使用。

连续或块采集需要 DMA 时先查 `dma available_periph_requests ADC1` 和 `dma list_all`，确认 ADC1 请求与 DMA1_Channel1 等实际映射；C8 ADC2 没有同等独立 DMA 路径，双 ADC multimode 是另一种配置。

按实际模式设置 `dma add ADC1`、普通/循环、外设/内存数据宽度及递增，启用映射 channel IRQ。生成应关联 `hadc1.DMA_Handle`，应用在校准完成后使用 `HAL_ADC_Start_DMA`。Length 为转换数据项数量，不是 uint32_t 指针就一定按 32 位传输；按 DMA 宽度、缓冲区类型和 API 转换要求保持一致。

多通道 DMA 的缓冲区顺序依 rank；半满/全满按完整扫描组处理，循环模式防止覆盖。定时器触发时配置 ADC 的确切外部触发候选与 TIM 事件，再分别启动 ADC 和触发源，不能只初始化 TIM。参数候选中出现不在该芯片上的 TIM 不代表可用。

修改后保存重载、生成、构建；板端用已知输入检查原始值、参考电压、通道顺序和采样速率。构建通过不是采样准确度证据。

## 多通道数据路径选择

需要 regular 扫描 DMA 时，启用实际通道后查询 `ScanConvMode`、`NbrOfConversion`、`ContinuousConvMode`、`DiscontinuousConvMode` 及每项 rank/channel/sampling 字段；按数量设扫描、关闭不连续模式，并由需求选择单次或连续/外部触发。缓冲区若使用 `uint16_t samples[count]`，DMA 外设和内存均设 HALFWORD、内存递增；count 必须覆盖完整扫描组且存储期覆盖 DMA 工作时间。初始化和校准成功后，调用形式为：

```c
HAL_StatusTypeDef status = HAL_ADC_Start_DMA(&hadc1, (uint32_t *)samples, count);
```

`samples`、`count` 来自实际缓冲区；类型转换不改变传输宽度。普通模式完成后按业务决定是否再次启动；循环模式按半满/全满边界消费，别在 DMA 正覆盖的区间读取整块数据。

若需求允许逐次软件触发、且不能用 DMA，可在 regular 扫描中启用不连续模式、`NbrOfDiscConversion=1`，并关闭 ContinuousConvMode。校准后每次 `HAL_ADC_Start()` → `HAL_ADC_PollForConversion()` → `HAL_ADC_GetValue()` 只处理一个 rank，等待本组所有 rank 完成再 `HAL_ADC_Stop()`；不要每个 rank 都 Stop 以免复位序列进度。核对实际 HAL 的不连续轮询实现和每次触发来源，不能套用于连续扫描。

Injected 是另一条路径，支持数量由器件决定：用 `HAL_ADCEx_InjectedStart[_IT]`，按实际 injected rank 调用 `HAL_ADCEx_InjectedGetValue(&hadc1, ADC_INJECTED_RANK_1)` 等读取各自 JDR。先配置 injected 通道/触发与 IRQ，不把 regular DMA 缓冲区规则套到 injected。

依据为工程 `stm32f1xx_hal_adc.h` 中 ScanConvMode 的限制及 `stm32f1xx_hal_adc.c` 的轮询实现；[官方 HAL ADC 头文件](https://github.com/STMicroelectronics/stm32f1xx-hal-driver/blob/master/Inc/stm32f1xx_hal_adc.h)用于对照，工程版本优先。
