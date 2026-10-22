# 下载、校验与启动

烧录需要用户实际下载意图和明确目标。只询问环境或制作技能包时不连接、复位或写入板卡。命令中的序列号、固件和地址必须从当前任务取得。

## 发现工具与目标

先读 `cube --list`、`cube programmer --help`；如果没有该入口，定位原生 STM32_Programmer_CLI，见[环境](../../stm32f1-env/references/tools.md)。列举 ST-Link 的形式：

```text
cube programmer -l st-link-only
```

多个探针时用序列号绑定目标，不凭枚举顺序。确认目标供电、共地、SWDIO/SWCLK 和必要 NRST 接线。SWD 连接可能停止或改变目标状态；先判断现场状态是否要留给调试取证。

已允许连接、需要核实目标时，按实际序列号和适当频率使用：

```text
cube programmer -c port=SWD sn=<实际序列号> freq=4000 mode=NORMAL
```

4000 是 kHz 参数示意，接线/板卡不可靠时降低。命令含 `<...>` 的位置必须替换，不能原样粘贴到 shell。连接输出的 device ID 可能覆盖多个容量型号；结合完整丝印、Flash 容量信息和项目 CPN，不用通用名称覆盖用户提供的 C8T6。

## 选择固件和地址

- ELF：优先使用本次成功构建的 ELF，核对 loadable segments 与链接脚本。
- HEX：带地址；确认地址属于当前应用/引导布局。
- BIN：无地址；下载地址来自链接布局和生成方式，不能仅因是 F1 就固定为 0x08000000。

只在“从内部 Flash 起始处运行、无 bootloader 偏移”的工程中，0x08000000 才通常是应用下载地址。保留参数区或引导区时先核对固件覆盖和擦除范围。

## 下载与校验

以下是互相替代的格式选择。ELF 路径按实际替换，命令不附带默认复位：

```text
cube programmer -c port=SWD sn=<实际序列号> freq=4000 mode=NORMAL -w "实际固件.elf" -v
```

需要 BIN 且已确定地址时：

```text
cube programmer -c port=SWD sn=<实际序列号> freq=4000 mode=NORMAL -w "实际固件.bin" <实际起始地址> -v
```

检查返回码及下载、校验日志，不能只看到工具启动成功。地址/校验错误先停止，检查固件、目标、保护与连接，不自动整片擦除或改选项字节。

## 启动与完成状态

下载校验通过且任务要求运行时，按当前工具帮助使用 `-rst` 或已确认的复位运行方式；检查 BOOT 引脚、复位接线、调试停机状态。复位命令形式：

```text
cube programmer -c port=SWD sn=<实际序列号> freq=4000 mode=NORMAL -rst
```

按实际任务验证运行指标，例如协议响应、波形、任务计数或主机枚举。下载校验仅说明存储内容匹配，不证明时钟、硬件和业务正确。报告固件/目标、下载校验结果、复位动作及实际功能证据，故障见[恢复](recovery.md)。

## 选择系统串口 bootloader 时

只有任务选择该下载路径、准确型号支持且允许改变启动状态时使用。先按 AN2606 核对系统存储器启动条件、支持的 USART 和引脚。F103C8 常用 USART1 PA9/PA10，连接 USB-UART 的 RX/TX 并共地，电平须符合芯片要求；不能接 RS-232 电平。

按板卡电路设置 BOOT 并复位进入系统 bootloader。用 `cube programmer -l uart` 取得实际端口，再根据当前帮助连接。若实际端口为 COM6 且 115200/偶校验符合当前路径，Windows 形式为：

```text
cube programmer -c port=COM6 br=115200 P=EVEN
```

COM6 不是默认端口；Linux/macOS 用枚举返回的端口名称或路径。识别目标后，按前述固件格式追加 `-w`/`-v`，不得携带 SWD 专用 sn/freq/mode 参数。串口已被终端占用、启动引脚错误、TX/RX 或电平错误时先处理对应原因。

下载后按应用要求恢复 BOOT 状态并复位，或使用当前帮助与启动布局允许的跳转方式。不能把一次写入成功理解成下一次上电必然进入用户 Flash。
