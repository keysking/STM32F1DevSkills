# 内部 Flash 与持久化数据

仅在需求涉及参数存储、升级或 bootloader 时采用。先确定完整型号、内存布局、现有数据和掉电行为；不得为演示写入任意 Flash 地址。

## 布局与保留区

STM32F103C8T6 额定 64 KB Flash、20 KB SRAM。即使器件读写高地址似乎成功，也不能把额定容量外空间当成可发布设计依据。该密度主 Flash 页大小为 1 KB；其他 F1 按具体密度核对页大小。

应用代码、引导程序和数据区必须在链接脚本中明确分区。单纯在 C 里定义“参数地址”不会阻止链接器把代码放在那里；检查 map/ELF 段与擦除页范围。烧录工具的擦除行为也要与保留数据要求一致。

## 擦除与编程

先读取现有数据并确认可修改范围。按当前 HAL 的 `HAL_FLASH_Unlock()`、`HAL_FLASHEx_Erase()`、`HAL_FLASH_Program()`、`HAL_FLASH_Lock()` 组织操作，每步检查状态和错误码，擦除结构按页地址/页数设定。

F1 编程有半字对齐和擦除态要求；较宽的 HAL 编程选项可能拆成多个半字操作，不具备整体掉电原子性。按实际电压、时序、写入寿命设计，不能把擦写 API 成功当成事务提交。读回比较或 CRC 仅验证当前数据，不替代掉电恢复协议。

需要可靠参数存储时，设计版本、长度、校验、提交标记、双页/日志和恢复顺序；先验证擦除或写入中断后的选择逻辑。看门狗和实时任务要计入擦写阻塞，不能无条件关闭中断覆盖整个存储流程。

## Bootloader 与启动

只有已有引导布局或明确升级需求才调整应用起始地址。链接 FLASH ORIGIN、向量表、VTOR 初始化和下载地址要一致。当前系统文件可能用 `USER_VECT_TAB_ADDRESS` 等宏控制 VTOR；只改链接脚本不会自动改运行时向量位置。

应用跳转前要满足实际引导协议：有效初始 MSP、复位入口及 Thumb 位，关闭/移交中断、SysTick、DMA 和外设，设置向量表和栈后进入应用。不把跳转函数当作复位等价物。

ROM bootloader 能力由准确型号和 [AN2606](https://www.st.com/resource/en/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)决定。F103C8 的系统 bootloader 不能因芯片有 USB 就推断支持 USB DFU；自定义 USB bootloader 则需要已经烧入且布局匹配。
