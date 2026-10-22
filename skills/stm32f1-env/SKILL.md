---
name: stm32f1-env
description: 定位并启动 STM32F1 开发所需的传统 CubeMX、Java、cube、STM32CubeF1、GCC、CMake 和 Ninja，恢复依赖并排查环境故障。用于工具缺失、版本冲突或不清楚 CubeMX.exe 如何操作的任务。
license: MIT
---

# STM32F1 环境

F1 使用传统 CubeMX 的 `.ioc` 和 STM32CubeF1 的 HAL/LL，内核为 Cortex-M3。`cube` 是工具发现与执行入口；先查其映射，不能把解析到 CubeMX2 的 `cube mx` 用来操作 F1 `.ioc`。

只处理当前任务需要的环境。复用已经确认的路径和版本，不因每次编译重新扫描整机或升级全部依赖。本包只覆盖 CMake 工程，新建默认 Ninja + GCC、裸机。

1. 从现有 `.ioc`、CMake 预设、工具链文件和依赖声明确定所需版本；新工程先确认完整料号。
2. 缺工具或路径时读取[工具与依赖](references/tools.md)。按 PATH、安装记录、有限候选目录定位，再执行版本/帮助确认身份。确认尚未安装或依赖损坏时，读[安装与恢复](references/install.md)。
3. 需要操作 CubeMX 时读取[三平台启动与脚本会话](references/mx-cli.md)，使用当前安装的运行时和启动配置。
4. 不清楚型号、容量或库层时读取[F1 基础与型号](references/f1-basics.md)。不要用 C8 的能力代替整个系列。
5. 用版本查询、工程依赖路径和最小构建检查闭环。只有任务确实包含创建、配置或下载时才进入对应技能。

完成时给出可直接调用的工具路径、版本、SDK 位置及尚缺条件；区分工具可运行、工程可构建和板端可用。不把探针枚举等同于已经识别芯片。
