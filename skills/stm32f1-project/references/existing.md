# 接手 CMake 工程与官方例程

先盘点，不改变工程：`git status`；查 `.ioc`、`CMakeLists.txt`、`CMakePresets.json`、`*.cmake`、`*.ld`、启动文件、`FreeRTOSConfig.h`。使用 `rg --files` 或平台文件查找命令；排除 build 和依赖缓存，避免把旧产物当作当前工程。

## 已有 CubeMX 工程

对照 IOC 的准确型号、固件包、生成器版本与 CMake 的设备宏/链接布局。读取用户区及自有 App/Drivers 目录，确认初始化和 IRQ 由谁维护。先尝试现有预设的基线构建，再处理本次需求；编译失败不能通过重新生成覆盖业务代码来消除。

原工程版本可用时沿用。新 CubeMX 打开旧 IOC 可能迁移配置；在可比较的副本中检查版本和差异，再决定合并。不把打开成功理解成无变化。

## 官方例程

STM32CubeF1 的 `Projects/<板卡>/Examples`、`Examples_LL`、`Examples_MIX` 和 `Applications` 可提供驱动、硬件连接及启动顺序参考。先查其 README、实际芯片、时钟、引脚、编译文件和依赖。

本包只接手已有 CMake 例程或把**用户要求的代码**集成进现有 CMake 工程；不假定所有 CubeF1 例程原生含 CMake，也不自动迁移 Keil/IAR 工程。参考板 F103RB 不等于 C8T6：GPIO、晶振旁路、内存、按钮和 LED 设置必须重新核对。

SDK 版本固定到工程要求；从 Git 获取时确保子模块源码齐全。没有目标板时，参考代码的编译检查不能代替外围电路验证。

## 无生成器、SPL 或寄存器工程

无 IOC 的 CMake 工程沿用现有启动和初始化，不新建 IOC 覆盖它。区分 `stm32f10x_*` SPL 与 `stm32f1xx_hal_*`，确认是否有同名类型、IRQ 和启动文件冲突。

首版对 SPL 提供识别与集成边界：日常改动查项目实际 API；只有用户要求迁移时逐模块替换，先建立可构建基线和明确中断/时钟所有者。不要同时调用两套系统初始化或把 HAL 句柄传给 SPL。

完成后应能说明准确型号、依赖、构建入口、生成边界、用户模块位置及当前故障。具体扩展见[移植与集成](../../stm32f1-firmware/references/porting.md)。
