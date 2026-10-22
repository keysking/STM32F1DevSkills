# HAL 与 LL 的使用边界

F1 使用传统 CubeF1 HAL/LL。先读项目头文件和 CMake 源列表，不能套用 C5 HAL2 的句柄、初始化函数或 GPIO 参数。

## 保留现有驱动

HAL 工程常用 `HAL_Init()`、`SystemClock_Config()`、`MX_*_Init()`、`HAL_*`；LL 为 `LL_*` 和对应头文件。旧 SPL 的 `GPIO_InitTypeDef` 等重名类型不表示可与 HAL 混用，先查包含来源。用户只要求应用功能时，不自动迁移驱动层。

HAL GPIO 调用形态为 `HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET)`；这里是 STM32F1 接口，实际端口、电平按需求替换。UART、ADC、TIM 等接口通常接收实际 `*_HandleTypeDef` 的指针，先找生成的声明和初始化。

## 确有需求时切换生成驱动

只有用户要求或明确的功能/资源约束需要时才切换 LL。加载并保存实际 IOC 后，在独立重载会话使用生成器驱动类别：

```text
setDriver USART LL
```

这是 **USART 类别** 的选择，不能写成 `setDriver USART1 LL` 并假设只改变 USART1。当前生成器可能同时影响 USART1、USART2 等同类实例；回读 IOC 的 driver 设置并检查所有相关生成文件。GPIO 类别的形式是 `setDriver GPIO LL`，仅在 GPIO 确需切换时使用。

再次保存、重载和生成，核对 CMake 是否包含所需 LL 源、头文件和定义；只修改一次参数但没有落实到 IOC 不算完成。已有 HAL 应用需相应适配，尤其 IRQ、DMA 和错误处理。API 参数不通过名称替换机械迁移。

## 初始化、启动和回调

区分外设 Init 与业务 Start。TIM、ADC、UART DMA、CAN 一般还需相应启动；IWDG/WWDG 的 Init 可直接启动。按具体驱动源码确定，不能采用“所有 Init 都不启动”的通则。

HAL 回调由实际 IRQ/状态机触发。弱回调覆盖必须使用准确签名、非 static，C++ 定义使用 `extern "C"`；多个实例共享回调时按 Instance/句柄分派。若项目启用回调注册机制，按该版本注册规则维护，不同时建立两套所有权。

LL 直接操作寄存器时，应用负责时钟、标志、错误、超时和状态一致性。可以在清晰边界混用 HAL/LL，但不能由 LL 擅改 HAL 正在使用的 DMA 或外设状态，再期待 HAL 句柄自动同步。

已经确定采用 LL，需要具体 GPIO、USART、IRQ/DMA 调用时，读[LL 操作](ll-operations.md)；只实现现有 HAL 功能时不加载或执行驱动切换章节。

## 查证接口

按任务搜索项目自身驱动目录，例如：

```text
rg -n "HAL_ADCEx_Calibration_Start|HAL_ADC_Start_DMA" Drivers
rg -n "HAL_UARTEx_ReceiveToIdle_DMA|HAL_UARTEx_GetRxEventType" Drivers
rg -n "LL_GPIO_SetOutputPin|USE_FULL_LL_DRIVER" Drivers CMakeLists.txt cmake
```

缺失 API 时先核对版本与功能等价路径，不复制其他系列签名。在线摘要可能误述 F1；例如 F1 `HAL_ADCEx_Calibration_Start()` 在实际 HAL 中执行复位校准和校准，并非空操作。

官方源：[HAL/LL 驱动](https://github.com/STMicroelectronics/stm32f1xx-hal-driver)、[CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)。以工程锁定版本为准。
