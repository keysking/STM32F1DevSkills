# CMake 构建、应用目标与产物

仅适用于 CMake 工程。先读取根 CMakeLists.txt、CMakePresets.json、CMakeUserPresets.json、工具链文件和生成子目录；下列目录/目标名按实际工程替换，不同时套用全部片段。

## 工具链与配置

先定位 GCC、Ninja、CMake，见[工具定位](../../stm32f1-env/references/tools.md)。工具不在 PATH 时，PowerShell 在当前进程中设置：

```powershell
$gccBin = 'D:/toolchains/arm-gcc/bin' # 替换为已定位的真实目录
$ninjaDir = 'D:/tools/ninja' # 替换为已定位的真实目录
if (-not (Test-Path -LiteralPath (Join-Path $gccBin 'arm-none-eabi-gcc.exe'))) { throw 'GCC 路径无效' }
if (-not (Test-Path -LiteralPath (Join-Path $ninjaDir 'ninja.exe'))) { throw 'Ninja 路径无效' }
$env:PATH = "$gccBin;$ninjaDir;$env:PATH"
cube cmake --list-presets
```

Linux/macOS Bash 使用实际已解析的目录，设置仅影响当前终端及子进程：

```bash
gcc_bin='/opt/arm-gcc/bin' # 替换为已定位的真实目录
ninja_dir='/opt/ninja' # 替换为已定位的真实目录
test -x "$gcc_bin/arm-none-eabi-gcc" && test -x "$ninja_dir/ninja" || exit 1
export PATH="$gcc_bin:$ninja_dir:$PATH"
cube cmake --list-presets
```

若当前工具入口不是 `cube cmake`，用已定位的原生 cmake。先查看预设是否提供 Debug，再选择执行：

```text
cube cmake --preset Debug
cube cmake --build --preset Debug
```

只有 configure preset、没有 build preset 时，构建其实际 binaryDir，例如 `cube cmake --build build/Debug`。不存在预设则读取工具链，并用独立目录配置：

```text
cube cmake -S . -B build/Debug -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cube cmake --build build/Debug
```

以上工具链路径须确实存在。切换编译器、生成器或芯片时使用新的构建目录，不复用不匹配缓存。Debug/Release 使用各自目录，实际优化/调试参数以 compile_commands.json 为准；名字为 Debug 不保证所有源都包含调试信息。

## 添加应用源码

先确认目标名；CubeMX 根文件常用 `${CMAKE_PROJECT_NAME}`，但已有项目不一定如此。少量应用文件可加到根目标：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE App/app.c)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE App)
```

只在实际需要该模块且文件存在时加入。文件使用 STM32 头文件时，确认目标已经链接生成器提供的 `stm32cubemx` 接口。需要独立模块时，在用户维护的 cmake 文件中建立目标，例如：

```cmake
add_library(app OBJECT App/app.c)
target_include_directories(app PUBLIC App)
target_link_libraries(app PRIVATE stm32cubemx)
target_link_libraries(${CMAKE_PROJECT_NAME} app)
```

这是替代组织形式，不与前一段重复注册同一源。CubeMX 生成的 executable 通常已使用无关键字的 `target_link_libraries`，上段最后一行与之保持一致；若实际工程该目标已使用 PRIVATE/PUBLIC/INTERFACE，则最后一行改用 `target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE app)`。同一目标不能混用两种签名；新建 `app` 目标的 PRIVATE 不受 executable 的签名影响。

`stm32cubemx` 名称从实际生成文件确认，不能在不含该目标的手写工程中照抄。生成器管理的 `cmake/stm32cubemx/CMakeLists.txt` 可能被覆盖，应用扩展放根 CMake 或自己的 include 文件。

编译定义作用于目标：只对主 executable 加 PRIVATE 宏，不会自动影响独立 HAL/应用库。共享定义可放用户维护的 INTERFACE 目标并由需要的目标链接；检查 compile_commands 中每个相关源实际命令，避免只看到 main.c 带宏就认为全部生效。

## 链接选项与 BIN/HEX

只有需要浮点 printf 时，且确认使用 GNU 链接流程，给实际固件目标添加：

```cmake
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE -u _printf_float)
```

输出仍需实际重定向，见[应用移植](../../stm32f1-firmware/references/porting.md)。不要为没有需求的工程启用以增加体积。

需要 BIN/HEX 时，确认 CMAKE_OBJCOPY 已由工具链设置，再添加构建后转换：

```cmake
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}>
          $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>/${CMAKE_PROJECT_NAME}.bin
  COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}>
          $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>/${CMAKE_PROJECT_NAME}.hex
  VERBATIM)
```

目标、文件名按工程替换；确认产物确实来自本次 ELF。BIN 不带地址；HEX/ELF 带地址信息。

## 产物检查

对本次 ELF 执行原生工具，路径用真实输出替换：

```text
arm-none-eabi-size firmware.elf
arm-none-eabi-objdump -h firmware.elf
arm-none-eabi-readelf -h -l firmware.elf
arm-none-eabi-nm --size-sort --print-size firmware.elf
```

结合 map、链接 MEMORY 区及 load address 检查向量、代码、初始化数据、零初始化数据和预留区。常规布局 Flash 近似 text+data，静态 RAM 为 data+bss，再考虑栈/堆/其他段；复杂布局按段和区域逐项核算。调试符号使 ELF 文件较大，不占对应 Flash。

C8T6 链接区应与 64 KB Flash、20 KB RAM 及实际保留区相符；`startup_stm32f103xb.s`/`STM32F103xB` 不能推出 128 KB。还需确认内核为 Cortex-M3、Thumb、软浮点，以及正确 startup/system/linker 文件。构建不隐含下载或功能验证。

依据：[CMake 目标](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html)、[预设](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)。
