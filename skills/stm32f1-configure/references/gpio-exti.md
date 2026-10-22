# GPIO、EXTI 与调试引脚

各节按实际需求选择；PA5/PB0 用于说明语法，不表示 LED/按键或固定极性。输出不需要自动启用 EXTI 或切换 LL。会话先加载实际 IOC；启动及扩展查询见[操作基础](operations.md)。

## 查询

```text
get pinout
get functions PA5
get signal PA5
get gpio parameters PA5 .*
get mode active SYS
```

参数集合随当前信号变化。查不到 GPIO 参数时先看该引脚是否仍为外设信号，不为查询直接抢占它。

## 输出

需要把实际选定的 PA5 设为输出时：

```text
set pin PA5 GPIO_Output
get gpio parameters PA5 .*
```

按负载选择本次需要修改的字段；以下命令彼此独立，不要求全部执行：

| 条件 | 会话写法（PA5） |
| --- | --- |
| 需要推挽 | `set gpio parameters PA5 GPIO_ModeDefaultOutputPP GPIO_MODE_OUTPUT_PP` |
| 需要开漏 | `set gpio parameters PA5 GPIO_ModeDefaultOutputPP GPIO_MODE_OUTPUT_OD` |
| 初始化应高/释放 | `set gpio parameters PA5 PinState GPIO_PIN_SET` |
| 初始化应低 | `set gpio parameters PA5 PinState GPIO_PIN_RESET` |

上下拉和速率先从当前 `get gpio parameters PA5 .*` 获取字段，再按 GPIO 数据库确认枚举。当前字段的写法包括：

| 需求已确认时 | 会话写法（PA5） |
| --- | --- |
| 不启用内部上下拉 | `set gpio parameters PA5 GPIO_PuPd GPIO_NOPULL` |
| 选择低速输出 | `set gpio parameters PA5 GPIO_Speed GPIO_SPEED_FREQ_LOW` |

速率另有当前模型支持的 MEDIUM/HIGH 候选，按负载、边沿和电气规范选择；并非业务翻转频率。开漏依赖上拉，不默认选最高速率。修改后回读、保存重载并生成，检查 GPIOA 时钟、模式、速率和初始电平设置顺序。

HAL GPIO 已初始化后，固件按需要调用其中一项：

| 操作 | C 调用 |
| --- | --- |
| 高电平/开漏释放 | `HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);` |
| 低电平 | `HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);` |
| 读输入电平 | `GPIO_PinState level = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_5);` |
| 翻转输出 | `HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);` |

ReadPin 读输入数据，不等同于输出锁存。Toggle 不隐含闪灯、周期或 HAL_Delay。PC13–PC15 依数据手册核对特殊电气限制。

## 普通输入

需求仅为读取电平时，选实际输入脚，例如 `set pin PB0 GPIO_Input`，再 `get gpio parameters PB0 .*` 查询输入模式和上下拉。只有实际需要内部上拉时使用 `set gpio parameters PB0 GPIO_PuPd GPIO_PULLUP`；下拉/无上下拉分别查询选择 GPIO_PULLDOWN/GPIO_NOPULL。不要用输出模式键替代输入模式键。生成后以 `HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0)` 读取，不需 NVIC。

## 边沿中断

仅在需要中断通知时使用。假定真实输入为 PB0 且需要下降沿：

```text
set pin PB0 GPIO_EXTI0
get gpio parameters PB0 .*
set gpio parameters PB0 GPIO_ModeDefaultEXTI GPIO_MODE_IT_FALLING
NVIC enable EXTI0_IRQn
NVIC get_enable_state EXTI0_IRQn
NVIC get_priority EXTI0_IRQn
```

上升沿或双边沿分别查询/选择 `GPIO_MODE_IT_RISING`、`GPIO_MODE_IT_RISING_FALLING`；只有硬件事件需求时使用相应事件模式，不自动打开 CPU 中断。上下拉依接线单独确定。

同号 EXTI 只能选择一个 GPIO 端口来源，例如 PB0 会替代 PA0 的 EXTI0。EXTI5–9、10–15 共享 IRQ 组，按目标查询真实 IRQ 名。优先级按裸机/RTOS 规则设置，不默认所有中断为 0。

生成后检查 AFIO 来源、触发边沿、NVIC 和 `EXTI0_IRQHandler → HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0)`。传统 HAL 用户回调：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0) {
        /* 按实际需求记录事件，或使用允许在 ISR 调用的通知接口。 */
    }
}
```

回调参数是引脚掩码；端口通过配置来源确定。机械输入若需消抖，在应用中处理，不在 ISR 内阻塞延时。

## SWD、JTAG 与重映射

改 PA13/PA14、PA15/PB3/PB4 或 AFIO 重映射前先查 `get mode active SYS` 和所有受影响引脚。将 PA13 改成 GPIO 可能令 PA14 被释放、SYS 变成 No_Debug。

只有任务需要保留 SWD 并释放 JTAG 引脚时，使用 `set mode SYS Serial_Wire`，生成结果应保留 SWD 并出现相应 SWJ_NOJTAG 配置；这与禁用全部调试不同。AFIO 重映射按外设组作用，不把一个引脚的修改描述成完全局部。

LL 需求另读[HAL/LL](../../stm32f1-firmware/references/hal-ll.md)；普通 GPIO 修改沿用原驱动层。
