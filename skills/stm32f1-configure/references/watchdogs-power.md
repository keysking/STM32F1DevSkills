# 看门狗与低功耗

这些操作改变运行和复位行为，仅在实际需求要求时采用。配置、初始化、启动的边界必须查生成代码；部分看门狗在 Init 内已经启动。

## IWDG

先查 `get mode active IWDG`、`get ip parameters IWDG Prescaler` 和 `Reload`。需要独立看门狗时，启用形式为 `set mode IWDG IWDG_Activate`。若目标 LSI 标称 40 kHz、需求约 1 秒，参数形式可为：

```text
set ip parameters IWDG Prescaler IWDG_PRESCALER_64
set ip parameters IWDG Reload 624
```

超时近似为 `(Reload + 1) × 分频 / LSI频率`。实际按数据手册 LSI 上下界评估；标称 1 秒不表示精确 1 秒。不能引用其他 F1 型号的 LSI 数值，也不能因示例使用 TIM5 测 LSI 就给没有 TIM5 的 C8 配置 TIM5。

生成的 `HAL_IWDG_Init()` 会启动 IWDG，启动后通常只能复位结束。按健康条件调用 `HAL_IWDG_Refresh()`，不能在无条件高优先级循环喂狗而掩盖业务死锁。选项字节可能配置硬件自动启动，排查时一并读取。

## WWDG

需要窗口约束且目标支持时，模式形式为 `set mode WWDG WWDG_Activate`。先查 Prescaler、Window、Counter、EWIMode，再按窗口要求设置 `WWDG_PRESCALER_8` 等实际候选值。

计数时钟为 PCLK1 / (4096 × 分频)，不能使用 APB 定时器倍频后的时钟。刷新过早或过迟都可能复位；按窗口、计数器初值、任务最坏延迟计算允许时间。`HAL_WWDG_Init()` 会启动，刷新使用 `HAL_WWDG_Refresh()`。

需要提前唤醒中断时，设置 EWIMode、启用实际 WWDG IRQ 并检查 `HAL_WWDG_IRQHandler()` 与 EWI 回调。不要把 EWI 当成无限延期的保活机制。调试暂停时如需冻结看门狗，显式设置对应 DBGMCU 行为，并在正常运行验证时核对它。

## Sleep、STOP、STANDBY

先定义唤醒源、功耗目标、数据保留与恢复时间。模式选择互斥，以下不是顺序执行步骤。

- Sleep：CPU 休眠，外设和时钟行为按配置继续。用 `HAL_PWR_EnterSLEEPMode()`，避免不需要的 tick 持续唤醒。
- STOP：通常保留 SRAM/寄存器，唤醒后系统时钟会回到 HSI；按实际器件恢复所需 HSE/PLL、分频及外设时钟。
- STANDBY：主 SRAM 和大多数寄存器丢失，唤醒走复位启动；只把允许保留的数据放入备份域。不能靠普通 RAM 变量跨 STANDBY 保存状态。

进入前清理/确认唤醒标志和 pending 中断，安排 DMA、外设事务和 GPIO 状态。`HAL_SuspendTick()` 是否合适取决于模式和运行模型；FreeRTOS 应与其 tickless/调度流程协调，不能只暂停 HAL tick 就认为 OS 已进入低功耗。

STOP 恢复不宜机械重跑完整 `SystemClock_Config()`。按当前时钟状态构造恢复流程；HAL RCC 超时依赖 `HAL_GetTick()`，tick 停止时等待故障可能无限卡住，应先建立可用超时基准或采用独立有界等待。恢复后核对时钟寄存器、SystemCoreClock、HAL/OS tick 和外设速率。

### 裸机 Sleep

只有实际唤醒 IRQ 已配置、当前没有待处理工作、且不需要 HAL tick 周期唤醒时，可用：

```c
HAL_SuspendTick();
HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
HAL_ResumeTick();
```

这段不负责应用的“检查工作→休眠”竞态；共享队列/事件状态须和中断配合，避免检查之后事件已处理而继续睡眠。IRQ 内不调用依赖已暂停 tick 的 HAL_Delay/超时等待。需要时间持续累计时另用实际持续运行的时间源。

### 裸机 STOP 与时钟恢复

需要 STOP 时，先按需求配置能在 STOP 中工作的 EXTI/RTC 闹钟等唤醒源，完成外设/DMA收尾并处理已有 pending；不要清掉尚未消费的业务事件。启用 PWR 时钟后，实际进入形式为：

```c
HAL_SuspendTick();
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
```

这只是进入部分；返回后必须接上与实际时钟和 tick 所有者匹配的恢复。下面仅适用于 **F103、8 MHz HSE 晶体、PLL×9、AHB/1、APB1/2、APB2/1，裸机 HAL SysTick 时基** 的恢复函数形态；需要该时钟方案才采用，不用于 HSE bypass、连接型 F1、RTOS 或 HAL 定时器时基：

```c
static HAL_StatusTypeDef restore_f103_hse8_72(void)
{
    /* STOP 返回后运行于 HSI。先按当前频率重建超时 tick。 */
    SystemCoreClockUpdate();
    if (HAL_InitTick(uwTickPrio) != HAL_OK) return HAL_ERROR;
    HAL_ResumeTick();

    RCC_OscInitTypeDef osc = {0};
    osc.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    osc.HSEState = RCC_HSE_ON;
    osc.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    osc.PLL.PLLState = RCC_PLL_ON;
    osc.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    osc.PLL.PLLMUL = RCC_PLL_MUL9;
    HAL_StatusTypeDef status = HAL_RCC_OscConfig(&osc);
    if (status != HAL_OK) return status;

    RCC_ClkInitTypeDef clk = {0};
    clk.ClockType = RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK |
                    RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    clk.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    clk.AHBCLKDivider = RCC_SYSCLK_DIV1;
    clk.APB1CLKDivider = RCC_HCLK_DIV2;
    clk.APB2CLKDivider = RCC_HCLK_DIV1;
    return HAL_RCC_ClockConfig(&clk, FLASH_LATENCY_2);
}
```

调用时须处于允许 SysTick 正常抢占的线程上下文，不能保持 PRIMASK/BASEPRI 屏蔽或在高优先级 ISR 中等待 RCC。返回 HAL_OK 后再恢复依赖原频率的通信/采样；失败按应用策略保持可用 HSI、停止相关业务并报告故障，不能继续声称72 MHz已恢复。此函数不补偿休眠期间的时间，HAL 定时器时基和 RTOS 需要按其实际计时/调度恢复流程另写。

### STANDBY 与启动判别

只有允许复位式唤醒且数据已持久化时进入。按实际源选择 RTC 闹钟或 WKUP 引脚；使用 WKUP 时先核对电平不会立即触发，调用 `HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1)`。清理旧唤醒标志用 `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU)`，之后调用 `HAL_PWR_EnterSTANDBYMode()`；不得把这段接在 STOP 后当作下一步。

复位启动早期、PWR 时钟使能后用 `__HAL_PWR_GET_FLAG(PWR_FLAG_SB)` 判别是否从 STANDBY 返回，记录原因后以 `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB)` 清除。恢复备份数据而非普通RAM；RTC日期恢复仍遵循[备份域](rtc.md)。没有实际唤醒源时不进入该模式。

完成验证包含实际唤醒源、时钟恢复、通信恢复、功耗测量与失败路径；调试连接会影响低功耗行为。依据：[RM0008](https://www.st.com/resource/en/reference_manual/cd00171190.pdf)及具体型号数据手册。
