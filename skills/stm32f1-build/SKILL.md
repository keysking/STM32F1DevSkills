---
name: stm32f1-build
description: 构建 STM32F1 CMake 固件，检查 Ninja/GCC 工具链、预设、源码目标、链接脚本、ELF/map 和资源占用，并定位编译链接错误。用于 F1 CMake 构建失败、应用模块接入和产物确认。
license: MIT
---

# STM32F1 构建

只考虑 CMake 工程。新建默认 Ninja + GCC；已有 CMake 工程先读取其预设、工具链和依赖，不重建为另一套布局，也不默认升级 SDK。

1. 确认源目录、准确芯片、编译器和构建预设。已有构建缓存中的路径与当前环境不符时使用新的构建目录，不随意删除源码或其他输出。
2. 按[CMake 构建与扩展](references/cmake.md)配置、构建并取得本次目标产物。
3. 出错时从首个有效错误定位，按[编译链接诊断](references/errors.md)区分路径、API、重复/缺失源码、ABI 和内存布局。
4. 核对 ELF 目标、向量表、Flash/RAM、map 和必要符号。C8T6 的额定 Flash 是 64 KB、RAM 20 KB；`STM32F103xB` 宏本身不能证明是 128 KB。

只构建不隐含烧录。报告实际预设、ELF 路径及静态资源占用；ELF 文件大小不等于 Flash 使用量，链接通过不等于运行时堆栈不会溢出。
