# 创建 F1 CMake 工程

仅用于用户要求新建工程。先确认完整料号、封装、板级时钟和目标目录；文中的 C8T6、Control、路径是语法实例，按需求替换。新建默认 HAL、裸机、CMake/Ninja/GCC，不附带外设或 RTOS。

## 选择准确器件

CubeMX 选择器中的商业料号、CLI 的器件参考名和内部共享模型名不是同一个字段。C8T6 的已知对应关系：

| 字段 | C8T6 对应值 |
| --- | --- |
| 商业料号 CPN | STM32F103C8T6 |
| 器件参考名 RefName | STM32F103C8Tx |
| 共享数据库模型 | STM32F103C(8-B)Tx |

当前传统 CubeMX 支持以下加载形式，仅在目标确为 C8T6 时使用：

```text
load STM32F103C8Tx STM32F103C8T6
waitclock 2
```

单参数 `load STM32F103C8Tx` 也可选到对应器件。不要直接用商业料号当 XML 名称执行 `load STM32F103C8T6`；也不要用共享 `STM32F103C(8-B)Tx` 代替准确参考名，它可能默认到 CB。

其他型号先在安装的 `db/mcu/families.xml` 查询 RefName，再对应商业料号。必要时只读查看 MCU Finder 数据库 `db/plugins/mcufinder/mcu/cube-finder-db.zip` 内的数据库或用 GUI 选择器核对；数据库是查询依据，不修改安装文件来“支持”未知器件。

## 设置工程与保存

确认 SDK 为目标工程需要的已安装 CubeF1 版本。以下会话命令中的目录仅为 Windows 语法说明；Linux/macOS 使用相应绝对路径。创建前确认目标子目录不存在或为空，不覆盖已有工程。

本节命令接在上节 `load`/`waitclock` 后，在同一会话或脚本中执行；不能省略加载器件直接运行工程设置。

```text
project name Control
project path "D:/firmware"
project toolchain CMake
project compiler GCC
project setCustomFWPath "D:/SDK/STM32Cube_FW_F1_V1.8.7"
config save
exit
```

需要 SWD、外设或指定时钟时，仅加入实际需求对应的配置。新工程默认内部时钟不是对板卡外部晶振的确认。

实际保存路径通常包含工程名，例如 `D:/firmware/Control/Control.ioc`。找到真实文件，不只相信传给 `config save` 的路径；未设置工程名/目录的全新会话可能等待交互。

对 C8T6 检查：`Mcu.CPN=STM32F103C8T6`、`Mcu.UserName=STM32F103C8Tx`、`ProjectManager.DeviceId=STM32F103C8Tx`。`Mcu.Name=STM32F103C(8-B)Tx` 为共享模型，单独出现它不是选错。若身份字段冲突，回到准确参考名重新创建，不只手改一个名称掩盖错误。

## 独立重载、生成与构建

用新进程执行：

```text
config load "D:/firmware/Control/Control.ioc"
waitclock 2
project generate
config save
exit
```

使用新进程避免新建会话中的生成状态问题；日志无异常且生成的路径、源文件存在才继续。`Src/Inc` 和 `Core/Src` 均可能有效，不为迎合模板无条件改目录。

检查 `CMakeLists.txt`、`CMakePresets.json`、工具链、启动文件和链接脚本。C8T6 应为 Cortex-M3、`STM32F103xB`、Flash 64K、RAM 20K。按[构建指导](../../stm32f1-build/references/cmake.md)完成 configure/build 并核对 ELF。遇到重复绝对路径、sysmem/syscalls 缺失等见[生成故障](regenerate.md)。
