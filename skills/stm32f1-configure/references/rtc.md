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

仅当目标为对应 F103 时钟模型、板上确有 32768 Hz LSE 晶体且需求使用它时，在已启用 RTC 的会话配置：

```text
set mode RCC LSE-External-Oscillator
clock set RTCClkSource 1
set ip parameters RTC AutoPrediv RTC_AUTO_1_SECOND
get ip parameters RTC AutoPrediv
get ip parameters RTC AsynchPrediv
isPluginError "Clock Configuration"
```

1 是当前 F103 模型的 LSE 选择索引，不是频率；其他型号/源先查实际 ClockTree。保存后 IOC 应为 `RCC_RTCCLKSOURCE_LSE`，生成应启用 LSE 并选择对应 RTC 时钟。自动预分频按 HAL 得到的 RTC 输入频率减1；32768 Hz 对应32767。需要手动预分频时，先 `set ip parameters RTC AutoPrediv 0`，再 `set ip parameters RTC AsynchPrediv 32767`，数值按实际输入/目标 tick 重算。自动与手动是替代方案。

不需要 RTC 输出占用引脚时，查当前模式并选 `set mode RTC RTC_OUT_NO`。不要为了日历功能无意启用 PC13 的报警/校准输出。精确参数查询失败时读 RTC Modes.xml 与生成代码，不能把 `get ip parameters RTC .*` 的半截输出当作完整结果。

改 RTC 时钟源可能需要备份域复位，并丢失 RTC/备份寄存器数据。先确认保留要求，按实际恢复方案执行，不把复位当成常规初始化步骤。

## 初始化与掉电保持

检查生成的 `MX_RTC_Init()` 是否每次执行 `HAL_RTC_SetTime()`/`HAL_RTC_SetDate()`。无条件设置会覆盖已有时间；需要保持时，在可保留的用户代码区根据备份标记、时钟源有效性和应用策略判断是否首次设置。标记不是时间有效性的唯一证据，电池耗尽和备份复位后要重新建立有效状态。

F1 RTC 的硬件核心是计数器；HAL 日历中的日期维护含软件状态。不能承诺调用 SetDate 后完整日期就会由 VBAT 永久保存。检查项目 HAL 的 `DateToUpdate`、备份寄存器使用和跨日更新路径，按需求保存日期/纪元及恢复依据。

需要备份标记时，选择实际空闲的 BKP 寄存器，不覆盖已有用途。F1 备份寄存器按16位数据规划；下列是单次读/写的替代操作，mark来自应用协议：

```c
uint32_t mark = HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR1);
```

```c
HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR1, mark);
```

调用前启用 PWR/BKP 时钟并允许备份域写访问（写入时）；不调用备份复位来获得写权限。标记应在实际初次设置成功并保存必要元数据后写入。当前生成的 `Check_RTC_BKUP` 用户区位于 HAL_RTC_Init 之后、默认 SetTime/SetDate 之前；可在此恢复已有软件日期状态并按需求跳过默认设时，但须检查之后是否还有必须执行的用户初始化。不能在更早的用户区简单 return 而跳过句柄初始化。将“硬件/句柄初始化”“首次设时”“已有计数及日期恢复”分开，在自有函数/可保留用户区安排，并检查生成差异。

需要跨复位日期时，设计备份日期或纪元、版本和校验及更新时间策略；HAL_RTC_Init 会重置句柄的 DateToUpdate，必须恢复软件状态。单个魔数只能说明曾初始化，不能恢复日期，也不能证明停电期间时间可靠。

## API 与闹钟

按当前 HAL 头文件构造 `RTC_TimeTypeDef`、`RTC_DateTypeDef`，明确 `RTC_FORMAT_BIN` 或 `RTC_FORMAT_BCD`。读时间与日期按照当前 F1 驱动要求配对，避免把其他系列影子寄存器规则直接套用。

只有用户需要设时且输入已验证范围时，依次调用并检查 `HAL_RTC_SetTime(&hrtc, &time, RTC_FORMAT_BIN)`、`HAL_RTC_SetDate(&hrtc, &date, RTC_FORMAT_BIN)`；time/date 用实际输入构造，不把文档日期写进固件。读取时：

```c
RTC_TimeTypeDef time = {0};
RTC_DateTypeDef date = {0};
HAL_StatusTypeDef status = HAL_RTC_GetTime(&hrtc, &time, RTC_FORMAT_BIN);
if (status == HAL_OK) status = HAL_RTC_GetDate(&hrtc, &date, RTC_FORMAT_BIN);
```

status==HAL_OK 后再使用结果。F1 GetTime 路径参与跨日的软件日期更新，应用读数与持久化需要一致，不能只周期读Date而忽略Time更新。

需要闹钟中断时，设置 `HAL_RTC_SetAlarm_IT()`，检查 RTC/EXTI/NVIC 链路、对应 IRQ 的 HAL 处理函数以及 `HAL_RTC_AlarmAEventCallback()`。不需要闹钟时不启用中断。

验证分别覆盖正常计时、跨日、软件复位、主电源断电但 VBAT 保持，以及备份域失效后的恢复。STOP/STANDBY 唤醒另见[低功耗](watchdogs-power.md)。
