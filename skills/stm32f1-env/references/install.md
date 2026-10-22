# 缺失工具、固件库与依赖恢复

只有定位结果确认缺失、版本不符或损坏时才安装/恢复。先读取工程所需版本；下面各软件分别处理，不要求安装整套新环境。安装动作按用户任务授权执行，不用全量 update 替代诊断。

## 没有 cube

从 [ST 官方安装说明](https://dev.st.com/stm32cube-docs/stm32cubecli/latest/en/docs/markup/STM32CubeCLI_Getting_Started/STM32CubeCLI_Installation.html)取得当前主机适用的安装脚本。Windows PowerShell 可先下载到本次工作目录、检查内容，再执行该本地脚本：

```powershell
Invoke-WebRequest 'https://developer.st.com/bundles/install/install.ps1' -OutFile './install-cube.ps1'
Get-Content -LiteralPath './install-cube.ps1'
```

确认来源与安装内容后，执行 `& './install-cube.ps1'`。Linux/macOS 对应获取方式：

```bash
curl -fL 'https://developer.st.com/bundles/install/install.sh' -o install-cube.sh
cat install-cube.sh
```

确认后执行 `bash ./install-cube.sh`。按实际安装输出更新当前进程 PATH 或重新打开终端，再执行 `cube --version`、`cube --help`。下载失败先查网络/TLS/代理/主机架构，不反复用管理员权限执行脚本；不因 PATH 未刷新再次安装。

## 只有某个构建工具缺失

先查询当前平台的可用 Bundle 与已安装路径：

```text
cube bundle list-online
cube --json bundle list
cube bundle install --help
```

从输出获取名称、支持平台和版本。常见名称有 `cmake`、`ninja`、`gnu-tools-for-stm32`；它们是 Bundle 标识，不保证每一个都提供同名可执行命令。只在实际需要安装所查版本时执行 `cube bundle install "名称@版本"`，引号内换成完整标识。例如工程确需且目录存在该版本时：

```text
cube bundle install "cmake@4.3.1+st.1"
```

没有版本约束且用户允许选当前版本时可省略 @版本；否则不静默换最新版。需要项目范围时在工程根使用 `cube bundle install --project "名称@版本"`，已有 bundle 项目先查 `cube bundle list --project` 和依赖声明，不能重新 init 覆盖选择。缺少项目声明且确需建立时先查 `cube bundle init --help`，再用 `cube bundle init --project` 创建。

非交互环境在版本与安装范围已明确后可给安装命令加 `--yes`；这会接受安装提问，应先确认许可及实际下载范围。安装后用 `cube --list`、版本命令和 `cube --json bundle list` 核对解析结果；GCC/Ninja 仍需按[构建参考](../../stm32f1-build/references/cmake.md)提供给 CMake 子进程。

已有包异常时，先 `cube bundle --help` 查看当前版本诊断与修复入口，结合官方说明按失败对象处理；不要直接 remove/autoremove 全部依赖。部分子命令不处理自己的 `--help`，可能直接执行动作，不能对 fix 等修改入口以试探帮助的名义运行。项目声明、实际解析版本和 CMakeCache 三者应一致；换编译器后使用新的构建目录。

## 缺少传统 CubeMX

从 [STM32CubeMX 产品下载页](https://www.st.com/en/development-tools/stm32cubemx.html)选择**传统 STM32CubeMX**与主机 OS/架构对应发行包；页面同时可能列出 CubeMX2，F1 `.ioc` 路线不因此改用 CubeMX2。Windows 解压并运行发行包内安装器；Linux 使用该发行包的 Linux 安装入口；macOS 打开对应应用/安装包。具体文件名随版本变化，以发行包内容和安装说明为准。

需要账户下载时让官方认证流程处理，不把密码放进脚本。安装后定位运行时、启动配置和 db/mcu，按[mx-cli](mx-cli.md)执行仅含 `get version`、`help`、`exit` 的脚本。`cube mx` 若仍映射 CubeMX2，不替换它；直接调用已定位的传统 CubeMX。

## 缺少 CubeF1 固件库

CubeMX GUI 的 Help → Manage embedded software packages 中选择 STM32F1 与工程要求的版本；离线包使用该管理器的 From Local/本地安装入口，安装后检查 repository 中对应目录。版本标签和安装器菜单可能随发行版变化，以当前界面为准。

脚本方式先在 CubeMX 会话查询 `swmgr help`。当前帮助给出的形式为 `swmgr install <完整固件名称或本地包路径> <deny|ask>`；例如仅当已取得对应本地包且当前环境可处理许可提问时：

```text
swmgr install "D:/downloads/stm32cube_fw_f1_v187.zip" ask
```

该路径是语法实例；以实际文件为准。`ask` 可能等待许可交互，不能放入无人值守 Agent 命令并假定成功；`deny` 也不是自动接受。无人值守任务优先使用已安装库或先完成官方管理器的安装步骤，不编造 `accept` 参数。

确需采用 Git 源码时，在新的目标目录获取官方仓库：

```text
git clone --recursive https://github.com/STMicroelectronics/STM32CubeF1.git
git -C STM32CubeF1 tag --list
```

选择与工程一致的真实 tag/commit 后执行 checkout，再 `git -C STM32CubeF1 submodule update --init --recursive`。已有仓库先检查本地修改，不为了补子模块丢弃工作。Git checkout 与 ST 发布包的目录/版本元数据可能不同，先检查 package.xml、Drivers/CMSIS、HAL 源和所需 Middlewares；缺项不使用别系列库补齐。

在实际 IOC 中用 `project setCustomFWPath "实际库根目录"` 指向完整库，保存并重载，核对 FirmwarePackage/CustomerFirmwarePackage 和生成引用。最后用最小需要的工程构建确认恢复成功；只找到目录不算依赖完整。

## 探针驱动或权限

只有下载/调试需要时补充 Programmer/ST-Link 工具。Windows 先从 `cube --list` 查是否有 `stlink-usb-driver-install-check` 并读其帮助，或使用 ST 官方发行包的驱动安装入口；不要因目标未连接就自动重装驱动。Linux 使用 ST 安装包附带 udev 规则并确认用户 USB 权限；macOS 核对工具二进制主机架构和 USB 访问。枚举探针只证明主机入口可用，连接目标、复位和烧录仍按具体任务进行。

依据：[Bundle 安装说明](https://dev.st.com/stm32cube-docs/stm32cubecli/latest/en/docs/markup/STM32CubeCLI_Getting_Started/STM32CubeCLI_Bundles.html)、当前 CubeMX 帮助及[官方 CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)。
