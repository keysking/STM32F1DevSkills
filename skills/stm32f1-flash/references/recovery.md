# 连接与下载故障

先保留日志、固件和已有配置。按证据逐层处理，不把 mass erase、解除保护或改 boot 设置作为默认连接步骤。

| 现象 | 检查顺序 |
| --- | --- |
| 无探针 | USB 数据线、主机识别、ST-Link 驱动/权限、工具占用；Linux udev |
| 探针有、目标无 | 目标电压、共地、SWDIO/SWCLK、NRST、频率、板卡焊接 |
| 运行后失联 | 固件关闭 SWD、错误重映射、低功耗、快速复位、看门狗；保留现场或复位重现二选一 |
| 多工具争抢 | 关闭本任务明确识别的旧 Programmer/GDB 会话；不结束所有 Java/调试进程 |
| 地址/校验失败 | ELF/BIN 地址、真实容量、擦除页、供电、保护、芯片识别和旧产物 |

## 特殊连接模式

`mode=HOTPLUG` 用于尝试附加运行目标；它不是“完全没有扰动”的保证。需要保存故障现场时先采用合适的附加调试方案，避免 reset。

只有允许复位且 NRST 实际连接时才尝试 `mode=UR`（under reset），必要时降低 SWD 频率。它用于在故障应用夺走 SWD 前连接；不能解决缺电源、断线或错误目标。

F1 AFIO 可关闭 JTAG/SWD。PA13/PA14 被分配为普通 GPIO 后应回查 SYS Serial Wire 及生成 remap；常用保留 SWD、释放 JTAG 的配置与完全禁用调试不同。

## 保护、擦除与恢复路径

先读取实际保护状态并解释后果。解除读保护可能擦除主 Flash，写保护修改会改变持久安全/更新行为；需要用户明确接受具体数据丢失和目标范围后才执行，不把通用烧录授权扩展成任意保护变更。

需要擦除时选择经确认的页/区域，遵守应用和参数区布局。设备身份或容量有疑问时停在诊断，不能通过越界写入“测容量”。

ROM bootloader 按准确型号核对 [AN2606](https://www.st.com/resource/en/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)、BOOT 引脚和支持接口。F103C8 常用系统 bootloader 路径是 USART1；没有预装自定义 USB bootloader 时，不能因存在 USB 外设就使用 DFU 恢复。

恢复后修正导致失联的 IOC/固件并重新构建，检查正常上电能否启动与再次连接；不要只把一次 under-reset 成功视作故障已解决。
