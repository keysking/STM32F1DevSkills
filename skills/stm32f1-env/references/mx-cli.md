# 启动传统 CubeMX 与执行脚本

本页说明工具入口；查询与修改脚本分开准备，只执行任务所需命令。F1 路线是传统 CubeMX，不是 CubeMX2；不能把 `cube mx` 的工程/参数命令套到这里。

## Windows

先按[工具定位](tools.md)找到安装目录。部分传统 CubeMX Windows 发行版的 `STM32CubeMX.exe` 本身可作为 Java 归档运行，这是该工具特有的布局，不是所有 exe 都能如此启动。

核对安装自带 `jre/bin/java.exe`、`STM32CubeMX.l4j.ini` 等启动配置，读取其 JVM 参数。不要依赖 PATH 中碰巧存在的 Java。以下是 Java 21 配套发行版的启动形式；`$mxRoot`、`$script`、`$log` 均由当前任务提供的真实绝对路径赋值：

```powershell
$mxRoot = Read-Host '传统 CubeMX 安装目录'
$script = (Resolve-Path -LiteralPath (Read-Host '已准备的 CubeMX 脚本')).Path
$log = Read-Host '本次日志文件的绝对路径'
$java = Join-Path $mxRoot 'jre/bin/java.exe'
$jar = Join-Path $mxRoot 'STM32CubeMX.exe'
if (!(Test-Path -LiteralPath $java) -or !(Test-Path -LiteralPath $jar)) {
    throw '安装布局不符，先定位该安装的运行时与入口'
}
$jvm = @(
    '--add-exports=java.desktop/sun.awt=ALL-UNNAMED',
    '--add-opens=java.desktop/java.awt=ALL-UNNAMED',
    '-Djavax.net.ssl.trustStoreType=WINDOWS-ROOT',
    '-Dsun.java2d.d3d=false', '-Dfile.encoding=UTF8'
)
Push-Location $mxRoot
try { & $java @jvm '-jar' $jar '-q' $script 2>&1 | Tee-Object -FilePath $log }
finally { Pop-Location }
```

JVM 选项在 `-jar` 之前，CubeMX 参数在归档路径之后。若安装自带启动配置不同，按其配置调整；发生模块访问或 Java 版本错误时先对照运行时，不盲目换系统 Java。

## Linux / macOS

使用实际安装中的启动器，先切到它要求的工作目录。Linux 常见形式是安装目录下 `./STM32CubeMX`；macOS 使用应用包 `Contents` 内的实际启动器，先查看路径，注意 `MacOS`/`MacOs` 等名称的大小写。若发行版要求 Java 启动，使用其自带运行时及配置。

以下 Bash 中变量取实际路径；`script`、`log` 为绝对路径，`launcher` 是相对于该工作目录的真实启动器：

```bash
read -r -p 'CubeMX 启动器工作目录: ' mx_dir
read -r -p '启动器路径: ' launcher
read -r -p '脚本绝对路径: ' script
read -r -p '日志绝对路径: ' log
test -f "$script" || exit 1
cd "$mx_dir" || exit 1
set -o pipefail
"$launcher" -q "$script" 2>&1 | tee "$log"
```

`-i` 进入交互模式；`-q` 执行安静脚本，`-s` 是普通脚本模式。第一次定位可用只含 `get version`、`help`、`exit` 的脚本。没有图形会话时是否可启动由当前发行版依赖决定；不能仅凭 `-q` 推断它没有 GUI 运行时依赖。

## 脚本上下文与结果

UTF-8 文本，每行一条 CubeMX 会话命令，以 `exit` 结束。会话命令不是 PowerShell/Bash 命令。引用路径与带空格的模式名；不要把 `# 注释`、shell 变量声明或 Markdown 围栏混进脚本，除非已确认当前解析器支持。

已有工程查询的脚本形式（替换 IOC 路径）：

```text
config load "D:/firmware/Control/Control.ioc"
waitclock 2
get version
get mcu name
get mcu package
get mode active USART1
exit
```

这里没有 save/generate；纯查询不需要生成。新建流程见[创建工程](../../stm32f1-project/references/create.md)。不同脚本进程不会共享已加载模型，因此每个进程都必须明确 `load` 或 `config load`。

- 留存 stdout/stderr，INFO 行也可能包含有效参数值；不要一律删掉带时间戳的行。
- 检查命令旁的 OK/KO、异常、生成结果。Java 进程退出 0 不保证每条命令成功；发生异常后脚本可能继续。
- 裸命令用于查询 Usage 时出现 KO 与完整修改命令失败不同，分开解释。
- 长时间无输出时检查是否在等保存位置、登录或弹窗。只终止本次启动且确认卡住的子进程，不结束用户其他 CubeMX/Java 会话。
- 避免同一 IOC 同时被 GUI 和脚本编辑。涉及驱动切换或多项目时优先独立进程加载目标，减少上下文残留影响。

命令入口依据：[CubeMX CLI](https://dev.st.com/stm32cube-docs/stm32cubemx/6.18.1/en/docs/markup/CubeMX_CLI.html)。详细硬件参数的发现方法见[配置操作基础](../../stm32f1-configure/references/operations.md)。
