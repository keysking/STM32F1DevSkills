# GDB 与 ST-Link 会话

调试先确定当前 ELF 与板上固件是否一致，以及要附加取证还是从复位重现。附加操作本身也可能暂停程序；需要持续运行的系统先选择合适观测方式。

## 定位与启动服务器

通过 `cube --list` 查实际 GDB server 入口，再读取其 `--help`；直接工具常名为 ST-LINK_gdbserver。定位对应 arm-none-eabi-gdb 和 STM32CubeProgrammer 安装，见[工具定位](../../stm32f1-env/references/tools.md)。不把编译器包或 Programmer 可执行文件路径误作服务器 `-cp` 所需的安装目录。

先用实际服务器路径执行 `--help` 和 `-q`，确认版本及探针。选择附加路径时，当前 ST-Link server 的参数形态为：

```text
ST-LINK_gdbserver -d -p 61234 -i <实际探针序列号> -cp "实际Programmer安装目录" -g --frequency 1000
```

`-d` 为 SWD，`-p` 为本地端口，`-i` 选择探针，`-g` 附加，频率单位 kHz。实际二进制用已发现的绝对路径执行；PowerShell 含空格路径使用 `&`，不能复制占位符直接运行。

如果任务允许从复位重现且 NRST 可用，改用当前帮助支持的 `-k` 初始化复位路径，按需要选择 halt。不要在保留故障现场的附加会话中顺手加 reset、`--erase-all` 或 load。启动后台进程时 Windows 使用隐藏窗口并记录 PID、日志，只结束本次启动的进程。

## GDB 连接与基本取证

原生调用 `arm-none-eabi-gdb "实际固件.elf"`，在 GDB 内：

```text
target extended-remote localhost:61234
info registers
bt
info breakpoints
```

端口必须与服务器一致。若目标正在运行，需要取静止快照时中断执行，再记录停止位置和状态。`file` 只加载本地符号，`load` 会写目标，两者不能混淆。服务器特有命令先 `monitor help`，不猜测 reset 的准确形式。

只有明确需要下载并复位调试时，执行已确认的 `load`、monitor reset/halt 流程，在 main 设置临时断点后继续。断点数量受硬件限制，启动/Flash 代码可按需 `hbreak`；Debug 与优化版本的变量可见性、单步顺序不同。

## 寄存器与内存

按当前 GDB 目标描述查询 MSP、PSP、LR、PC、xPSR、CONTROL、PRIMASK、BASEPRI 等；不支持某寄存器名时查 `info all-registers`。局部变量使用 `p`，内存使用 `x`，例如查看已确认的 SRAM 地址 `x/16wx <地址>`。

只读必要寄存器。某些状态位读后清除，读取外设数据寄存器可能消费数据，不能连续扫描整个外设地址空间。对活跃 DMA 缓冲区，多次读取不保证同一时刻一致；按应用设计快照或暂停生产者，并说明扰动。

保存证据时记录固件路径/哈希、服务器版本、目标、停止原因、关键寄存器和反汇编。退出时根据任务选择 detach/继续运行或保持停止，结束本次服务器；不要在用户要求保留现场时自动复位。

故障路径见[启动与异常](startup-faults.md)、[外设](peripherals.md)、[FreeRTOS](freertos.md)。服务器选项以实际安装帮助为准。
