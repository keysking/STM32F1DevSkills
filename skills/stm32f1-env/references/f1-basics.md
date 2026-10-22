# F1 型号、驱动与工程基础

按实际料号查能力；本文中的 C8T6 只说明具体规则，不能作为未知板卡的选择。资料入口：[STM32F1](https://www.st.com/en/microcontrollers-microprocessors/stm32f1-series.html)、[STM32CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)。

## 系列内的差异

| 产品线 | 最高主频 | 判断要点 |
| --- | --- | --- |
| F100 | 24 MHz | Value line，参考 RM0041；不能套用 F103 时钟树和外设集合。 |
| F101 | 36 MHz | Access line；按密度核对外设。 |
| F102 | 48 MHz | USB access line；不能默认有 CAN。 |
| F103 | 72 MHz | Performance line；容量档次影响 TIM、DMA、FSMC、SDIO 等能力。 |
| F105/F107 | 72 MHz | Connectivity line，PLL、USB、DMA、CAN 和 Ethernet 等按具体型号判断。 |

均为 Cortex-M3，没有 FPU。F1 GPIO 使用 CRL/CRH、AFIO 与重映射机制，不是通用的 GPIO AFR/DMAMUX 配置。HAL 是传统句柄式接口，与 HAL2 不同。

## STM32F103C8T6 的具体约束

- C8T6：LQFP48、额定 Flash 64 KB、SRAM 20 KB；同分组 CB 是 128 KB。不要利用非规格容量或凭调试器显示扩大链接区。
- 预处理宏 `STM32F103xB`、启动文件 `startup_stm32f103xb.s` 同时服务 x8/xB，不意味着 C8 变成 CB。最终容量来自准确料号和链接脚本。
- TIM1 为高级定时器，TIM2–TIM4 为 16 位通用定时器；没有 TIM5。不能因通用 HAL 文件出现某 API 就认定该器件支持。
- ADC 固定 12 位；独立 ADC DMA 路径重点检查 ADC1，不能把 ADC2 视为同样具有独立 DMA 请求。
- bxCAN 与 USB 共用专用 SRAM，不能并发使用；重映射 CAN 引脚不能解除该硬件限制。
- PC13–PC15 有额外驱动能力和速率限制，按数据手册电气条件设计；不能默认 PC13 接 LED 或默认其有效极性。
- 仅知道料号为 C8T6，仍不能确定晶振值、LED、BOOT 接法和 bootloader；这些从实际原理图或已有工程取得。

依据：[C8/xB 数据手册 DS5319](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)、[RM0008](https://www.st.com/resource/en/reference_manual/cd00171190.pdf)、[器件头文件](https://github.com/STMicroelectronics/cmsis-device-f1/blob/master/Include/stm32f103xb.h)。

## 识别项目路线

| 文件/代码 | 含义与动作 |
| --- | --- |
| `.ioc`、`MX_*_Init`、`USER CODE` | 传统 CubeMX；修改配置源并维护用户区。 |
| `Drivers/STM32F1xx_HAL_Driver`、`HAL_Init` | CubeF1 HAL/LL；实际版本读包与源码。 |
| `stm32f10x.h`、`STM32F10X_MD`、`GPIO_Init` | 可能是 SPL，确认头文件和库；不混入 HAL 类型。 |
| 手写启动、寄存器初始化、无 IOC | 沿用已有 CMake 和初始化所有权。 |
| `FreeRTOSConfig.h`、`cmsis_os.h` / `cmsis_os2.h` | 进一步确认原生 API 或 CMSIS v1/v2，避免混用启动链。 |

`CMakeLists.txt`、`CMakePresets.json`、工具链文件和链接脚本比 IDE 名称更能说明构建方式。只有 Keil/IAR/Makefile 的工程不在本包范围，不自动迁移。
