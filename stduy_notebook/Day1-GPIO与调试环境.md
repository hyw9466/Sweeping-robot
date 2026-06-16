# 学习笔记 - Day 1

> 日期：2026-06-16  
> 项目：STM32 扫地机器人嵌入式固件

---

## 今日学习内容

### 1. 项目编译环境搭建

- 安装 **ARM Compiler 5**（64 位 Windows 兼容 32 位编译器）
- Keil 魔法棒 → Target → 选择 ARM Compiler 5 版本
- 成功编译：0 Error, 1 Warning（仅末尾换行符警告，不影响功能）
- 程序大小：Code=10KB, SRAM=1.7KB，STM32F103C8T6 绰绰有余

### 2. Keil 模拟器调试

- 魔法棒 → Debug → Use Simulator（纯软件运行，无需硬件）
- **Ctrl+F5** 进入/退出调试模式
- **F5** 运行，**F10** 单步跳过，**F11** 单步进入函数，**Ctrl+F11** 跳出函数
- 设断点：进入调试后点击行号左侧空白处
- 看串口输出：View → Serial Windows → **UART #2**（本项目的 printf 输出到 USART2）
- 手动改变量：Watch 窗口双击变量值修改
- 模拟蓝牙指令：把 `USART2_RX_BUF` 写为 `F`，`USART2_RX_STA` 改为 `0x8001`

### 3. GPIO 外设驱动复习

#### 3.1 GPIO 初始化四步法

```c
// ① 开时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
// ② 配置结构体（引脚、模式、速度）
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_13;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
// ③ 调用初始化
GPIO_Init(GPIOC, &GPIO_InitStructure);
// ④ 设置初始电平
GPIO_SetBits(GPIOC, GPIO_Pin_13);
```

#### 3.2 项目中用到的 4 种 GPIO 模式

| 模式 | 用途 | 文件 |
|------|------|------|
| 推挽输出 `Out_PP` | LED 控制 | led.c (PC13) |
| 推挽输出 `Out_PP` | 超声波 TRIG | ultrasonic.c (PA1) |
| 推挽输出 `Out_PP` | HC-05 KEY | hc05.c (PC4) |
| 上拉输入 `IPU` | HC-05 状态读取 | hc05.c (PA4) |
| 下拉输入 `IPD` | 超声波 ECHO | ultrasonic.c (PA0) |

#### 3.3 位带操作（Bit-Banding）

**核心公式：**
```c
#define BITBAND(addr, bitnum) \
    ((addr & 0xF0000000) + 0x2000000 + ((addr & 0xFFFFF) << 5) + (bitnum << 2))
```

**实际使用：**
```c
#define LED      PCout(13)  // PC13 输出
#define HC05_KEY PCout(4)   // PC4 输出
#define HC05_LED PAin(4)    // PA4 输入

LED = 1;       // 亮灯，一条指令搞定
HC05_KEY = 1;  // 进入 AT 模式
if(HC05_LED) { ... }  // 读取状态
```

**优势：** 原子操作（不会被中断打断）、高效、代码简洁

#### 3.4 GPIO 时钟总线

- GPIOA~G 全部挂在 **APB2** 总线（72MHz）
- 对应时钟函数：`RCC_APB2PeriphClockCmd`

### 4. 编码问题

- 项目源码是 **GBK** 编码（Keil 默认）
- Trae IDE 默认 UTF-8 → 中文注释显示乱码
- 解决：右下角编码区 → Reopen with Encoding → GBK

---

## 今日实操记录

- [x] 安装 ARM Compiler 5，成功编译项目
- [x] 在 Keil 模拟器中运行，UART #2 窗口看到 `AT` 和 `0 cm` 输出
- [x] 在 `while(1)` 设断点，F10 单步跟踪主循环
- [x] F11 进入 `MiddleValueFilter()` 函数，观察 13 次采样 + 冒泡排序
- [x] 手动修改 `USART2_RX_BUF` 和 `USART2_RX_STA` 模拟蓝牙指令
- [x] 阅读 GPIO 相关源码（sys.h, led.c/h, hc05.c/h, ultrasonic.c/h）

---

## 明日学习计划

**主题：UART 串口通信与 DMA**

根据 [LEARNING_GUIDE.md](file:///e:/embedded-linux-project/Sweeping-robot/LEARNING_GUIDE.md) 阶段二内容：

1. **USART 串口通信原理**
   - 波特率、数据位、停止位、校验位
   - 一帧数据格式（起始位 + 8 位数据 + 停止位 = 10 bit）
   - 波特率计算公式：`USARTDIV = f_PCLK / (16 × BaudRate)`

2. **printf 重定向**
   - 重写 `fputc` 函数，将 printf 输出到 USART1
   - 半主机模式 vs MicroLIB 两种方式对比

3. **USART2 接收协议设计**
   - 不定长帧 + 超时判断（TIM4 定时器 10ms）
   - `USART2_RX_STA` 状态字设计（bit15 完成标志，bit13~0 字节计数）
   - 中断服务函数设计原则（快进快出）

4. **DMA 直接存储器访问**
   - DMA1_Channel7 配合 USART2 发送
   - DMA 工作流程：CPU 配置 → DMA 后台搬运 → 传输完成中断
   - 内存对齐优化（`__align(8)`）

5. **HC-05 蓝牙 AT 指令**
   - AT 指令收发流程
   - 在模拟器中手动模拟 AT 指令交互

**准备文件：** usart.c, usart2.c, hc05.c, delay.c