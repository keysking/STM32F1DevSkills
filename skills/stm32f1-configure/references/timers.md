# 定时器、PWM、捕获与编码器

各节是独立操作参考。TIM2、PA0 和参数值仅说明语法，依实际需求替换。C8 的 TIM1–TIM4 为 16 位；其他型号查数据手册和设备头文件。先加载 IOC，未列功能按[操作基础](operations.md)查询。

## 查询模式和参数

```text
get mode available TIM2
get mode active TIM2
get mode_param_list TIM2 "PWM Generation1 CH1"
get ip parameters TIM2 Prescaler
get ip parameters TIM2 Period
get functions PA0-WKUP
```

未启用 PWM 时不为查询强制启用。`get mode_param_list TIM2 ".*"` 可能在多通道上下文中抛异常；改查明确模式。返回集合可能包含不适用/其他模式字段，如 TIM5_TI4_REMAP，不能据此认定 C8 有 TIM5。

## 内部时基

仅需要内部时钟驱动 TIM2 时 `set mode TIM2 Internal`。按实际 TIM 时钟、计数方向和周期计算 PSC/ARR；向上边沿对齐更新频率为 `fTIM / ((PSC+1) × (ARR+1))`。APB 分频不为 1 时 TIM 时钟可能是 PCLK 的两倍。

例如实际 fTIM=8 MHz、需要 1 kHz 向上计数时可设 `Prescaler 7`、`Period 999`；通过 `set ip parameters TIM2 <字段> <值>` 写入。中心对齐和高级定时器重复计数器按其事件规律另算，不机械套公式。

若需周期中断，启用实际 TIM IRQ、检查 handler，并在应用使用 `HAL_TIM_Base_Start_IT(&htim2)`。普通计数使用 `HAL_TIM_Base_Start`；生成 Base_Init 不等于开始计数。PeriodElapsed 回调判断 Instance，不能覆盖用于 HAL tick 的其他定时器分支。

## PWM 输出

只有需要对应通道输出时，模式名完整加引号：

```text
set mode TIM2 "PWM Generation1 CH1"
get mode active TIM2
get mode_param_list TIM2 "PWM Generation1 CH1"
```

按实际引脚/AFIO 配置 CH1；“No Output” 模式不能替代需要物理引脚的 CH1 模式。PSC/ARR 是同一 TIM 共享时基，修改会影响其他通道。

需要设置该通道 Pulse 时，当前模式键的形式为 `"Pulse-PWM Generation1 CH1"`，第二通道为 `"Pulse-PWM Generation2 CH2"`。例如已确认 ARR=999、向上 PWM1、高有效且需求为 25% 时：

```text
set ip parameters TIM2 "Pulse-PWM Generation1 CH1" 250
```

实际值由目标频率/占空比决定，0%/100% 与 ARR 边界按当前模式处理。完整后缀 getter 可能返回 null，而裸 `get ip parameters TIM2 Pulse` 显示第一通道；核对 IOC 和生成的逐通道 ConfigChannel，不因 null 盲目重设全部通道。

生成后检查 PWM_Init、ConfigChannel、MspPostInit、GPIO 和 CCR。应用按所需通道调用 `HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1)`；改变占空比可用 `__HAL_TIM_SET_COMPARE`，同时考虑预装载/更新事件。持续硬件 PWM 不必启用周期中断。

`HAL_TIM_PWM_Start_IT` 开的是通道比较中断，不能当作 Base_Start_IT 的同义接口；根据需求选择 PulseFinished 或 PeriodElapsed，判断活动通道。TIM1 的 MOE、互补输出、刹车/死区和锁定是额外条件，按高级定时器能力和负载要求配置。

## 输入捕获、PWM 输入或编码器

先用 `get mode available TIM2` 选择实际需求。C8 模型可见 `Input_Capture1_from_TI1`、`PWM_Input_1`、`Encoder_Interface` 等，它们占用的通道组合不同。

| 需求 | 关键配置与应用 |
| --- | --- |
| 单通道捕获 | 查 IC 极性、选择、分频、滤波与 GPIO；用 HAL_TIM_IC_Start/Start_IT/Start_DMA，读取 HAL_TIM_ReadCapturedValue。 |
| PWM 输入测频/占空比 | 核对直接/间接输入通道、reset/slave trigger 和计数时钟；不能只开一个普通捕获通道就假定可得完整周期。 |
| 编码器 | 查 EncoderMode、IC1/IC2、滤波和 A/B 引脚；用 HAL_TIM_Encoder_Start，检查方向及 16 位回绕。 |

模式修改会占用/替代既有通道；先查已有所有者。测量计算处理溢出、无信号和输入滤波影响。按[DMA/NVIC](dma-irq.md)补齐需要的链路，未要求的模式不启用。
