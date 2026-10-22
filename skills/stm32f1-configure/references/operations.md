# 配置操作基础

所有场景都是命令用途与语法参考，按实际需求选择，不顺序执行整篇文件。参数来自用户要求、板卡约束、已有配置和当前工具查询。本文无法覆盖所有模式；以下查询及安装数据库用于补充未知参数。

## 建立上下文

本配置技能所列 `set pin`、`get mode`、`clock`、`dma` 等命令在**传统 CubeMX 会话/脚本**中执行，不是 shell 命令；其他技能中的 cube/cmake/GDB 指令按各页注明的环境执行。按[启动方法](../../stm32f1-env/references/mx-cli.md)运行。已有工程先 `config load "真实 IOC 路径"`、`waitclock 2`；只有新建才 `load <准确 RefName> [CPN]`。

记录 `get version`、准确型号与 SDK。命令路径以传统 CubeMX 6.18.1、CubeF1 1.8.7 的 F103C8T6 模型为具体参考；其他型号和版本先查询。已有项目版本优先，不要求升级到这些版本。

## 查询当前能力和参数

| 目的 | 会话命令形式 |
| --- | --- |
| 模型、封装、IP | `get mcu name`、`get mcu package`、`get mcu peripherals` |
| 引脚/候选信号 | `get pinout`、`get functions PA5`、`get signal PA5` |
| 模式 | `get mode available USART1`、`get mode active USART1` |
| 活动模式参数 | `get mode_param_list USART1 Asynchronous` |
| 枚举候选 | `get mode_param_possvalue USART1 Asynchronous WordLength` |
| 当前参数 | `get ip parameters USART1 BaudRate` |
| GPIO 参数 | `get gpio parameters PA5 .*` |
| 检验一个候选是否合法 | `possible_value ip parameters USART1 WordLength WORDLENGTH_9B` |
| DMA 请求/分配 | `dma available_periph_requests USART1`、`dma list_all` |
| IRQ | `NVIC get_ip_interrupts USART1`、`NVIC get_enable_state USART1_IRQn` |

实例与引脚替换为真实对象；不能为了查询先启用一个原本不需要的模式。某些参数仅在模式已配置时存在。

日志可能显示 `USART1 | BaudRate = 115200`，这是当前值而非推荐值。候选如 `WORDLENGTH_8B` 是 CubeMX 内部字符串，生成的 HAL 宏可能是 `UART_WORDLENGTH_8B`，两者不能互换。

引脚名称用查询结果：如 `PA0-WKUP` 而不只 `PA0`。带空格的模式/字段必须加引号，例如 `"PWM Generation1 CH1"`。`get ip parameters USART1 .*` 不是可靠的通用枚举入口。

## 仅在需要修改时写入

先查当前值和候选。以下只说明写法：若真实需求是把已有 USART1 改为 57600，执行：

```text
set ip parameters USART1 BaudRate 57600
get ip parameters USART1 BaudRate
```

新分配引脚用 `set pin <准确引脚> <信号>`；激活模式用 `set mode <实例> "准确模式"`。已有配置只改需求字段。启用外设后检查自动分配引脚，不认为它必然符合用户接线。

禁用仅在确实不再需要时使用 `reset mode <实例> "模式"`；释放指定引脚用 `reset pin <引脚>`。模式关闭可能遗留手动引脚、DMA、IRQ，先查所有者再释放，不批量清空。

## 查询不完整时

1. 先检查命令拼写、完整引脚名、带空格参数的引用和已加载模型。
2. 参数枚举的值可能位于 INFO 行；保留日志，不过滤掉结果。
3. FREERTOS、USB_DEVICE、IWDG/WWDG 的模式枚举可能 KO，仍可能支持配置；不要仅凭 KO 判定不存在。
4. RTC/TIM 等通配参数查询可能抛异常后继续执行。改查准确模式或字段，不能把部分输出当完整列表。
5. 根据 `get mcu peripherals` 或 `db/mcu/<共享模型>.xml` 中的 IP Name、Version，读取 `db/mcu/IP/<Name>-<Version>_Modes.xml`。核对 RefParameter、PossibleValue、Condition、RefMode 和目标条件；只读数据库。
6. 必要时查对应 `db/mcu/config/` 的生成映射和 `db/plugins/clock/` 的 ClockTree。共享模型/时钟树文件名可以与商业系列不同；F103 模型引用 F102 命名文件不一定选错。

一个字段出现在通用数据库不等于目标支持，`null` 也不总等于写入失败。以当前条件、IOC 保存值和生成代码交叉确认；如 TIM 多通道的 Pulse 字段见[TIM](timers.md)。无法确定时报告具体字段和证据，不能猜一个相似系列的参数写入。

## 修改完成标准

回读需求字段和共享资源，`config save`；独立进程重新加载同一 IOC，再核对。需要代码时按[重新生成](../../stm32f1-project/references/regenerate.md)生成、比较和构建。时钟还检查派生频率及 `isPluginError "Clock Configuration"`。命令 OK、保存成功、构建通过和实际功能是不同结果。
