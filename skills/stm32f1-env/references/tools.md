# 工具定位与依赖

按任务查找需要的工具；本页不是要求安装所有软件的清单。先复用工程给出的路径和版本，再按以下顺序查找。目录和 Bundle 名称从实际输出取得，不把某台机器的安装布局写进工程。

## 定位 cube 与原生工具

PowerShell 查询：

```powershell
Get-Command cube,cmake,ninja,arm-none-eabi-gcc,STM32_Programmer_CLI -ErrorAction SilentlyContinue |
    Select-Object Name,Source
```

Bash 查询：

```bash
command -v cube cmake ninja arm-none-eabi-gcc STM32_Programmer_CLI
```

找到 cube 后查询，不立即安装或升级：

```text
cube --help
cube --list
cube --json bundle list
cube bundle --help
cube cmake --version
cube ninja --version
cube programmer --help
```

`--list` 显示当前可调用入口；Bundle JSON 的 `details.name`、版本和 `path` 用于定位实际安装。按返回的目录查 GCC 的 `bin/arm-none-eabi-gcc`、Ninja、GDB 等。需要改变 Bundle 时按[安装与恢复](install.md)查询当前接口和项目版本，不把当前列表视为固定全集，也不假设所有修改子命令都会处理 `--help`。

`cube` 可找到 CMake 不等于 CMake 子进程可找到 GCC 和 Ninja。构建前将已确认的编译器 bin 与 Ninja 所在目录加入**当前构建进程** PATH；方法见[CMake](../../stm32f1-build/references/cmake.md)。不要修改系统 PATH 来掩盖工程缓存中的旧编译器路径。

## 定位传统 CubeMX

`cube --list` 和 Bundle 路径若指向 CubeMX2，不将其作为 F1 生成器。独立安装的传统 CubeMX 可能完全不在 PATH。

Windows 按顺序查：

1. `Get-Command STM32CubeMX -ErrorAction SilentlyContinue`。
2. 卸载记录中的安装位置、启动菜单快捷方式目标。
3. 有限候选目录，例如 `$env:LOCALAPPDATA/Programs/STM32CubeMX`、`$env:ProgramFiles/STMicroelectronics/STM32Cube/STM32CubeMX`。不存在再按安装记录扩展，不全盘递归扫描。

读取卸载记录的形式：

```powershell
$keys = @(
  'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
  'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
  'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
)
Get-ItemProperty $keys -ErrorAction SilentlyContinue |
  Where-Object DisplayName -Match 'STM32CubeMX' |
  Select-Object DisplayName,DisplayVersion,InstallLocation,DisplayIcon
```

注册表版本可能滞后；实际版本用 CubeMX 会话 `get version` 确认。目录内核对 `STM32CubeMX.exe`、自带 `jre`、启动配置和 `db/mcu`。

Linux 先 `command -v STM32CubeMX`，再看已安装的 `.desktop` 启动项 `Exec=` 以及用户指定的安装目录，常见父目录为 `$HOME`、`/opt`；macOS 查看 `/Applications`、`$HOME/Applications` 中的应用包及其 `Contents`。仅在候选目录下查实际可执行文件，不预设每版 bundle 内部路径/大小写相同。启动方式见[mx-cli](mx-cli.md)。

## STM32CubeF1 与工程依赖

传统 CubeF1 固件库通常位于用户配置的 STM32Cube Repository，候选位置为用户主目录下 `STM32Cube/Repository/STM32Cube_FW_F1_V*`。以实际 CubeMX 设置、IOC 的 `ProjectManager.FirmwarePackage` / `ProjectManager.CustomerFirmwarePackage`、CMake 路径和 `package.xml` 为准。

- CubeCLI 工具 Bundle、CubeMX 的 STM32CubeF1 固件库、CMSIS-Pack 是不同对象；找到编译器 Bundle 不等于 HAL/FreeRTOS 库已安装。
- 指定已有固件库时可在 CubeMX 会话调用 `project setCustomFWPath "实际固件包目录"`，再保存并核对 IOC 路径与生成源码。目录应包含 `Drivers`、`package.xml`，按需求包含 `Middlewares`。
- 缺包时按[安装与恢复](install.md)选择 ST 管理器、本地包或官方 Git 源；`swmgr` 的安装格式与许可交互先读当前会话帮助，不把可能等待输入的 ask 直接交给非交互进程。
- 使用 Git 获取官方 STM32CubeF1 时检查嵌套依赖是否完整；不要看到顶层目录就以为 HAL、CMSIS、RTOS 已有源码。已有发布包和 Git checkout 的布局/版本约束可能不同。
- FreeRTOS 使用工程真实内核、适配层和 Cortex-M3 port。数据库版本标签不等于实际内核版本。

失败判断：生成器找不到包看指定路径/包版本；编译找不到头文件看 CMake include 与依赖完整性；工具能启动但无法访问探针看驱动/权限。Linux 按 ST 安装包配置 udev 与用户权限；macOS/Windows 核对支持的主机架构与 USB 驱动，不自动用管理员权限重试所有命令。

依据：[STM32CubeCLI](https://dev.st.com/stm32cube-docs/stm32cubecli/latest/en/index.html)、[STM32CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)。
