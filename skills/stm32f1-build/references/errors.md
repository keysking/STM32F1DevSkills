# 编译与链接故障定位

保留第一次失败的完整命令和日志，从首个有效错误开始。必要时 `cmake --build <实际构建目录> --verbose`；不要同时升级 SDK、重建工程并修改代码，导致原因无法区分。

| 现象 | 优先检查 | 修复边界 |
| --- | --- | --- |
| Ninja/GCC 找不到 | 当前进程 PATH、工具链路径、CMakeCache | 定位工具或新建匹配缓存，不随意改系统 PATH |
| 头文件缺失 | 包是否完整、目标 include、大小写 | 修复实际依赖，不把全盘目录加入 include |
| HAL 类型/函数不存在 | F1/F4/C5 混用、HAL 版本、模块宏 | 按本项目声明适配，不凭名称补空函数 |
| undefined reference | 源未编入、静态库依赖、C++ linkage、弱符号覆盖 | 检查目标源列表、nm/map 和链接顺序 |
| multiple definition | 重复 main/startup/system/syscalls/sysmem、双 timebase、双 heap | 只保留实际所有者；不默认加 allow-multiple-definition |
| hard-float/架构不兼容 | 第三方预编译库、CPU/ABI 参数 | F1 Cortex-M3 软浮点，换匹配库或重编 |
| FLASH/RAM overflow | 芯片容量、库功能、缓冲/RTOS 堆栈 | 先确认真实布局，再优化；不虚增 C8 容量 |
| 宏只部分生效 | 目标 PRIVATE/INTERFACE 范围 | 查每个源 compile_commands，修复依赖传播 |

## 生成器造成的重复源

创建并立即在同一会话生成时，部分版本可能把 syscalls/sysmem 等路径重复写入生成 CMake。按[创建流程](../../stm32f1-project/references/create.md)先保存，再独立加载 IOC 并生成；重新检查源列表。不要把重复源问题归因于 GCC 版本并批量升级。

## 资源与运行时内存

`_Min_Heap_Size`/`_Min_Stack_Size` 常是链接布局中的最低预留，不等同所有运行时分配的硬上限。检查实际 `_sbrk` 的 `_end`、`_estack` 和边界逻辑。FreeRTOS heap_4 自有 ucHeap，任务栈来自相应分配路径，MSP 仍承担异常栈；不同内存池分别评估。

浮点格式化、USB 缓冲、队列、静态数组常影响 C8 预算。先用 map/nm 找实际大对象和符号，再减少需求不需要的功能；不要盲目减栈直到链接通过。

## 成功判断

重跑受影响的配置/构建，确认 ELF 时间和路径、目标芯片、必要符号、段布局与资源预算。若只修 CMake，不因链接通过就声称外设已工作。运行错误移交[启动与异常](../../stm32f1-debug/references/startup-faults.md)。
