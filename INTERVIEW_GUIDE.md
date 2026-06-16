# 嵌入式软件开发面试指南

> 基于 STM32 扫地机器人项目，涵盖嵌入式面试高频考点。每个问题都结合本项目源码给出参考答案。

---

## 目录

- [一、C 语言基础](#一c-语言基础)
- [二、STM32 与 ARM 架构](#二stm32-与-arm-架构)
- [三、GPIO 与位带操作](#三gpio-与位带操作)
- [四、中断与 NVIC](#四中断与-nvic)
- [五、定时器与 PWM](#五定时器与-pwm)
- [六、串口通信（USART）](#六串口通信usart)
- [七、DMA 直接存储器访问](#七dma-直接存储器访问)
- [八、传感器与信号处理](#八传感器与信号处理)
- [九、系统设计与调试](#九系统设计与调试)
- [十、综合场景题](#十综合场景题)

---

## 一、C 语言基础

### Q1：`volatile` 关键字的作用？本项目中哪些变量应该用 `volatile`？

**参考答案：**

`volatile` 告诉编译器不要优化该变量，每次访问都从内存地址读取，而不是使用寄存器缓存。适用于：
- 在中断服务函数中被修改的变量
- 多线程/多任务共享的变量
- 映射到硬件寄存器的变量

**本项目中的例子：**

```c
// 1. sys.h 中的位带宏——本质是强制 volatile 访问
#define MEM_ADDR(addr)  *((volatile unsigned long *)(addr))

// 2. 在中断中修改、在主循环中读取的变量，应该声明为 volatile
extern u32 TIME;       // TIM2 中断中写入，main 中通过 get_distance() 读取
u16 USART2_RX_STA;     // USART2 中断中写入，main 中轮询判断

// 3. delay.c 中，SysTick 的 VAL 寄存器每次读取值不同，必须 volatile
// SysTick->VAL 本身在头文件中已定义为 volatile
```

> **延伸**：如果 `USART2_RX_STA` 不加 `volatile`，编译器优化后可能把 `if(USART2_RX_STA&0X8000)` 优化成只读一次寄存器缓存值，导致永远检测不到接收完成。

---

### Q2：`static` 关键字的三种用法？结合项目举例。

**参考答案：**

| 用法 | 含义 | 本项目例子 |
|------|------|-----------|
| 静态局部变量 | 函数内保持值，生命周期=整个程序 | `main()` 中的 `static int l=500, r=500;` |
| 静态全局变量 | 文件内可见，其他文件不可访问 | `delay.c` 中的 `static u8 fac_us=0;` |
| 静态函数 | 仅本文件内可调用 | 本项目未使用，但常用于隐藏实现细节 |

**本项目中关键例子：**

```c
// main.c — static 局部变量保存调速状态
static int l=500, r=500;  // 保持左右轮 PWM 值，每次指令不丢失

// delay.c — static 全局变量，仅供 delay.c 内部使用
static u8  fac_us=0;    // us 延时倍乘数
static u16 fac_ms=0;    // ms 延时倍乘数
```

---

### Q3：`const` 的用法？以下两种写法有什么区别？

```c
const int *p;       // ①
int * const p;      // ②
const int * const p; // ③
```

**参考答案：**

- ① **指针指向的内容不可变**：`*p = 10;` 错误，`p = &b;` 正确
- ② **指针本身不可变**：`*p = 10;` 正确，`p = &b;` 错误
- ③ **都不可变**：既不能改指向，也不能改内容

**本项目例子：**
```c
#define USART_REC_LEN 200       // 宏常量，简单但无类型检查
// 更推荐：static const u16 USART_REC_LEN = 200;
```

---

### Q4：解释 `__align(8)` 的作用。

**本项目代码：** [usart2.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart2.c)

```c
__align(8) u8 USART2_TX_BUF[USART2_MAX_SEND_LEN];
```

要求数组按 8 字节地址对齐。DMA 传输时，内存地址对齐可以：
- 提高 DMA 传输效率
- 避免某些硬件平台上的对齐异常
- 配合 Cache 行对齐优化性能

---

### Q5：位运算实战。项目中如何实现单比特操作？

**本项目代码：** [sys.h](file:///e:/embedded-linux-project/Sweeping-robot/USER/sys.h)

```c
// STM32 位带操作——将每个 GPIO 引脚映射成独立的 32 位地址
#define BITBAND(addr, bitnum) ((addr & 0xF0000000)+0x2000000+((addr &0xFFFFF)<<5)+(bitnum<<2))
#define MEM_ADDR(addr)  *((volatile unsigned long *)(addr))
#define PCout(n)   BIT_ADDR(GPIOC_ODR_Addr,n)  // 输出
#define PCin(n)    BIT_ADDR(GPIOC_IDR_Addr,n)   // 输入

// 效果：LED = 1;  等同于操作 PC13 引脚
```

**位运算常用操作：**

| 操作 | 写法 |
|------|------|
| 置位 bit3 | `reg \|= (1 << 3)` |
| 清零 bit3 | `reg &= ~(1 << 3)` |
| 翻转 bit3 | `reg ^= (1 << 3)` |
| 读取 bit3 | `(reg >> 3) & 1` |

---

## 二、STM32 与 ARM 架构

### Q6：STM32 的启动流程是怎样的？

**参考答案：**

```
上电复位
  → 从 0x00000000 取 MSP（主堆栈指针）
  → 从 0x00000004 取 PC（复位向量 = Reset_Handler）
  → 执行 Reset_Handler：
      1. 调用 SystemInit() 配置系统时钟
      2. 从 Flash 复制 .data 段到 SRAM（初始化全局变量）
      3. 清零 .bss 段（未初始化全局变量）
      4. 调用 main()
```

**本项目相关文件：**
- `startup_stm32f10x_hd.s`：启动汇编文件，定义向量表
- [system_stm32f10x.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/system_stm32f10x.c)：`#define SYSCLK_FREQ_72MHz 72000000`，配置 72MHz 系统时钟

---

### Q7：STM32 的时钟树，简述 HSI/HSE/PLL 的关系。

**参考答案：**

```
HSI (8MHz 内部RC)
  └── 精度低，启动快，做备份时钟
HSE (8MHz 外部晶振)
  └── 精度高，本项目的时钟源
      └── PLL 倍频：8MHz × 9 = 72MHz (SYSCLK)
          ├── AHB 总线 (HCLK = 72MHz)
          ├── APB1 (PCLK1 = 36MHz，给 TIM2/3/4、USART2 等)
          └── APB2 (PCLK2 = 72MHz，给 GPIO、USART1、TIM1 等)
```

**本项目关键点：**
- `system_stm32f10x.c` 配置 `SYSCLK_FREQ_72MHz`
- `delay.c` 使用 `SysTick_CLKSource_HCLK_Div8`（72/8=9MHz → `fac_us=9`）
- 每个外设初始化前都要 `RCC_xxxPeriphClockCmd()` 开时钟

> **面试常问**：串口波特率怎么算？`USARTDIV = f_PCLK / (16 × BaudRate)`，例如 USART1 挂 APB2=72MHz，9600 波特率 → `USARTDIV = 72000000 / (16×9600) = 468.75`

---

### Q8：Cortex-M3 的异常/中断有哪些？HardFault 怎么排查？

**本项目代码：** [stm32f10x_it.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/stm32f10x_it.c)

```c
// Cortex-M3 系统异常（优先级固定，负数）
void NMI_Handler(void);         // 不可屏蔽中断
void HardFault_Handler(void);   // 硬件错误（访问非法地址、非对齐等）
void MemManage_Handler(void);   // MPU 内存管理错误
void BusFault_Handler(void);    // 总线错误
void UsageFault_Handler(void);  // 未定义指令、除零等
void SVC_Handler(void);         // 系统服务调用
void PendSV_Handler(void);      // 可挂起系统调用（RTOS 上下文切换）
void SysTick_Handler(void);     // 系统滴答定时器
```

**HardFault 排查方法：**
1. 在 `HardFault_Handler` 中设断点
2. 查看 `CFSR`（Configurable Fault Status Register）判断错误类型
3. 查看 `PC`（程序计数器）定位出错的代码地址
4. 常见原因：访问野指针、栈溢出、外设未开时钟就操作、非对齐访问

---

## 三、GPIO 与位带操作

### Q9：GPIO 的 8 种模式分别是什么？什么时候用推挽，什么时候用开漏？

**参考答案：**

| 模式 | 用途 |
|------|------|
| 浮空输入 | 按键检测（外部上拉/下拉） |
| 上拉输入 | 默认高电平的输入 |
| 下拉输入 | 默认低电平的输入 |
| 模拟输入 | ADC 采集 |
| 推挽输出 | LED、电机方向控制（能输出高低电平） |
| 开漏输出 | I2C 总线（需要外部上拉） |
| 复用推挽 | USART TX、PWM 输出 |
| 复用开漏 | I2C SDA 等 |

**本项目配置对应：**

```c
// LED 控制 [led.c]
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;    // 推挽输出

// 超声波 ECHO 检测 [ultrasonic.c]
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPD;       // 下拉输入

// PWM 输出 [motor.c]
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;     // 复用推挽

// HC-05 KEY 引脚 [hc05.c]
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;       // 上拉输入

// HC-05 LED 状态引脚 [hc05.c]
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;    // 推挽输出
```

---

### Q10：位带操作（Bit-Banding）的原理是什么？

**本项实现：** [sys.h](file:///e:/embedded-linux-project/Sweeping-robot/USER/sys.h)

```
SRAM 位带区：0x20000000 ~ 0x200FFFFF (1MB)
SRAM 位带别名区：0x22000000 ~ 0x23FFFFFF (32MB)

外设位带区：0x40000000 ~ 0x400FFFFF (1MB)
外设位带别名区：0x42000000 ~ 0x43FFFFFF (32MB)

映射关系：bit_word_addr = bit_band_base + (byte_offset × 32) + (bit_number × 4)

本质原理：位带区的 1 个 bit → 别名区的 1 个 32-bit word
         写别名区 = 写位带区的目标位（原子操作，无需读-改-写）
```

**位带操作的优势：**
- 单条指令完成单比特操作
- 线程安全（原子操作）
- 无需关中断保护

---

## 四、中断与 NVIC

### Q11：NVIC 中断优先级分组怎么理解？

**本项目代码：** [main.c](file:///e:/embedded-linux-project/Sweeping-robot/USER/main.c)

```c
NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
// 分组 2：2 位抢占优先级 + 2 位子优先级
// 抢占优先级：0~3（4 级）
// 子优先级：0~3（4 级）
```

| 分组 | 抢占位 | 子优先级位 | 抢占级数 | 子优先级级数 |
|------|--------|-----------|---------|-------------|
| NVIC_PriorityGroup_0 | 0 | 4 | 1 | 16 |
| NVIC_PriorityGroup_1 | 1 | 3 | 2 | 8 |
| **NVIC_PriorityGroup_2** | **2** | **2** | **4** | **4** |
| NVIC_PriorityGroup_3 | 3 | 1 | 8 | 2 |
| NVIC_PriorityGroup_4 | 4 | 0 | 16 | 1 |

**本项目中各中断优先级：**

| 中断源 | 抢占优先级 | 子优先级 | 说明 |
|--------|-----------|---------|------|
| USART1 | 3 | 3 | 调试串口，优先级最低 |
| TIM2 | 2 | 0 | 超声波输入捕获，需要精确计时 |
| USART2 | 2 | 3 | 蓝牙接收，稍低于 TIM2 |
| TIM4 | 默认 | 默认 | USART2 接收超时判断 |

> **思考**：TIM2 和 USART2 的抢占优先级都是 2，但 TIM2 子优先级更高(0<3)，当同时挂起时 TIM2 先响应。

---

### Q12：中断服务函数（ISR）应该遵循哪些原则？

**参考答案：**

1. **快进快出**：ISR 中只做必要操作，复杂处理放主循环
2. **不要用延时**：ISR 中 `delay_ms()` 会阻塞其他中断
3. **共享变量加 volatile**
4. **注意重入**：函数如果在 ISR 和主循环都用，必须可重入

**本项目 ISR 设计分析：**

```c
// 好的设计——ISR 只记录数据，主循环处理
void USART2_IRQHandler(void) {
    res = USART_ReceiveData(USART2);
    USART2_RX_BUF[USART2_RX_STA++] = res;  // 只存数据
}
// 主循环中判断 USART2_RX_STA&0X8000 后统一处理

// 超声波 ISR——只做边沿切换和计数值保存
void TIM2_IRQHandler(void) {
    if(i==0) {
        TIM_SetCounter(TIM2, 0);                     // 清零计数器
        TIM_OC1PolarityConfig(TIM2, TIM_ICPolarity_Falling);  // 切换边沿
        i = 1;
    } else {
        TIME = TIM_GetCounter(TIM2);                 // 保存计数值
        TIM_OC1PolarityConfig(TIM2, TIM_ICPolarity_Rising);
        i = 0;
    }
    TIM_ClearITPendingBit(TIM2, TIM_IT_CC1|TIM_IT_Update);  // 必须清标志！
}
```

---

### Q13：中断标志位为什么必须手动清除？

STM32 外设中断标志位不会自动清零（除少数例外），忘记清零会导致中断**反复触发**，表现为：
- 程序卡死在 ISR
- 其他中断无法响应
- 看门狗复位

**本项目中的清理操作：**
```c
TIM_ClearITPendingBit(TIM2, TIM_IT_CC1|TIM_IT_Update);  // 超声波
TIM_ClearITPendingBit(TIM4, TIM_IT_Update);             // 串口超时
// USART1 读取 DR 寄存器时自动清除 RXNE 标志
```

---

## 五、定时器与 PWM

### Q14：STM32 定时器的输入捕获和输出比较有什么区别？

| | 输入捕获 | 输出比较 / PWM |
|------|----------|---------------|
| 方向 | 外部信号 → 定时器 | 定时器 → 外部引脚 |
| 作用 | 测量脉宽/频率 | 产生波形 |
| 本项目中 | TIM2 捕获超声波 ECHO | TIM3 输出 PWM 控制电机 |

**输入捕获测量脉宽**（本项目 [ultrasonic.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/ultrasonic.c)）：
```
上升沿 → 捕获计数值 → 清零计数器 → 切换为下降沿
下降沿 → 捕获计数值 → TIME = 计数值 → 切换回上升沿
```

---

### Q15：PWM 的频率和占空比怎么计算？

**本项目代码：** [motor.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/motor.c)

```c
TIM3_PWM_Init(899, 0);
// arr = 899, psc = 0
// PWM 频率 = TIM3_CLK / (arr+1) / (psc+1)
//          = 72MHz / 900 / 1
//          = 80kHz

// 占空比 = CCR / (arr+1) × 100%
// TIM_SetCompare1(TIM3, 700) → 700/900 ≈ 77.8%
// TIM_SetCompare1(TIM3, 10)  → 10/900 ≈ 1.1%（近似接地）

// 如果要以 20kHz 频率输出：
// arr = 72MHz/20kHz/1 - 1 = 3599
```

**调速范围（本项目）：**
```c
l += 50;  // 加速：500 → 550 → 600 → 650 → 700
l -= 50;  // 减速：500 → 450 → ... → 100
// 调速范围：100~700，对应占空比约 11%~78%
```

---

### Q16：高级定时器和通用定时器的区别？

| 特性 | 通用定时器 (TIM2/3/4) | 高级定时器 (TIM1/8) |
|------|----------------------|---------------------|
| 计数器位数 | 16 位 | 16 位 |
| 互补输出 | 无 | 有（带死区） |
| 刹车功能 | 无 | 有 |
| 适用场景 | 普通 PWM、输入捕获 | 电机控制、电源逆变 |

本项目的 TIM3 是通用定时器，`TIM_CtrlPWMOutputs(TIM3,ENABLE)` 对于通用定时器其实不需要调用（仅高级定时器需要）。

---

## 六、串口通信（USART）

### Q17：UART 通信的波特率、数据位、停止位、校验位各是什么？

| 参数 | 本项目配置 | 说明 |
|------|-----------|------|
| 波特率 | 9600 bps | 每秒传输 9600 个 bit |
| 数据位 | 8 bit | 每个字节的有效数据 |
| 停止位 | 1 bit | 帧结束标志 |
| 校验位 | 无 | 不使用奇偶校验 |

**一帧数据 = 起始位(1) + 数据位(8) + 停止位(1) = 10 bit**
**有效传输速率 = 9600 / 10 = 960 字节/秒**

---

### Q18：本项目的 USART2 接收协议是如何设计的？为什么这样设计？

**协议格式：** 不定长字符串帧，以 10ms 超时作为帧结束标志。

```c
// 接收状态字 USART2_RX_STA
// bit15: 接收完成标志
// bit14: 保留
// bit13~0: 接收字节计数

// 工作流程：
// 1. 收到第一个字节 → 启动 TIM4 定时器（10ms 超时）
// 2. 每收到一个字节 → 重置 TIM4 计数器（重新计时 10ms）
// 3. 10ms 内无新字节 → TIM4 溢出中断 → 置位 bit15
// 4. 主循环检测 bit15 → 处理 USART2_RX_BUF 中的数据
```

**为什么会选择这种协议？**

| 方案 | 优点 | 缺点 |
|------|------|------|
| **不定长 + 超时判断（本项目）** | 灵活，无需转义，适合 AT 指令 | 需要定时器资源 |
| 固定长度 | 简单可靠 | 浪费带宽 |
| 特殊结束符（\r\n） | 类似 AT 指令风格 | 二进制数据需转义 |
| 帧头+长度+数据+校验 | 可靠，支持二进制 | 协议复杂 |

> **本项目实际是两层设计**：USART2 用超时判断帧结束，具体指令用 `strcmp` 匹配字符串（如 `"F"`, `"B"`）

---

### Q19：printf 重定向到串口的原理？

**本项目代码：** [usart.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart.c)

```c
// 核心：重写 fputc，使得 printf 输出到 USART1
int fputc(int ch, FILE *f) {
    while((USART1->SR & 0x40) == 0);  // 等待 TC 标志（发送完成）
    USART1->DR = (u8)ch;              // 写入数据寄存器
    return ch;
}

// 要正常工作还需避免半主机模式：
#pragma import(__use_no_semihosting)  // 不使用半主机
_sys_exit(int x) { x = x; }          // 空实现 _sys_exit
```

**MicroLIB 方式**（代码中注释掉的部分）：勾选 Keil 的 "Use MicroLIB" 选项后，只需重写 `fputc`，无需 `_sys_exit` 和 `__stdout`。

**半主机模式**：ARM 的一种调试机制，printf 默认通过调试器输出到 IDE。嵌入式环境中需要关闭。

---

## 七、DMA 直接存储器访问

### Q20：DMA 的工作流程是怎样的？

```
CPU 配置 DMA：
  1. 源地址（内存 USART2_TX_BUF）
  2. 目的地址（外设 USART2->DR）
  3. 传输长度（字节数）
  4. 触发源（USART2_TX 请求）
  → 启动 DMA
  → DMA 在后台自动搬运数据，CPU 可以干别的事
  → 传输完成可触发中断
```

**本项目 DMA 使用：** [usart2.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/usart2.c)

```c
// 初始化：DMA1 通道7，外设 = USART2->DR，内存 = USART2_TX_BUF
UART_DMA_Config(DMA1_Channel7, (u32)&USART2->DR, (u32)USART2_TX_BUF);

// 发送：直接发字符串
void u2_printf(char* fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    vsprintf((char*)USART2_TX_BUF, fmt, ap);
    va_end(ap);
    while(DMA_GetCurrDataCounter(DMA1_Channel7) != 0);  // 等待上一帧发完
    UART_DMA_Enable(DMA1_Channel7, strlen((const char*)USART2_TX_BUF));
}
```

> **注意**：`while(DMA_GetCurrDataCounter...)` 等待上一帧发送完成——这是一个简单的互斥保护，防止覆盖正在发送的缓冲区。

---

### Q21：DMA 通道和请求映射，以本项目为例说明。

STM32F103 有两个 DMA 控制器（DMA1 和 DMA2），每个有 7 个通道，通道和外设的映射是**硬件固定**的：

**本项目映射：**
| DMA | 通道 | 外设 | 用途 |
|-----|------|------|------|
| DMA1 | Channel7 | USART2_TX | 蓝牙串口 DMA 发送 |

**其他常见映射：**
| DMA | 通道 | 外设 |
|-----|------|------|
| DMA1 | Channel4 | USART1_TX |
| DMA1 | Channel5 | USART1_RX |
| DMA1 | Channel1 | ADC1 |

---

## 八、传感器与信号处理

### Q22：HC-SR04 超声波测距的工作原理？

```
1. MCU 给 TRIG ≥ 10μs 高电平
2. 模块自动发送 8 个 40kHz 脉冲
3. ECHO 引脚变高（表示开始等待回波）
4. 声波遇到障碍物反射回来
5. 模块收到回波 → ECHO 变低
6. 距离 = 声速 × ECHO 高电平时间 / 2
```

**本项目实现：** [ultrasonic.c](file:///e:/embedded-linux-project/Sweeping-robot/Drivers/ultrasonic.c)

```c
// 出发脉冲：TRIG 保持 1ms 高电平（远大于 10μs 最小要求）
GPIO_SetBits(ULT_GPIO, Tx_GPIO);
delay_ms(1);
GPIO_ResetBits(ULT_GPIO, Tx_GPIO);

// 距离计算：TIME 是定时器计数值（单位 μs）
// 声速 340m/s → 0.034cm/μs
// 往返 → 除以 2 → 0.017cm/μs
// distance = TIME × 0.017 = TIME × 17 / 1000
// 本项目的公式：TIME × 340 / 20000 = TIME × 0.017
distance = (float)TIME * 340 / 20000;
```

---

### Q23：为什么要做中值滤波？还有其他滤波方式吗？

**本项目中值滤波：**

```c
#define N 13  // 采集 13 个样本
// 1. 采集 N 个距离值
// 2. 冒泡排序
// 3. 取中间值 ArrDataBuffer[(N-1)/2]
```

| 滤波方式 | 原理 | 优点 | 缺点 |
|----------|------|------|------|
| **中值滤波** | 排序取中间值 | 抗脉冲干扰强 | 计算量大 |
| 均值滤波 | 取 N 个样本的平均值 | 平滑效果好 | 对脉冲干扰敏感 |
| 限幅滤波 | 两次差值 > 阈值则丢弃 | 简单快速 | 可能丢失真实变化 |
| 卡尔曼滤波 | 递推最优估计 | 动态跟踪好 | 实现复杂 |

**为什么用中值滤波：** 超声波偶尔会因反射干扰产生异常大/小的值，中值滤波能有效剔除这些野值。

---

### Q24：项目中 L298N 为什么用 4 路 PWM，而不是 2 路 PWM + 2 路 GPIO？

```
方案 A（本项目）：4 路 PWM，每路电机用 2 路 PWM
  IN1=PWM, IN2=GND  → 正转
  IN1=GND, IN2=PWM  → 反转
  优点：纯软件控制方向，无需额外电路
  代价：多用 2 路 PWM

方案 B：2 路 PWM + 2 路 GPIO
  GPIO 控制方向，PWM 控制速度
  优点：省 PWM 通道
  缺点：需要额外逻辑电路（如方向控制板）
```

本项目中 TIM3 刚好有 4 路 PWM 可用，且接线最少，所以用方案 A。

---

## 九、系统设计与调试

### Q25：嵌入式系统的程序架构有哪些？本项目属于哪种？

| 架构 | 特点 | 适用场景 |
|------|------|---------|
| **前后台系统（本项目）** | 主循环 + 中断 | 简单任务 |
| 时间触发 | 定时器驱动状态机 | 周期性任务 |
| RTOS | 多任务抢占 | 复杂多任务 |

**本项目的前后台架构：**

```c
int main(void) {
    // ==== 前台：中断服务函数 ====
    // USART2_IRQHandler()  → 接收蓝牙数据
    // TIM2_IRQHandler()    → 捕获超声波脉宽
    // TIM4_IRQHandler()    → 串口接收超时判断
    
    // ==== 后台：主循环 ====
    while(1) {
        // 1. 处理蓝牙指令
        if(USART2_RX_STA & 0X8000) { ... }
        // 2. 读取超声波距离
        temp = MiddleValueFilter();
        // 3. 避障判断与执行
        if(distance <= 5 && flag) { ... }
        // 4. LED 心跳
        LED = !LED;
        delay_ms(100);
    }
}
```

---

### Q26：如果程序跑飞了（HardFault），你会怎么排查？

**排查步骤：**

1. 在 `HardFault_Handler` 设断点
2. 查看调用栈（Call Stack），定位进入 HardFault 前的代码位置
3. 检查寄存器：
   - `PC`：错误发生的地址，通过 `.map` 文件找到对应函数
   - `LR`：返回地址
4. 检查 `CFSR` 寄存器判断错误类型：
   - bit0=1：未定义指令
   - bit1=1：非对齐访问
   - bit3=1：除零
   - bit9=1：精确总线错误
5. **常见原因（本项目场景）：**
   - 数组越界：`USART2_RX_BUF[USART2_RX_STA++]` 如果 `USART2_RX_STA` 超过缓冲区
   - 外设时钟未开启：操作 GPIO 前忘了 `RCC_xxxPeriphClockCmd`
   - 栈溢出：局部变量太大或递归太深

---

### Q27：嵌入式开发中如何做性能优化？

**以本项目为例：**

| 优化点 | 方法 | 本项目体现 |
|--------|------|-----------|
| 中断轻量化 | ISR 只做最少操作 | TIM2 ISR 只记计数值 |
| DMA 代替轮询 | 数据搬运交给 DMA | USART2 发送用 DMA |
| 位带操作 | 单指令操作 GPIO | sys.h 的 LED 控制 |
| 避免浮点运算 | 用整数替代 | 超声波计算用了 float（可优化为整数） |
| 编译器优化 | -O1/-O2 等级 | Keil 中设置优化等级 |

**可优化点**（主动提出可加分）：
```c
// 当前代码用浮点计算
distance = (float)TIME * 340 / 20000;

// 优化为定点整数（单位 0.1cm，避免浮点）
// distance = TIME * 340 / 2000;  → TIME * 0.17
// 用整数放大 10 倍：distance_x10 = TIME * 17 / 100
```

---

## 十、综合场景题

### Q28：如果让你给这个扫地机器人增加手机 APP 控制功能，你会怎么做？

**参考方案：**

```
手机 APP → WiFi → ESP8266 → USART3 → STM32 → 执行动作

方案设计：
1. ESP8266 通过串口 AT 指令接入 WiFi，建立 TCP Server
2. 手机 APP 连接热点，发送 TCP 数据包
3. 自定义协议（JSON 格式，更灵活）：
   {"cmd":"move", "dir":"F", "speed":500}
   {"cmd":"stop"}
   {"cmd":"sensor"}
4. STM32 解析 JSON → 执行对应的运动控制函数
5. 定时上报超声波距离、电机状态给 APP
```

---

### Q29：本项目的按键调速（A/D 指令）有个隐藏 bug，你发现了吗？

```c
// main.c 中的调速逻辑
if((strcmp((const char*)USART2_RX_BUF,"A")==0)&&flag) {
    if(l<700&&r<700) {
        l+=50;
        r+=50;
    }
    // 只调用运动控制，但没有反馈当前速度！
}

if((strcmp((const char*)USART2_RX_BUF,"D")==0)&&flag) {
    if(l>100&&r>100) {
        l-=50;
        r-=50;
    }
}
```

**潜在问题：**

1. **没有限幅保护**：`l` 最高只限制到 700，但如果初始值就是 650，加速一次 `650+50=700` OK，但 PWM 最大是 899，700 并非真正上限
2. **没有速度反馈**：手机 APP 端不知道当前实际速度，`u2_printf` 只打印了方向指令
3. **加减速没有打印确认**：建议加上 `u2_printf("speed:%d\r\n", l);`

---

### Q30：避障逻辑在实际使用中会有什么问题？如何改进？

**当前避障代码：**
```c
if(distance <= 5 && flag) {
    stod_init();                   // 停止
    delay_ms(1000);               // 等 1 秒
    left_init_pwm(l, r);          // 左转
    delay_ms(500);                // 转 0.5 秒
    run_init_pwm(l, r);           // 前进
}
```

**问题分析：**

| 问题 | 原因 | 改进方向 |
|------|------|---------|
| 只左转不死板 | 左边也可能有障碍 | 加入右转备选逻辑 |
| delay 阻塞 | `delay_ms(1000)` 期间无法接收蓝牙指令 | 用非阻塞状态机 + 定时器 |
| 距离阈值固定 | 5cm 挡不住高速前进 | 动态阈值（速度越快阈值越大）|
| 没有后退距离 | 只左转可能原地打转 | 先退一段再转向 |

**改进后的状态机伪代码：**
```c
typedef enum {FORWARD, STOP, BACKOFF, TURN_LEFT, TURN_RIGHT} State;
State state = FORWARD;
u32 state_timer = 0;

switch(state) {
    case FORWARD:
        if(distance <= threshold) state = STOP, state_timer = get_tick();
        break;
    case STOP:     if(tick - state_timer > 300ms) state = BACKOFF; break;
    case BACKOFF:  if(tick - state_timer > 500ms) state = TURN_LEFT; break;
    case TURN_LEFT: 
        if(tick - state_timer > 500ms) {
            if(distance <= threshold) state = TURN_RIGHT; // 左边也堵了就右转
            else state = FORWARD;
        }
        break;
    // ...
}
```

---

### Q31：如果让你用 RTOS（如 FreeRTOS）重构这个项目，任务怎么划分？

```c
// 任务划分方案：

// 任务1：蓝牙通信任务（优先级 3）
void Task_Bluetooth(void *pvParameters) {
    while(1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // 等 USART2 中断通知
        // 解析指令，发送到消息队列
        xQueueSend(cmdQueue, &cmd, 0);
    }
}

// 任务2：传感器任务（优先级 2，周期 50ms）
void Task_Sensor(void *pvParameters) {
    while(1) {
        distance = MiddleValueFilter();
        xQueueOverwrite(distQueue, &distance);  // 写入最新距离
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

// 任务3：运动控制任务（优先级 3，等待指令队列）
void Task_Motion(void *pvParameters) {
    while(1) {
        if(xQueueReceive(cmdQueue, &cmd, portMAX_DELAY)) {
            xQueuePeek(distQueue, &distance, 0);  // 读最新距离
            execute_command(cmd, distance);
        }
    }
}

// 任务4：LED 心跳任务（优先级 0）
void Task_HeartBeat(void *pvParameters) {
    while(1) {
        LED = !LED;
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

**RTOS 带来的改进：**
- 蓝牙指令实时响应（不再被 `delay_ms(1000)` 阻塞）
- 传感器数据周期性采集，不依赖主循环速度
- 各模块解耦，易于扩展

---

## 面试准备清单

- [ ] 能手绘 STM32 时钟树，解释各总线时钟关系
- [ ] 能写 GPIO 初始化代码（不开 IDE）
- [ ] 能解释 USART 一帧数据的格式
- [ ] 能手写位运算操作（置位、清零、翻转、读取）
- [ ] 能画 PWM 波形图，标出周期和占空比
- [ ] 能讲清楚中断从触发到 ISR 执行的完整流程
- [ ] 能分析出一个简单 C 程序的 `.data` / `.bss` / `.stack` 分布
- [ ] 能用本项目源码回答上述任何一个问题
- [ ] 能提出至少 3 个本项目的改进点并给出方案