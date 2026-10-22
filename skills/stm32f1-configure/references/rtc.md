# RTC 与备份域

只有需求涉及计时、闹钟或掉电保持时使用。先查实际晶振、VBAT 连接、备份数据用途及当前初始化；不要为了查询而复位备份域。

## 模式与时钟

当前传统 CubeMX 的 RTC 日历配置需先启用 RTC：

```text
set mode RTC RTC_Enabled
set mode RTC RTC_Calendar
```

只执行需求所需模式。单独 Calendar 可能失败；不需要日历时不要照抄。查询用 `get mode active RTC`，参数按具体名称查询；部分版本的 `get ip parameters RTC .*` 会异常退出该命令，应改查当前模式字段或数据库。

时钟可来自型号支持的 LSE、LSI 或 HSE 分频。仅板上实际有相应晶振/外部时钟时才启用它。RTC 预分频按实际输入频率计算；LSI 误差影响计时和看门狗，不能把标称值当作实测值。

改 RTC 时钟源可能需要备份域复位，并丢失 RTC/备份寄存器数据。先确认保留要求，按实际恢复方案执行，不把复位当成常规初始化步骤。

## 初始化与掉电保持

检查生成的 `MX_RTC_Init()` 是否每次执行 `HAL_RTC_SetTime()`/`HAL_RTC_SetDate()`。无条件设置会覆盖已有时间；需要保持时，在可保留的用户代码区根据备份标记、时钟源有效性和应用策略判断是否首次设置。标记不是时间有效性的唯一证据，电池耗尽和备份复位后要重新建立有效状态。

F1 RTC 的硬件核心是计数器；HAL 日历中的日期维护含软件状态。不能承诺调用 SetDate 后完整日期就会由 VBAT 永久保存。检查项目 HAL 的 `DateToUpdate`、备份寄存器使用和跨日更新路径，按需求保存日期/纪元及恢复依据。

## API 与闹钟

按当前 HAL 头文件构造 `RTC_TimeTypeDef`、`RTC_DateTypeDef`，明确 `RTC_FORMAT_BIN` 或 `RTC_FORMAT_BCD`。读时间与日期按照当前 F1 驱动要求配对，避免把其他系列影子寄存器规则直接套用。

需要闹钟中断时，设置 `HAL_RTC_SetAlarm_IT()`，检查 RTC/EXTI/NVIC 链路、对应 IRQ 的 HAL 处理函数以及 `HAL_RTC_AlarmAEventCallback()`。不需要闹钟时不启用中断。

验证分别覆盖正常计时、跨日、软件复位、主电源断电但 VBAT 保持，以及备份域失效后的恢复。STOP/STANDBY 唤醒另见[低功耗](watchdogs-power.md)。
