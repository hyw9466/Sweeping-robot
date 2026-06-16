# Sweeping Robot - 基于STM32的蓝牙遥控扫地机器人

## 项目简介

本项目是一个基于 **STM32F103** 的嵌入式扫地机器人固件工程。支持 **蓝牙遥控**（HC-05）和 **超声波自动避障** 两种模式，使用 L298N 驱动双路直流电机实现差速转向控制。

## 硬件平台

| 模块 | 型号/说明 | 接口 |
|------|----------|------|
| 主控 | STM32F103 (Cortex-M3, 72MHz, 高密度) | - |
| 电机驱动 | L298N 双H桥驱动，TIM3 4路PWM | PA6/PA7, PB0/PB1 |
| 超声波测距 | HC-SR04，TIM2 输入捕获 | PA0(ECHO), PA1(TRIG) |
| 蓝牙模块 | HC-05，USART2 + DMA 收发 | PA2(TX), PA3(RX) |
| 调试串口 | USART1，printf 输出 | PA9(TX), PA10(RX) |
| LED | 板载 LED，系统心跳指示 | PC13 |

## 项目架构

```
Sweeping-robot/
├── USER/                          # 应用层
│   ├── main.c                     # 主控逻辑（蓝牙指令解析、自动避障）
│   ├── sys.h                      # GPIO 位带操作宏定义
│   ├── stm32f10x_conf.h           # 外设库头文件配置
│   ├── stm32f10x_it.h/c           # 中断服务函数模板
│   └── system_stm32f10x.c         # 系统时钟配置（72MHz）
│
├── Drivers/                       # 硬件驱动层
│   ├── motor.h/c                  # 电机驱动（TIM3 PWM，前进/后退/左转/右转/停止）
│   ├── ultrasonic.h/c             # 超声波测距（TIM2 捕获，中值滤波）
│   ├── hc05.h/c                   # 蓝牙 HC-05 模块 AT 指令驱动
│   ├── usart.h/c                  # USART1 串口驱动（调试 printf）
│   ├── usart2.h/c                 # USART2 串口驱动（蓝牙通信 + DMA 发送）
│   ├── led.h/c                    # LED 控制（PC13）
│   └── delay.h/c                  # SysTick 延时（us/ms）
│
├── Libraries/                     # 标准库
│   ├── CMSIS/                     # ARM Cortex-M3 内核支持
│   │   ├── CoreSupport/           # core_cm3.c/h
│   │   └── DeviceSupport/ST/STM32F10x/  # 启动文件、设备头文件
│   └── STM32F10x_StdPeriph_Driver/  # STM32 标准外设库
│       ├── inc/                   # 头文件
│       └── src/                   # 源文件
│
└── MDK-ARM/                       # Keil MDK 工程 & 编译产物
    ├── Project.uvprojx            # Keil 工程文件
    ├── Obj/                       # .hex、.axf、.o 等编译输出
    └── List/                      # .map、.lst 链接文件
```

## 功能说明

### 1. 电机控制 ([motor.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/motor.c))

- 使用 **TIM3** 的 4 路 PWM 通道（CH1~CH4）驱动 L298N
- 通过改变 PWM 占空比（0~899）实现调速
- 提供两种接口：固定速度版（`run_init` / `back_init` / `left_init` / `right_init` / `stod_init`）和可调速版（`*_pwm(l, r)`）

### 2. 超声波测距 ([ultrasonic.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/ultrasonic.c))

- TIM2 输入捕获通道1 检测 HC-SR04 回波脉宽
- **中值滤波**：采集 13 个样本取中值，滤除干扰
- 距离计算公式：`distance = TIME * 340 / 20000`（单位 cm）

### 3. 蓝牙通信 ([hc05.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/hc05.c))

- HC-05 通过 AT 指令进行初始化和配置
- USART2 使用 **DMA1_Channel7** 发送，TIM4 定时器实现接收超时判断（10ms）
- 支持 AT 指令查询/设置角色、波特率等

### 4. 主控制逻辑 ([main.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/main.c))

**蓝牙指令集：**

| 指令 | 功能 |
|------|------|
| `F` | 前进 |
| `B` | 后退 |
| `L` | 左转 |
| `R` | 右转 |
| `S` | 停止 |
| `A` | 加速（PWM 占空比 +50，最大 700） |
| `D` | 减速（PWM 占空比 -50，最小 100） |

**自动避障逻辑：**
- 实时读取超声波距离
- 距离 ≤ 5cm 且处于运动状态时：停止 → 后退 → 左转 → 前进
- 距离 5~10cm：低速前进（PWM=200）
- 距离 10~15cm：中速前进（PWM=300）
- 距离 15~20cm：中高速前进（PWM=400）

## 开发环境

- **IDE**：Keil MDK-ARM 5
- **编译器**：ARMCC (MDK 内置)
- **调试器**：J-Link
- **库版本**：STM32F10x Standard Peripheral Library V3.4.0 + CMSIS

## 编译与烧录

1. 使用 Keil MDK-ARM 打开 `MDK-ARM/Project.uvprojx`
2. 编译工程（Build），生成 `MDK-ARM/Obj/Project.hex`
3. 通过 J-Link / ST-Link 烧录到目标板

## 启动文件说明

工程使用 `startup_stm32f10x_hd.s`（高密度设备启动文件），对应 STM32F103 系列 Flash >= 256KB 的型号。其他密度版本的启动文件位于：

```
Libraries/CMSIS/CM3/DeviceSupport/ST/STM32F10x/startup/
├── arm/         # ARM Compiler 启动文件
├── gcc_ride7/   # GCC 启动文件
├── iar/         # IAR 启动文件
└── TrueSTUDIO/  # TrueSTUDIO 启动文件
```