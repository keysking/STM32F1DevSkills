# 时钟查询与配置

先确定完整型号、板上晶振/外部时钟及实际需求。以下数值仅说明已知 F103 模型的语法，不是整个 F1 的固定方案；尤其不由“C8T6”推断板上有 8 MHz 晶振。只查询时不执行 clock set 的写入形式。

## 查询现状和节点

```text
get ip parameters RCC HSE_VALUE
get ip parameters RCC SYSCLKFreq_VALUE
get ip parameters RCC HCLKFreq_Value
get ip parameters RCC APB1Freq_Value
get ip parameters RCC APB2Freq_Value
get ip parameters RCC ADCFreqValue
isPluginError "Clock Configuration"
clock set
```

不带节点/值的 `clock set` 用于查看当前入口/Usage；按实际输出和 `db/plugins/clock/<ClockTree>.xml` 查询节点及输入选择。节点名、选择器索引、倍率和分频分别解释，不能把索引当 MHz。

## 使用内部 HSI/PLL

只有实际需求采用 HSI 路径时设置。F103 的 PLL 输入可用 HSI/2；例如需要名义 48 MHz 且器件允许该方案时，当前 F103 模型的写法是：

```text
clock set PLLSource 0
clock set PLLMUL 12
clock set APB1CLKDivider 2
clock set SysClkSource 2
```

这里 0 为 HSI/2、2 为 SYSCLK 选择 PLL，须先确认目标模型。名义 SYSCLK=48 MHz、PCLK1=24 MHz；RC 精度和通信要求另行判断，不能只看频率值就声称 USB 时钟合格。

## 使用 HSE

晶体/陶瓷谐振器与外部有源时钟（bypass）是不同模式，先 `get mode available RCC` 查当前名称。只在实际为 8 MHz 晶体、需要 F103 的 72 MHz 配置时参考：

```text
set mode RCC HSE-External-Oscillator
set ip parameters RCC HSE_VALUE 8000000
clock set PLLSource 1
clock set HSEDivPLL 1
clock set PLLMUL 9
clock set APB1CLKDivider 2
clock set SysClkSource 2
```

不是晶体输入时不能套用此模式。F105/F107 还有预分频与其他 PLL，按其实际树查询，不复制该索引表。

## 检查各派生时钟

- F103 72 MHz 情况下 APB1 最大 36 MHz；APB1 分频不为 1 时相应定时器时钟通常为 PCLK1 的两倍。PWM 公式使用实际 TIM 时钟，不直接使用 PCLK1。
- C8 ADC 时钟上限 14 MHz。PCLK2=72 MHz 且需要 ADC 时，`clock set ADCPresc 6` 得 12 MHz；/4 得 18 MHz，即使设置 OK 也不合法。
- 只有启用 USB 的需求才设置 USB 时钟。PLL=72 MHz 时，当前模型 `clock set USBPrescaler 1.5` 得 48 MHz；还需合格时钟源和硬件。
- HSE_VALUE/LSI_VALUE 等描述模型或软件计算值，不是仪器测得频率；修改常量不等于修改真实晶振。

修改后回读需求频率、`isPluginError "Clock Configuration"`，保存重载。生成后检查 OscConfig、ClockConfig、Flash latency、APB 分频、ADC/USB 分频以及 SystemCoreClock。模型合法仍不能证明板上晶振起振。

只需调整波特率/采样率时不要无依据重配整个系统时钟。STOP 唤醒后的恢复见[低功耗](watchdogs-power.md)，不要把开机配置直接视为通用恢复函数。
