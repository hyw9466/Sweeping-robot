# STM32 扫地机器人 - 学习指南

> 适合有一定 C 语言基础、想入门 STM32 嵌入式开发的初学者。建议按阶段顺序学习，每阶段预计 1~3 天。

---

## 学习路线总览

```
阶段一：环境搭建与跑马灯 ──→ 理解工程结构与 GPIO
    ↓
阶段二：串口调试与蓝牙通信 ──→ 掌握 USART、中断、DMA
    ↓
阶段三：电机与 PWM 控制 ──→ 掌握定时器 PWM 输出
    ↓
阶段四：超声波测距 ──→ 掌握定时器输入捕获
    ↓
阶段五：主控逻辑与避障算法 ──→ 整合所有模块
```

---

## 阶段一：环境搭建与跑马灯

### 1.1 目标

理解工程结构，点亮板载 LED（PC13），实现第一个 "Hello World"。

### 1.2 需要阅读的文件

| 文件 | 说明 |
|------|------|
| [stm32f10x_conf.h](file:///e:/embedded-linux-project/Sweeping-robot/USER/stm32f10x_conf.h) | 外设库的"开关"，决定哪些外设驱动被编译 |
| [sys.h](file:///e:/embedded-linux-project/Sweeping-robot/USER/sys.h) | GPIO 位带操作宏，让你像 51 单片机一样操作 IO |
| [led.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/led.h) | LED 引脚宏定义 `LED = PCout(13)` |
| [led.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/led.c) | LED 初始化函数 |
| [system_stm32f10x.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/system_stm32f10x.c) | 系统时钟配置（72MHz） |

### 1.3 知识点

**GPIO 初始化流程：**
```c
// 1. 开启外设时钟（STM32 每个外设用之前必须先开时钟！）
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

// 2. 配置 GPIO 结构体
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;      // 选择引脚
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; // 推挽输出
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; // 输出速度

// 3. 调用初始化函数
GPIO_Init(GPIOC, &GPIO_InitStructure);
```

**位带操作（sys.h 的精髓）：**
```c
#define BITBAND(addr, bitnum) ((addr & 0xF0000000)+0x2000000+((addr &0xFFFFF)<<5)+(bitnum<<2))
#define MEM_ADDR(addr)  *((volatile unsigned long *)(addr))
#define PCout(n)   BIT_ADDR(GPIOC_ODR_Addr,n)   // 单比特输出
#define PCin(n)    BIT_ADDR(GPIOC_IDR_Addr,n)    // 单比特输入

// 效果：你可以直接写 LED = 1; 来控制 PC13，无需调用库函数
```

> **思考题**：位带操作的原理是什么？地址计算公式中各部分代表什么含义？

### 1.4 动手实验

1. 注释掉 `main.c` 中除 LED 初始化外的所有代码
2. 在主循环中实现 LED 以 500ms 间隔闪烁
3. 尝试改用库函数 `GPIO_SetBits` / `GPIO_ResetBits` 实现同样效果

---

## 阶段二：串口调试与蓝牙通信

### 2.1 目标

掌握 USART 串口通信、NVIC 中断管理、DMA 数据传输。

### 2.2 需要阅读的文件

| 文件 | 说明 |
|------|------|
| [usart.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart.h) | USART1 接口定义、接收缓冲区 |
| [usart.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart.c) | USART1 初始化 + printf 重定向 + 中断接收 |
| [usart2.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart2.h) | USART2 接口定义（含 DMA 发送） |
| [usart2.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart2.c) | USART2 初始化 + DMA 发送 + TIM4 超时判断 |
| [hc05.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/hc05.h) | HC-05 蓝牙模块接口 |
| [hc05.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/hc05.c) | HC-05 AT 指令通信 |
| [delay.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/delay.h) | 延时函数声明 |
| [delay.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/delay.c) | SysTick 延时实现 |

### 2.3 知识点

**USART1——printf 重定向：**

本工程用了两种方式，重点理解 `fputc` 重定向：
```c
// 方式一：不使用 MicroLIB（本工程采用）
_sys_exit(int x) { x = x; }                    // 避免半主机模式
int fputc(int ch, FILE *f) {
    while((USART1->SR & 0x40) == 0);            // 等待发送完成
    USART1->DR = (u8)ch;                         // 发送字符
    return ch;
}

// 方式二：使用 MicroLIB（代码中注释的部分，更简单）
```

**USART2——接收协议设计：**

```c
u16 USART2_RX_STA;  // 接收状态寄存器
// bit15: 接收完成标志（收到完整一帧 = 1）
// bit14: 收到 0x0D（回车符）标志
// bit13~0: 已接收的有效字节数

// 配合 TIM4 定时器实现超时判断：
// 10ms 内没有新字符 → TIM4 溢出中断 → 置位 bit15 → 主循环处理数据
```

**DMA（Direct Memory Access）——解放 CPU 的数据搬运工：**
```c
// USART2 发送使用 DMA1_Channel7
// 发送数据时 CPU 只需告知 DMA 起始地址和长度，DMA 自动搬运，无需 CPU 参与
UART_DMA_Config(DMA1_Channel7, (u32)&USART2->DR, (u32)USART2_TX_BUF);
UART_DMA_Enable(DMA1_Channel7, strlen((const char*)USART2_TX_BUF));
```

**NVIC 中断优先级分组：**
```c
NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
// 分组 2 的含义：2 位抢占优先级 + 2 位子优先级
// 抢占优先级高的可以打断低的，子优先级只决定同时挂起时的执行顺序
```

> **思考题**：为什么 USART2 接收不用 DMA 而是用中断？为什么发送用 DMA？

### 2.4 动手实验

1. 连接 USB 转 TTL 模块到 USART1，用串口助手观察 printf 输出
2. 修改 USART2 的接收超时时间（改 TIM4 的 arr/psc 参数），观察效果
3. 用手机蓝牙连接 HC-05，发送 `F`/`B`/`L`/`R`/`S` 观察串口打印的响应

---

## 阶段三：电机与 PWM 控制

### 3.1 目标

理解 L298N 的工作原理，掌握 STM32 定时器 PWM 输出。

### 3.2 需要阅读的文件

| 文件 | 说明 |
|------|------|
| [motor.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/motor.h) | 电机控制引脚定义、函数声明 |
| [motor.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/motor.c) | TIM3 PWM 初始化和运动控制 |

### 3.3 知识点

**L298N 工作原理：**

L298N 是一个双 H 桥驱动器，每路电机需要 3 根控制线：

| 控制信号 | 作用 |
|----------|------|
| IN1 = PWM, IN2 = GND | 电机正转，PWM 占空比 = 速度 |
| IN1 = GND, IN2 = PWM | 电机反转，PWM 占空比 = 速度 |
| IN1 = GND, IN2 = GND | 电机停止 |

本工程使用 **TIM3 的 4 路 PWM 通道**：
- 左电机：TIM3_CH1 (PA6) + TIM3_CH2 (PA7)
- 右电机：TIM3_CH3 (PB0) + TIM3_CH4 (PB1)

**PWM 初始化关键参数：**
```c
TIM3_PWM_Init(899, 0);
// arr=899：自动重装载值，周期 = (899+1)/72000000 ≈ 12.5μs
// psc=0：不预分频，计数频率 = 72MHz
// PWM 频率 = 72MHz / (899+1) = 80kHz

// 设置占空比（0~899）
TIM_SetCompare1(TIM3, 700);  // CH1 占空比 = 700/899 ≈ 78%
TIM_SetCompare1(TIM3, 10);   // CH1 占空比 = 10/899 ≈ 1%（近似 GND）
```

**差速转向原理：**
```c
// 前进：两轮同向同速
// 左转：左轮慢、右轮快（或左轮反转）
void left_init_pwm(int l, int r) {
    TIM_SetCompare1(TIM3, l);  // 左轮速度 = l
    TIM_SetCompare2(TIM3, 10); // 另一路接地
    TIM_SetCompare3(TIM3, 10); // 另一路接地
    TIM_SetCompare4(TIM3, r);  // 右轮速度 = r
}
```

> **思考题**：`TIM_SetCompare1(TIM3, 10)` 为什么设为 10 而不是 0？如果项目中 arr=899，占空比为 10/899 ≈ 1%，对电机有何影响？

### 3.4 动手实验

1. 单独测试电机：写一段代码让机器人前进 2 秒、停止 1 秒、后退 2 秒
2. 修改 PWM 频率：把 `TIM3_PWM_Init(899,0)` 改为 `TIM3_PWM_Init(999,71)`，计算新的 PWM 频率是多少？
3. 观察不同占空比（100/300/500/700）下电机转速的变化

---

## 阶段四：超声波测距

### 4.1 目标

掌握定时器输入捕获模式，理解超声波测距原理。

### 4.2 需要阅读的文件

| 文件 | 说明 |
|------|------|
| [ultrasonic.h](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/ultrasonic.h) | 超声波模块接口定义 |
| [ultrasonic.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/ultrasonic.c) | 超声波驱动（含中值滤波） |

### 4.3 知识点

**HC-SR04 工作流程：**
```
1. MCU 发送 ≥10μs 高电平到 TRIG → 模块发出 8 个 40kHz 超声波
2. 模块 ECHO 引脚变高 → 等待回波
3. 收到回波 → ECHO 变低 → 高电平持续时间 = 声波往返时间
4. 距离(cm) = 声速(cm/s) × 时间(s) / 2
```

**TIM2 输入捕获配置：**
```c
// 预分频：TIM2 计数频率 = 72MHz / (71+1) = 1MHz → 每个计数值 = 1μs
TIM_TimeBaseStructure.TIM_Prescaler = 71;

// 输入捕获配置：上升沿触发
TIM2_ICInitStructure.TIM_ICPolarity = TIM_ICPolarity_Rising;

// 中断处理中切换捕获边沿
// 第一次中断（上升沿）→ 计数器清零 → 切换为下降沿捕获
// 第二次中断（下降沿）→ 读取计数值 TIME → 切换回上升沿
```

**距离计算：**
```c
// 声速 340m/s = 34000cm/s
// TIME 单位是 μs，转换为秒：TIME / 1000000
// 往返除以 2
// distance = TIME * 3400000 / 1000000 / 20000
//          = TIME * 340 / 20000
distance = (float)TIME * 340 / 20000;
```

**中值滤波（抗干扰）：**
```c
#define N 13  // 采集 13 个样本
// 1. 连续采集 N 次距离值
// 2. 冒泡排序
// 3. 取中间值 ArrDataBuffer[(N-1)/2] 作为本次测量结果
// 优点：有效滤除偶尔的超大/超小干扰值
```

> **思考题**：中值滤波的 N=13 是奇数，为什么通常取奇数？如果 N=10 会怎样？

### 4.4 动手实验

1. 用串口打印超声波距离值，用手遮挡传感器观察数值变化
2. 把中值滤波改为**均值滤波**（去掉最大最小值后取平均），对比两种滤波效果
3. 修改 TIM2 预分频值，让计数器溢出更快/更慢，观察测量范围的变化

---

## 阶段五：主控逻辑与避障算法

### 5.1 目标

整合所有模块，理解主控制流程和避障策略。

### 5.2 需要阅读的文件

| 文件 | 说明 |
|------|------|
| [main.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/main.c) | 主控制逻辑（核心） |

### 5.3 知识点

**系统初始化顺序（顺序很重要！）：**
```c
NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);  // 1. 中断分组
uart_init(9600);         // 2. 调试串口
LED_Init();              // 3. LED
delay_init();            // 4. 延时（依赖 SysTick，后续模块都可能用到）
TIM3_PWM_Init(899,0);   // 5. 电机 PWM
HC05_Init();             // 6. 蓝牙（依赖 USART2）
ultrasonic_Init();       // 7. 超声波
```

**主循环逻辑（状态机思想）：**

```
┌─────────────────────────────────┐
│ 检查 USART2 是否有新数据        │
│   ├─ "F" → 前进                 │
│   ├─ "B" → 后退                 │
│   ├─ "L" → 左转                 │
│   ├─ "R" → 右转                 │
│   ├─ "S" → 停止，flag=0         │
│   ├─ "A" → 加速（PWM+50）       │
│   └─ "D" → 减速（PWM-50）       │
├─────────────────────────────────┤
│ 读取超声波距离（中值滤波）       │
│   ├─ ≤5cm 且 flag=1 → 避障     │
│   ├─ 5~10cm → PWM=200           │
│   ├─ 10~15cm → PWM=300          │
│   ├─ 15~20cm → PWM=400           │
│   └─ >20cm → 保持当前速度        │
├─────────────────────────────────┤
│ LED 翻转（100ms 心跳）           │
└─────────────────────────────────┘
```

**避障子流程：**
```c
if((distance <= 5) && flag) {
    stod_init();          // 1. 停止
    delay_ms(1000);       // 2. 等待 1 秒
    left_init_pwm(l,r);   // 3. 左转
    delay_ms(500);        // 4. 转 0.5 秒
    run_init_pwm(l,r);    // 5. 继续前进
}
```

> **思考题**：避障时只是左转然后前进，如果左边也有障碍怎么办？你会如何改进？

### 5.4 动手实验

1. 在避障逻辑中加入**右转备选**：如果左转后仍 ≤5cm，改为右转
2. 给蓝牙添加新指令 `"X"`——让机器人自动避障巡航（不加遥控持续前进，遇障自动转向）
3. 使用串口打印 `flag` 和 `distance` 值，绘制机器人的状态转换图

---

## 进阶拓展方向

完成基础学习后，可以尝试以下进阶方向：

| 方向 | 内容 | 涉及知识点 |
|------|------|-----------|
| 速度闭环控制 | 添加编码器，PID 控制精确调速 | 编码器接口、PID 算法 |
| 红外循迹 | 用红外对管实现循迹功能 | ADC 采集、PID 循迹算法 |
| 无线遥控升级 | 用 ESP8266 WiFi 模块替代 HC-05 | AT 指令、TCP/UDP 通信 |
| 传感器融合 | 添加红外避障、陀螺仪等 | I2C/SPI 通信、姿态解算 |
| FreeRTOS 移植 | 引入实时操作系统 | 多任务、信号量、消息队列 |

---

## 常用调试技巧

1. **串口是调试利器**：不确定变量值时，用 `printf` 打印出来
2. **LED 心跳**：主循环中用 LED 闪烁确认程序没有卡死
3. **分模块调试**：新模块先单独测试再集成，不要一次写完所有的再调试
4. **断点 + Watch 窗口**：Keil 仿真时在关键行设断点，观察变量变化
5. **逻辑分析仪**：调试 HC-SR04、串口时序时的神器

---

## 参考资源

- [STM32F10x 标准外设库使用手册](https://www.st.com)
- [《STM32 中文参考手册》](https://www.st.com)（寄存器级详解）
- [《Cortex-M3 权威指南》](https://developer.arm.com)（位带操作原理来源）
- [HC-SR04 超声波模块数据手册](https://www.sparkfun.com)
- [HC-05 蓝牙模块 AT 指令集](https://www.alldatasheet.com)
- [L298N 电机驱动原理](https://www.st.com)