# 基于 STM32F103C8T6 和 SSD1306 OLED 的简易示波器

## 概述

利用 STM32F103C8T6 内置 ADC 对模拟信号进行采样，通过 0.96 寸 128×64 I2C OLED 屏幕实时绘制波形。按键切换采样率，板载 PWM 输出可作为自测信号源。

![实物图](2.jpg)

## 功能

- **信号采样** — PA0 (ADC1_IN0) 单通道连续采样，12 位分辨率，软件触发
- **波形显示** — 128 个采样点连线绘制，叠加十字准线
- **采样率切换** — 按键 (PA8) 循环切换 4 档：**100 Hz / 500 Hz / 1 kHz / 2 kHz**（30 ms 去抖）
- **自测信号** — PA15 (TIM2_CH1) 输出约 100 Hz、50% 占空比的 PWM，可供探头测试
- **板载 LED** — PC13 (低电平点亮)

## 硬件接线

| STM32F103C8T6 | 外设 |
|:---|:---|
| PA0 (ADC1_IN0) | 模拟信号输入 |
| PB6 (I2C1_SCL) | OLED SCL |
| PB7 (I2C1_SDA) | OLED SDA |
| PA8 (EXTI) | 按键（下降沿触发，内部上拉） |
| PA15 (TIM2_CH1) | PWM 测试信号输出 |
| PC13 | 板载 LED（开漏输出） |

**OLED 供电：** 3.3 V — VCC，GND — GND

> **注意：** OLED I2C 地址为 0x3C，通信速率 400 kHz。

## 采样率参数

| 档位 | 频率 | TIM1 Prescaler | TIM1 Period |
|:---|:---|:---|:---|
| 0 | 100 Hz | 7199 | 99 |
| 1 | 500 Hz | 7199 | 19 |
| 2 (默认) | 1 kHz | 7199 | 9 |
| 3 | 2 kHz | 3599 | 9 |

系统时钟 72 MHz (HSE 8 MHz × 9 PLL)，TIM1 挂载在 APB2 (72 MHz)。

## 软件流程

1. 系统初始化（HAL、时钟 72 MHz、GPIO、ADC1、I2C1、TIM1、TIM2）
2. OLED 初始化，启动 ADC 连续转换，启动 TIM1 采样中断，启动 TIM2 PWM 输出
3. 主循环等待 `isDraw` 标志：
   - 使用 `ssd1306_Line` 将 128 个 ADC 样本连线绘制
   - 叠加水平线 (y=31) 和垂直线 (x=63) 十字准线
   - 刷新屏幕后清空缓冲
4. `HAL_TIM_PeriodElapsedCallback` — 每次 TIM1 溢出读取 ADC 数据寄存器，缩放至 0–63 像素范围
5. `HAL_GPIO_EXTI_Callback` — 按键检测，循环切换采样率（动态更新 TIM1 PSC/ARR）

![波形效果](1.jpg)

## 构建

项目使用 **CMake + Ninja + arm-none-eabi-gcc** 构建。

```bash
# Release 构建
cmake --preset Release
cmake --build build/Release

# Debug 构建
cmake --preset Debug
cmake --build build/Debug
```

生成 `.elf` 文件可用 ST-Link 等工具烧录。

## 依赖

- **STM32CubeMX** — 用于引脚/外设配置 (`.ioc` 文件)
- **ARM GCC 工具链** (`arm-none-eabi-gcc`)
- **SSD1306 库** — 位于 `../Library/SSD1306/`，提供 I2C 驱动及字体

## 项目结构

```
.
├── Core/
│   ├── Inc/                      # 头文件 (main.h, stm32f1xx_hal_conf.h 等)
│   └── Src/                      # 源文件 (main.c, stm32f1xx_it.c, stm32f1xx_hal_msp.c 等)
├── cmake/
│   ├── gcc-arm-none-eabi.cmake   # 工具链文件
│   └── stm32cubemx/              # CubeMX 生成的构建逻辑
├── CMakeLists.txt                # 顶层 CMake
├── CMakePresets.json             # 构建预设
├── STM32F103xx_FLASH.ld          # 链接器脚本
├── startup_stm32f103xb.s         # 启动文件
└── .03_Oscilloscope_OLED.ioc     # CubeMX 项目文件
```

## 联系

QQ：3619282776
