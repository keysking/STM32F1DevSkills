# STM32F1DevSkills

面向 **STM32F1 CMake 工程**的七技能包：环境、工程、配置、固件、构建、烧录与调试。使用传统 STM32CubeMX、STM32CubeF1 HAL/LL、`cube` 与原生工具。

内容中文，采用简短入口和按场景查阅的命令/API 参考。面向 Windows、Linux、macOS，覆盖裸机及 FreeRTOS 原生 API、CMSIS-RTOS v1/v2。新建默认 CMake + Ninja + GCC、裸机；已有 CMake 工程保留其结构和 API。

本包整体安装，包内允许相互引用，独立于 STM32C5DevSkills 和其他技能包。它提供指令与参考资料，不包含工具、SDK、固件二进制或运行包装脚本。

## 安装

先克隆；下面两种方式选一种，避免重复发现技能：

```text
git clone https://github.com/keysking/STM32F1DevSkills.git
cd STM32F1DevSkills
```

### 标准技能目录

将七个 `stm32f1-*` 目录一起复制到目标工程的 `.agents/skills/` 或客户端支持的用户级目录，保留 references、agents 和 LICENSE。不能只复制单个技能或 SKILL.md。

PowerShell，在克隆目录执行：

```powershell
$project = 'D:/firmware/Control' # 替换为实际目标工程根目录
if (-not (Test-Path -LiteralPath $project -PathType Container)) { throw '目标工程不存在' }
$dest = Join-Path $project '.agents/skills'
$skills = @(Get-ChildItem -LiteralPath './skills' -Directory -Filter 'stm32f1-*')
if ($skills.Count -ne 7) { throw '请在完整仓库根目录执行' }
foreach ($skill in $skills) {
    if (Test-Path -LiteralPath (Join-Path $dest $skill.Name)) { throw "已存在 $($skill.Name)，先比较已有版本" }
}
New-Item -ItemType Directory -Path $dest -Force | Out-Null
foreach ($skill in $skills) {
    Copy-Item -LiteralPath $skill.FullName -Destination $dest -Recurse
}
```

Linux/macOS Bash，在克隆目录执行：

```bash
project='/work/Control' # 替换为实际目标工程根目录
test -d "$project" || exit 1
dest="$project/.agents/skills"
skills=(skills/stm32f1-*)
test "${#skills[@]}" -eq 7 || exit 1
for skill in "${skills[@]}"; do
  test ! -e "$dest/$(basename "$skill")" || { echo '已有同名技能，请先比较'; exit 1; }
done
mkdir -p "$dest"
for skill in "${skills[@]}"; do cp -R "$skill" "$dest/"; done
```

重新加载客户端或开始新会话，确认发现七个 `stm32f1-*` 技能。其他支持 SKILL.md 的客户端按其规则安装完整目录。

### Codex 插件

插件标识 `stm32f1-dev-skills`，显示名 `STM32F1DevSkills`，版本 `0.1.1`，要求本地执行环境。根 `plugin.json` 和 `.codex-plugin/plugin.json` 使用同一份技能。

在克隆目录使用支持插件命令的 Codex CLI：

```text
codex plugin marketplace add .
codex plugin list
codex plugin add stm32f1-dev-skills@stm32f1-dev-skills
codex plugin list
```

重启会话确认启用和七技能发现。调用名可能带 `stm32f1-dev-skills:` 命名空间，以客户端实际名称为准。不支持命令时先查 `codex plugin --help`，或采用标准目录安装。

此为本地 marketplace 安装，不代表上架公共插件目录。结构依据：[官方插件打包](https://developers.openai.com/plugins/build/plugins)、[本地执行声明](https://developers.openai.com/plugins/deploy/desktop-only)。

## 前置工具

- 传统 STM32CubeMX 和匹配的 STM32CubeF1 固件库；F1 配置为 `.ioc`。
- CMake、Ninja、arm-none-eabi GCC；可通过 `cube` 发现并使用工具 Bundle。
- 下载和调试按需使用 STM32CubeProgrammer、ST-Link server、GDB 及实际硬件。
- FreeRTOS 工程需要相应内核、CMSIS 适配层或原生 API，以及正确的 Cortex-M3 port。

工具不必预先全部在 PATH。先用[环境技能](skills/stm32f1-env/SKILL.md)定位，再按[传统 CubeMX 启动](skills/stm32f1-env/references/mx-cli.md)运行实际安装。当前 `cube mx` 可能指向 CubeMX2，不能由命令名称推断适用于 F1。

## 七项能力

| 技能 | 内容 | 请求示例 |
| --- | --- | --- |
| [stm32f1-env](skills/stm32f1-env/SKILL.md) | 工具、Java、固件库、版本与三平台定位 | 找出传统 CubeMX，并检查 F1 构建环境。 |
| [stm32f1-project](skills/stm32f1-project/SKILL.md) | 准确选型、新建、接手与重生成 | 为 STM32F103C8T6 建立 CMake 工程。 |
| [stm32f1-configure](skills/stm32f1-configure/SKILL.md) | 引脚、时钟、外设、DMA/NVIC、RTOS | 按我的接线修改 USART1 参数，保留其他配置。 |
| [stm32f1-firmware](skills/stm32f1-firmware/SKILL.md) | HAL/LL、移植、任务、存储与生成边界 | 为已有接收 DMA 实现帧处理和错误恢复。 |
| [stm32f1-build](skills/stm32f1-build/SKILL.md) | 预设、目标、链接、ELF/map 与资源 | 把应用模块加入 CMake，并检查 C8 内存占用。 |
| [stm32f1-flash](skills/stm32f1-flash/SKILL.md) | 目标选择、下载、校验与启动 | 将本次 ELF 下载到指定 ST-Link 的 F1 板。 |
| [stm32f1-debug](skills/stm32f1-debug/SKILL.md) | GDB、异常、外设、DMA 与任务故障 | 程序只接收一次数据，保留现场定位原因。 |

## 三条工作流

1. **传统 CubeMX 新建与维护**：定位工具 → 准确选型/加载 IOC → 按需求配置 → 保存重载 → 生成比较 → 应用接入和构建。见[创建工程](skills/stm32f1-project/references/create.md)与[重生成](skills/stm32f1-project/references/regenerate.md)。
2. **官方 SDK/例程接入 CMake**：识别例程型号、依赖和硬件假设，已有 CMake 先建立基线；非 CMake 例程只有明确接入需求时提取所需源和初始化。见[已有工程与例程](skills/stm32f1-project/references/existing.md)。
3. **手写或无生成器 CMake 工程**：保留初始化和构建所有权，直接使用实际 HAL/LL/既有驱动；不强制生成 IOC。

仅在任务需要时进入烧录/调试。本包不提供 Keil、IAR、传统 Makefile 工程的维护路线。

配置参考包括 GPIO/EXTI、AFIO/SWD、时钟、UART、I2C/SPI、定时器/PWM/捕获、ADC、固定 DMA、中断、CAN、USB Device、RTC、看门狗、低功耗和 FreeRTOS。按当前任务读对应章节，不要求加载全包。

## 命令参考的使用规则

- 每节是特定条件下的操作参考，整篇文档不是待逐条执行的演示。只查询时不修改；输出/中断、HAL/LL、轮询/IT/DMA 等按需求分别选择。
- 型号、引脚、频率、实例、地址、路径和缓冲区大小须来自真实需求、板卡和现有工程。文中具体值用于表达语法和约束，不是默认配置。
- 先查当前模式、参数、帮助与实际 SDK 声明。工具查询缺项或报错时按文中方法查数据库/生成结果，不能猜测或跨系列套用。
- 修改后回读并保存、独立重载；需要同步固件时生成、比较、构建。成功日志、配置落盘、代码正确和硬件功能分别检查。
- API 片段需应用提供数据、生命周期、超时及错误处理，不要求复制演示业务。
- shell 路径由 Agent 从发现结果直接赋值；示意路径须替换，不能等待无交互终端的输入。新建开发工程默认保留 SWD，其他引脚/调试需求按真实约束调整。

## 适用条件与边界

命令参考基于传统 CubeMX 6.18.1、STM32CubeF1 1.8.7 / HAL 1.1.10、GCC 14.3.1、CMake 4.3.1；FreeRTOS 依据该 SDK 内核及适配层。实际项目版本优先，升级后重新查询。

`STM32F103C8T6` 是主要核对型号，不是任意 F1 请求的默认芯片。其额定 Flash 为 64 KB、RAM 20 KB；共享模型名 `STM32F103C(8-B)Tx` 和宏 `STM32F103xB` 不意味着应选择 CB。F100/F101/F102/F105/F107 及其他容量型号的时钟、引脚、外设和内存按实际器件确认，不能直接套用 C8 参数。

| 范围 | 使用边界 |
| --- | --- |
| F103C8 常见开发 | 提供具体 CubeMX 参数和 HAL/LL/API 操作参考；硬件电气与运行结果按实际板卡验证。 |
| 其他 F1 型号/容量 | 复用工具发现、工程与 CMake 方法；外设、时钟节点、引脚、DMA映射先查该型号，不宣称所有型号参数均已实测。 |
| FSMC、SDIO、Ethernet/LwIP、USB Host 等型号专属能力 | 当前不提供完整专题操作手册；从准确型号数据手册、对应 CubeF1 例程和驱动源码建立所需配置，不能套用 C8 模板。 |
| Windows / Linux / macOS | 统一方法配平台入口；Windows执行核对，Linux/macOS仍需在目标主机核实安装布局和运行依赖。 |

三平台均有操作指导；本版实际执行检查在 Windows，Linux/macOS 为文档和命令形式核对。构建、下载校验和板端功能是不同完成状态。外设电气、时序、任务和低功耗效果需在实际硬件验证。

## 许可

采用 [MIT](LICENSE)，版权主体 `STM32F1DevSkills contributors`。工具、SDK、ST、Arm 和 FreeRTOS 的名称及许可归各自权利人，本包不重新分发这些工具和库。
