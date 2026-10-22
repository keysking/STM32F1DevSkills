---
name: stm32f1-project
description: 创建、接手或整理 STM32F1 的 CMake 工程，识别准确料号、传统 CubeMX 配置、HAL/LL 和 SDK 依赖，处理保存、重载与重新生成。用于新建 F1 工程、理解已有目录及维护生成边界。
license: MIT
---

# STM32F1 工程

只覆盖 CMake。新建默认传统 CubeMX + STM32CubeF1 HAL + Ninja + GCC、裸机；已有 CMake 工程保留其组织和驱动层。无 `.ioc` 不代表工程损坏，不自动补建生成器配置或迁移驱动。

先确认完整料号、封装、存储容量、板级时钟及当前修改。STM32F103C8T6 是文档中的具体语法对象，不是所有新建任务的默认型号。缺少芯片信息时先查工程或询问，不以常见开发板推断。

按任务读取：

| 任务 | 资料 |
| --- | --- |
| 新建、正确选择 C8T6、建立 CMake | [创建工程](references/create.md) |
| 已有工程、官方例程、无生成器/SPL 识别 | [接手工程](references/existing.md) |
| 修改 IOC 后生成、用户代码保留、生成异常 | [重新生成](references/regenerate.md) |

CubeMX 路径或启动方法不明时直接读[启动与脚本会话](../stm32f1-env/references/mx-cli.md)。已确认环境则继续，不重复走整个环境技能。

完成标准：准确型号与链接容量一致，依赖可定位，CMake 配置与构建通过；重新生成时检查用户代码和构建扩展的差异。空工程不附带 LED、串口、LL 或 RTOS。构建通过不代表已下载或验证硬件。
