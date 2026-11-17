# 启动传统 CubeMX 与执行脚本

本页说明工具入口；查询与修改脚本分开准备，只执行任务所需命令。F1 路线是传统 CubeMX，不是 CubeMX2；不能把 `cube mx` 的工程/参数命令套到这里。

## 无人值守弹窗与恢复

每个由 Agent 执行的 `-q` 脚本都以 `popupwrapper set` 开头、以 `exit` 结束，并设置进程超时、保留标准输出和错误日志。即使环境和入口已确认，每次独立会话仍需启用弹窗处理。

- 对新发现的 CubeMX 版本，先用仅含 `popupwrapper set`、`get version`、`exit` 的会话探查，检查日志中 `popupwrapper set` 后的 `OK`。不支持时优先寻找匹配项目且支持该命令的安装。只能使用当前版本时，将操作分段并限制每段的等待时间，超时后结束本次会话并继续排查。
- 对标题、含义、按钮映射都已核实的弹窗，可在 `popupwrapper set` 后、触发命令前加入 `popupwrapper record_answer "<弹窗标题>" <回答>`，例如关闭完成通知或取消打开 IDE。以当前版本的会话帮助和实际结果确认回答，避免仅凭 `YES` 或按钮顺序推断效果。在英文 6.18.1 中，`popupwrapper record_answer "New STM32Cube firmware version available" CANCEL` 可在无需迁移且有匹配安装可重试时自动取消当前加载。
- 对未登记的弹窗，CubeMX 6.18.1 会返回选项值 `0`，通常是第一个选项，日志写出 `default Answer was returned`。让会话继续，随后核对弹窗、受影响的操作及产物。不能预判其影响的已有工程操作先在可恢复的工作副本中执行，核对 IOC、目标固件版本、生成文件和用户代码差异再应用结果。生成流程使用现有工程要求的版本和本地依赖，登录、许可证和有意迁移走各自的流程。
- 遇到失败先从日志定位触发命令，尝试复用匹配的 CubeMX 和固件包、修正工程路径或配置，再从实际保存状态开始重试。若默认回答带来非预期结果，按已查明的弹窗指定回答或消除触发条件，并验证重试产物。并行推进可以独立完成的工作。只有关键步骤仍然受阻且需要用户决定时，才集中报告已完成的工作、尝试过的恢复和剩余影响。

使用实际 Java 或会等待 CubeMX 结束的安装入口。启动前或未被 CubeMX 拦截的弹窗通过进程超时处理，只结束本次会话。确认会话执行到了 `exit`、关键命令没有 `KO`，并检查结果和产物。单独探查命令组用法时，`Usage` 后的 `KO` 可作为发现结果。

## Windows

先按[工具定位](tools.md)找到安装目录。部分传统 CubeMX Windows 发行版的 `STM32CubeMX.exe` 本身可作为 Java 归档运行，这是该工具特有的布局，不是所有 exe 都能如此启动。

核对安装自带 `jre/bin/java.exe`、`STM32CubeMX.l4j.ini` 等启动配置，读取其 JVM 参数。不要依赖 PATH 中碰巧存在的 Java。以下是 Java 21 配套发行版的启动形式；`$mxRoot`、`$script`、`$log` 均由当前任务提供的真实绝对路径赋值：

```powershell
$mxRoot = 'D:/tools/STM32CubeMX' # 替换为已发现的安装目录
$script = (Resolve-Path -LiteralPath 'D:/work/query.txt').Path # 替换为本次脚本
$log = 'D:/work/query.log' # 替换为本次日志；父目录须存在
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
$arguments = $jvm + @('-jar', $jar, '-q', $script)
$argLine = ($arguments | ForEach-Object { '"{0}"' -f $_ }) -join ' '
$process = Start-Process -FilePath $java -ArgumentList $argLine -WorkingDirectory $mxRoot `
    -RedirectStandardOutput $log -RedirectStandardError "$log.stderr" -WindowStyle Hidden -PassThru -ErrorAction Stop
$null = $process.Handle # 保留进程句柄，Windows PowerShell 5.1 才能稳定读取 ExitCode
if (!$process.WaitForExit(180000)) {
    Stop-Process -InputObject $process -ErrorAction SilentlyContinue
    throw "CubeMX 超时，检查 $log 和 $log.stderr"
}
if ($process.ExitCode -ne 0) {
    throw "CubeMX 退出码 $($process.ExitCode)，检查 $log 和 $log.stderr"
}
```

JVM 选项在 `-jar` 之前，CubeMX 参数在归档路径之后。若安装自带启动配置不同，按其配置调整；发生模块访问或 Java 版本错误时先对照运行时，不盲目换系统 Java。

这些路径直接由发现结果赋值。按操作规模调整示例中的 180 秒，按实际 CubeMX 输出编码读取原始日志。

## Linux / macOS

使用实际安装中的启动器，先切到它要求的工作目录。Linux 常见形式是安装目录下 `./STM32CubeMX`；macOS 使用应用包 `Contents` 内的实际启动器，先查看路径，注意 `MacOS`/`MacOs` 等名称的大小写。若发行版要求 Java 启动，使用其自带运行时及配置。

以下 Bash 中变量取实际路径；`script`、`log` 为绝对路径，`launcher` 是相对于该工作目录的真实启动器：

```bash
mx_dir='/opt/STM32CubeMX' # 替换为实际启动器工作目录
launcher='./STM32CubeMX' # 替换为实际启动器
script='/work/query.txt' # 替换为本次脚本绝对路径
log='/work/query.log' # 替换为本次日志绝对路径，父目录须存在
cd "$mx_dir" || exit 1
if command -v timeout >/dev/null 2>&1; then
    deadline=timeout
elif command -v gtimeout >/dev/null 2>&1; then
    deadline=gtimeout
else
    printf '%s\n' '用执行工具管理本次 CubeMX 进程的超时，然后再启动。' >&2
    exit 1
fi
"$deadline" -k 5s 180s "$launcher" -q "$script" >"$log" 2>&1
```

`-i` 进入交互模式，`-q` 执行安静脚本，`-s` 是普通脚本模式。无图形会话时能否启动取决于当前发行版的运行依赖。

## 脚本上下文与结果

UTF-8 文本，每行一条 CubeMX 会话命令，以 `exit` 结束。会话命令不是 PowerShell/Bash 命令。引用路径与带空格的模式名；不要把 `# 注释`、shell 变量声明或 Markdown 围栏混进脚本，除非已确认当前解析器支持。

已有工程查询的脚本形式（替换 IOC 路径）：

```text
popupwrapper set
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
- 避免同一 IOC 同时被 GUI 和脚本编辑。涉及驱动切换或多项目时优先独立进程加载目标，减少上下文残留影响。

命令入口依据为 [CubeMX CLI](https://dev.st.com/stm32cube-docs/stm32cubemx/6.18.1/en/docs/markup/CubeMX_CLI.html)和 [ST 的弹窗处理说明](https://community.st.com/stm32cubemx-mcus-29/partially-disabled-conflict-is-hindering-cicd-pipeline-153663)。详细硬件参数的发现方法见[配置操作基础](../../stm32f1-configure/references/operations.md)。
