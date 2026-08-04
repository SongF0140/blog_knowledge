---
title: "ESP32-S3 UART 异步串口通信"
published: 2026-08-20
description: "奇偶校验示例（数据 0x55 = 01010101B，含 4 个 1）："
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","通信协议","UART"]
category: 物联网
draft: false
---
# ESP32-S3 UART 异步串口通信

## 本章重要 API 函数

| API 函数 | 功能 | 参数 | 参数说明 |
|----------|------|------|----------|
| **`uart_param_config`** | 配置 UART 时钟源和通信参数 | `uart_port_t uart_num` | UART 端口号：`UART_NUM_0` / `UART_NUM_1` / `UART_NUM_2` |
| | | `const uart_config_t *uart_config` | 指向配置结构体的指针，内含：<br>• `baud_rate`：波特率（如 9600, 115200）<br>• `data_bits`：数据位（`UART_DATA_5_BITS` ~ `UART_DATA_8_BITS`）<br>• `parity`：校验位（`UART_PARITY_DISABLE` / `UART_PARITY_EVEN` / `UART_PARITY_ODD`）<br>• `stop_bits`：停止位（`UART_STOP_BITS_1` / `UART_STOP_BITS_1_5` / `UART_STOP_BITS_2`）<br>• `flow_ctrl`：流控（`UART_HW_FLOWCTRL_DISABLE` / `UART_HW_FLOWCTRL_RTS` / `UART_HW_FLOWCTRL_CTS` / `UART_HW_FLOWCTRL_CTS_RTS`）<br>• `source_clk`：时钟源（`UART_SCLK_DEFAULT` / `UART_SCLK_APB` / `UART_SCLK_RTC` 等） |
| **`uart_set_pin`** | 配置 UART 的 GPIO 引脚 | `uart_port_t uart_num` | UART 端口号 |
| | | `int tx_io_num` | TX 发送引脚 GPIO 编号，如 `GPIO_NUM_17`，不用则设 `UART_PIN_NO_CHANGE` |
| | | `int rx_io_num` | RX 接收引脚 GPIO 编号，如 `GPIO_NUM_16`，不用则设 `UART_PIN_NO_CHANGE` |
| | | `int rts_io_num` | RTS 流控引脚 GPIO 编号，不用则设 `UART_PIN_NO_CHANGE` |
| | | `int cts_io_num` | CTS 流控引脚 GPIO 编号，不用则设 `UART_PIN_NO_CHANGE` |
| **`uart_driver_install`** | 安装 UART 驱动，分配收发缓冲区 | `uart_port_t uart_num` | UART 端口号 |
| | | `int rx_buffer_size` | 接收环形缓冲区大小（字节），如 `1024`，建议不小于一帧数据量 |
| | | `int tx_buffer_size` | 发送环形缓冲区大小（字节），如 `0`（不使用发送缓冲），或 `1024` |
| | | `int queue_size` | UART 事件队列深度，如 `0`（不使用事件队列），或 `10` |
| | | `QueueHandle_t *uart_queue` | 事件队列句柄指针，不用则传 `NULL` |
| | | `int intr_alloc_flags` | 中断分配标志，如 `0`（默认）或 `ESP_INTR_FLAG_IRAM`（将 ISR 放入 IRAM） |
| **`uart_write_bytes`** | 向发送缓冲区写入数据 | `uart_port_t uart_num` | UART 端口号 |
| | | `const void *src` | 指向待发送数据的指针 |
| | | `size_t size` | 待发送数据的字节数 |
| | | **返回值 `int`** | 成功返回实际写入缓冲区的字节数（可能小于 `size`），失败返回 -1 |
| **`uart_read_bytes`** | 从接收缓冲区读取数据 | `uart_port_t uart_num` | UART 端口号 |
| | | `void *buf` | 接收数据存放的缓冲区指针 |
| | | `uint32_t length` | 期望读取的最大字节数 |
| | | `TickType_t ticks_to_wait` | 超时等待时间（FreeRTOS tick），如 `pdMS_TO_TICKS(1000)` 等待 1000ms，`0` 表示不等待立即返回 |
| | | **返回值 `int`** | 成功返回实际读取的字节数（可能小于 `length`），超时返回 0，失败返回 -1 |

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[学习前后依赖](#学习前后依赖)**
- **[一、UART / I2C / SPI 三者对比](#一uart-/-i2c-/-spi-三者对比)**
  - [双工模式说明](#双工模式说明)
  - [同步 vs 异步](#同步-vs-异步)
- **[二、电平标准](#二电平标准)**
  - [TTL 电平（ESP32-S3 使用 3.3V TTL）](#ttl-电平esp32-s3-使用-33v-ttl)
  - [RS232 电平（负逻辑！）](#rs232-电平负逻辑)
  - [RS485 电平（差分信号）](#rs485-电平差分信号)
- **[三、UART 数据帧结构](#三uart-数据帧结构)**
  - [帧格式](#帧格式)
  - [校验位详解](#校验位详解)
  - [硬件流控（RTS / CTS）](#硬件流控rts-/-cts)
- **[四、数据帧分析实例](#四数据帧分析实例)**
  - [发送数据：0x55（二进制 01010101）](#发送数据0x55二进制-01010101)
  - [每位的持续时间](#每位的持续时间)
  - [帧时序图（逻辑分析仪视角）](#帧时序图逻辑分析仪视角)
  - [奇校验示例：发送 0x55 带奇校验](#奇校验示例发送-0x55-带奇校验)
- **[五、波特率计算](#五波特率计算)**
  - [一帧数据的总位数](#一帧数据的总位数)
  - [传输效率](#传输效率)
  - [常用配置对比](#常用配置对比)
- **[六、ESP32-S3 UART 驱动代码](#六esp32-s3-uart-驱动代码)**
  - [项目结构](#项目结构)
  - [uart.c — UART 初始化模块](#uartc-—-uart-初始化模块)
  - [main.c — 主程序（串口回显）](#mainc-—-主程序串口回显)
  - [串口接收数据的两种显示模式](#串口接收数据的两种显示模式)
  - [其他常用 API](#其他常用-api)
- **[容易踩坑](#容易踩坑)**
- **[Tips](#tips)**
  - [波特率的本质——时钟同步的替代品](#波特率的本质——时钟同步的替代品)
  - [中断接收 vs 轮询接收——如何选择](#中断接收-vs-轮询接收——如何选择)
  - [踩坑实录](#踩坑实录)
  - [知识衔接](#知识衔接)

---

## 学习前后依赖

| 方向 | 内容 |
|------|------|
| 前置基础 | 轮询/中断/DMA 的适用场景、GPIO 引脚复用和基本通信协议概念。 |
| 本章核心 | 掌握 UART 异步串口帧、波特率、收发缓冲、回显流程和文本/Hex 显示。 |
| 后续承接 | 承接上位机通信、日志输出、传感器数据上传和后续自定义协议设计。 |

## 一、UART / I2C / SPI 三者对比

| 特性 | UART | I2C | SPI |
|------|------|-----|-----|
| 通信方式 | 异步（无时钟线） | 同步（有时钟线 SCL） | 同步（有时钟线 SCK） |
| 双工模式 | **全双工**（RX/TX 独立） | **半双工**（共享 SDA） | **全双工**（MOSI/MISO 独立） |
| 连线数量 | 最少 3 根（RX/TX/GND） | 2 根（SDA/SCL）+ GND | 最少 4 根（MOSI/MISO/SCK/CS） |
| 拓扑结构 | **一对一**（点对点） | **一对多**（总线地址寻址） | **一对多**（CS 片选信号选从机） |
| 速率 | 较低（常用 9600~115200 bps） | 标准 100k/400k，快速 1M/3.4M | 较高（可达几十 MHz） |
| 硬件复杂度 | 简单 | 中等 | 较高（多从机需多 CS 线） |
| 流控 | 可选（RTS/CTS） | 无 | 无 |

### 双工模式说明

- **单工（Simplex）**：数据只能单向传输，如广播。只有一根数据线，一方只发、一方只收。
- **半双工（Half-Duplex）**：数据可双向传输，但同一时刻只能一个方向。如 I2C，SDA 线分时复用。
- **全双工（Full-Duplex）**：数据可同时双向传输，收发独立。如 UART（RX/TX 独立）、SPI（MOSI/MISO 独立）。

### 同步 vs 异步

- **同步通信**（I2C、SPI）：有独立的时钟线，收发双方按同一时钟节拍工作，速率高。
- **异步通信**（UART）：没有时钟线，双方约定相同波特率，通过起始位/停止位同步，速率较低。

---

## 二、电平标准

| 标准 | 逻辑 0 | 逻辑 1 | 适用范围 | 特点 |
|------|--------|--------|----------|------|
| **TTL** | 0V ~ 0.4V | 2.4V ~ 5V（5V TTL）<br>2.4V ~ 3.3V（3.3V TTL） | 芯片间短距离通信 | 电平低、功耗小、距离短 |
| **RS232** | +3V ~ +15V | -3V ~ -15V | 早期 PC 串口 | **负逻辑**，电平高，距离 < 15m |
| **RS485** | A-B < -200mV | A-B > +200mV | 工业现场总线 | **差分信号**，抗干扰强，距离 > 1km |

### TTL 电平（ESP32-S3 使用 3.3V TTL）

- 逻辑 1（高电平）：2.4V ~ VCC（3.3V 或 5V）
- 逻辑 0（低电平）：0V ~ 0.4V
- **温度特性**：温度升高时，阈值可能漂移，但 TTL 一般使用场景温度范围较窄（0~70°C 商用级）
- **电压特性**：对电源噪声敏感，适合板级短距离连接

### RS232 电平（负逻辑！）

- 逻辑 0（SPACE）：+3V ~ +15V（正电压表示 0！）
- 逻辑 1（MARK）：-3V ~ -15V（负电压表示 1！）
- **注意**：RS232 是负逻辑，与 TTL 的电平逻辑相反，需要 MAX232 等芯片做电平转换
- **温度特性**：工业级 -40°C ~ +85°C，电压范围宽，驱动能力强

### RS485 电平（差分信号）

- 逻辑 1：A 线电压 - B 线电压 > +200mV
- 逻辑 0：A 线电压 - B 线电压 < -200mV
- **差分传输优势**：两根线同时受干扰，差值不变，共模抑制能力强
- **温度特性**：工业级 -40°C ~ +85°C，适合恶劣环境
- **电压特性**：共模电压范围 -7V ~ +12V，允许较大的地电位差

---

## 三、UART 数据帧结构

### 帧格式

```
         ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌─────
─────────┘   └──┘   └──┘   └──┘   └──┘   └──┘   └──┘   └──┘   └──┘
 IDLE   St  D0   D1  D2   D3  D4   D5  D6   D7 [P]  Sp  IDLE / Next St
        起始位   ←──── 数据位 5~8 位 (LSB 先行) ────→ 校验  停止位
        =0                               (可选) =1 (1/1.5/2位 高电平)
```

| 组成部分 | 说明 |
|----------|------|
| **空闲状态** | 总线空闲时保持高电平 |
| **起始位** | 1 位，**低电平**，标志一帧数据的开始 |
| **数据位** | 5~8 位可选（常用 8 位），**低位（LSB）先行** |
| **校验位** | 可选 1 位，用于数据完整性验证，分为奇校验/偶校验/无校验 |
| **停止位** | 1 / 1.5 / 2 位，**高电平**，标志一帧结束，也是帧间间隔 |

### 校验位详解

| 校验方式 | 说明 |
|----------|------|
| **无校验（None）** | 不发送校验位，效率最高 |
| **奇校验（Odd）** | 校验位使得：数据位+校验位中 1 的个数为**奇数** |
| **偶校验（Even）** | 校验位使得：数据位+校验位中 1 的个数为**偶数** |

**奇偶校验示例（数据 0x55 = 01010101B，含 4 个 1）：**

- 奇校验：校验位 = 1（使 1 的总数为 5，奇数）
- 偶校验：校验位 = 0（使 1 的总数为 4，偶数）

**注意**：奇偶校验只能检测奇数个位的错误，无法纠错。

---

### 硬件流控（RTS / CTS）

当接收方处理速度跟不上发送方时，若不加以控制，接收缓冲区会溢出导致数据丢失。硬件流控通过额外的 GPIO 线实现"发送许可"机制。

| 信号 | 方向 | 全称 | 作用 |
|------|------|------|------|
| **RTS** | 接收方 → 发送方 | Request To Send | 接收方**拉低**表示"我可以接收数据"；拉高表示"暂停发送" |
| **CTS** | 发送方 ← 接收方 | Clear To Send | 发送方检测到 CTS 低电平时才发送；高电平时停止发送 |

**工作流程：**

```
接收方准备好 ──→ RTS 拉低 ──→ 发送方 CTS 检测到低电平 ──→ 发送方开始发送
接收方缓冲区满 ──→ RTS 拉高 ──→ 发送方 CTS 检测到高电平 ──→ 发送方暂停发送
接收方处理完毕 ──→ RTS 拉低 ──→ 发送方继续发送
```

**连线方式（5 线 UART）：**

```
  发送方                          接收方
  ┌──────┐                      ┌──────┐
  │  TX  ├─────────────────────→│  RX  │  数据发送
  │  RX  │←─────────────────────┤  TX  │  数据接收
  │ CTS  │←─────────────────────┤ RTS  │  流控输入
  │ RTS  ├─────────────────────→│ CTS  │  流控输出
  │ GND  ├─────────────────────→│ GND  │  共地
  └──────┘                      └──────┘
```

> **注意**：RTS/CTS 是交叉连接的——己方 RTS 接对方 CTS，己方 CTS 接对方 RTS。

**硬件流控 vs 软件流控：**

| 对比   | 硬件流控 (RTS/CTS) | 软件流控 (XON/XOFF)               |
| ---- | -------------- | ----------------------------- |
| 实现方式 | 额外 GPIO 引脚     | 发送特殊字符 XON(0x11) / XOFF(0x13) |
| 速度   | 实时响应，无延迟       | 需解析数据帧，有延迟                    |
| 占用   | 多两根线           | 不占引脚                          |
| 适用场景 | 高速大数据量传输       | 低速、引脚受限                       |

**ESP32-S3 配置：**

```c
// 仅启用 CTS（发送方检查 CTS 信号）
.flow_ctrl = UART_HW_FLOWCTRL_CTS,

// 仅启用 RTS（接收方控制 RTS 信号）
.flow_ctrl = UART_HW_FLOWCTRL_RTS,

// 同时启用 RTS 和 CTS
.flow_ctrl = UART_HW_FLOWCTRL_CTS_RTS,

// 禁用（默认）
.flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
```

---

## 四、数据帧分析实例

### 发送数据：0x55（二进制 01010101）

```
参数：
  波特率：9600 bps
  起始位：1 位
  数据位：8 位
  校验位：无
  停止位：1 位
```

### 每位的持续时间

波特率 9600 → 1 bit 时长 = **1/9600 ≈ 104.17 μs**

### 帧时序图（逻辑分析仪视角）

```
           ┌───┐    ┌───┐    ┌───┐    ┌───┐    ┌──────
───────────┘   └────┘   └────┘   └────┘   └────┘
  IDLE    St  D0   D1  D2   D3  D4   D5  D6   D7  Sp  IDLE
  HIGH    =0  =1   =0  =1   =0  =1   =0  =1   =0  =1  HIGH
           ←──────── 每 bit = 1/9600 ≈ 104.17 μs ────────→
```

分析：
- **D0 (LSB)** = 1 → 高电平
- **D1** = 0 → 低电平
- **D2** = 1 → 高电平
- **D3** = 0 → 低电平
- **D4** = 1 → 高电平
- **D5** = 0 → 低电平
- **D6** = 1 → 高电平
- **D7 (MSB)** = 0 → 低电平

波形呈现交替的 1/0 方波：`1 0 1 0 1 0 1 0`（LSB 先行）

### 奇校验示例：发送 0x55 带奇校验

数据位 8 位 + 奇校验位 1 位：

```
           ┌───┐    ┌───┐    ┌───┐    ┌───┐        ┌──────
───────────┘   └────┘   └────┘   └────┘   └────────┘
  IDLE    St  D0   D1  D2   D3  D4   D5  D6   D7  P   Sp  IDLE
  HIGH    =0  =1   =0  =1   =0  =1   =0  =1   =0 =1  =1  HIGH
                             数据 0x55 (4个1)  →  校验位=1
                                         (使总 1 个数 = 5 → 奇数)
```

---

## 五、波特率计算

### 一帧数据的总位数

以 **9600-8-N-1**（9600 波特率，8 数据位，无校验，1 停止位）为例：

| 组成部分 | 位数 |
|----------|------|
| 起始位 | 1 |
| 数据位 | 8 |
| 校验位 | 0（无） |
| 停止位 | 1 |
| **总计** | **10 位** |

### 传输效率

- 实际数据比例：8/10 = **80%**（20% 被帧开销占用）
- 每秒最多帧数：9600 / 10 = **960 帧/秒**
- 每秒有效数据：960 × 8 = **7680 字节/秒** ≈ 7.5 KB/s

### 常用配置对比

| 配置 | 总位数 | 有效数据比 | 有效速率 @ 115200 |
|------|--------|-----------|-------------------|
| 8-N-1 | 10 | 80.0% | 92.16 kbps |
| 8-N-2 | 11 | 72.7% | 83.78 kbps |
| 8-E-1 | 11 | 72.7% | 83.78 kbps |
| 7-E-1 | 10 | 70.0% | 80.64 kbps |

---

## 六、ESP32-S3 UART 驱动代码

本项目采用模块化结构，UART 初始化独立为组件，`main.c` 专注业务逻辑。

### 项目结构

```
15_UART/
├── main/
│   └── main.c              ← 主程序：接收并回显数据
└── components/
    └── UART/
        ├── uart.h          ← 声明 uart_init()
        └── uart.c          ← UART 初始化的实现
```

### uart.c — UART 初始化模块

```c
#include "uart.h"
#include "driver/uart.h"
#include "driver/gpio.h"


void uart_init(void)
{
    uart_config_t uart_structure = {
        .baud_rate = 9600,                        // 波特率 9600 bps
        .data_bits = UART_DATA_8_BITS,            // 数据位 8 位
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,    // 禁用硬件流控 (RTS/CTS)
        .parity    = UART_PARITY_DISABLE,         // 无校验位
        .source_clk = UART_SCLK_DEFAULT,          // 默认时钟源 (APB)
        .stop_bits = UART_STOP_BITS_1,            // 停止位 1 位
    };
    uart_param_config(UART_NUM_1, &uart_structure);           // 端口1, 应用配置
    uart_set_pin(UART_NUM_1, GPIO_NUM_17, GPIO_NUM_18,       // 端口1, TX=17, RX=18
                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);     // RTS/CTS 不使用
    uart_driver_install(UART_NUM_1, 1024, 1024, 0, NULL, 0);  // 端口1, RX缓冲1024B, TX缓冲1024B, 无事件队列
}
```

**说明**：
- 使用 `UART_NUM_1`（ESP32-S3 有 3 个 UART：0、1、2）
- TX→GPIO17，RX→GPIO18，其余引脚不使用
- 收发缓冲区各 1024 字节，不使用事件队列

### main.c — 主程序（串口回显）

```c
#include <stdio.h>
#include <stdint.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/uart.h"      // uart_read_bytes、uart_write_bytes、uart_flush
#include "driver/gpio.h"
#include "uart.h"             // 本项目的 uart_init()

void app_main(void)
{
    int len = 0;
    uint8_t uart_buf[1024] = {0};   // 接收缓冲区，初始化为 0
    uart_init();                     // 调用组件中的初始化函数

    while (1)
    {
        // 从 UART_NUM_1 读取数据，超时 100ms
        len = uart_read_bytes(UART_NUM_1, uart_buf, 1024, pdMS_TO_TICKS(100));
        if (len > 0)
        {
            // 将收到的数据原样发送回去（回显）
            uart_write_bytes(UART_NUM_1, uart_buf, len);
            uart_flush(UART_NUM_1);  // 等待发送完成，清空缓冲区
        }
        vTaskDelay(5);               // 延时 5 ticks，让出 CPU
    }
}
```

**数据流**：

#### 总览图：回显方向

```
PC TX -- ESP32 RX(GPIO18)
PC RX -- ESP32 TX(GPIO17)
GND   -- GND
```

#### 局部图：驱动缓冲区

```
RX 引脚
  |
  v
UART RX FIFO
  | 中断搬运
  v
驱动 RX 环形缓冲
  |
  v
uart_read_bytes()
```

#### 时序图：一次回显

```
PC 发送字节
    |
    v
ESP32 读到 len > 0
    |
    v
uart_write_bytes() 原样写回
    |
    v
PC 收到回显
```

**参数解释**：

| 调用 | 参数 | 含义 |
|------|------|------|
| `uart_read_bytes` | `UART_NUM_1` | 从 UART1 读取 |
| | `uart_buf` | 数据存到这个数组 |
| | `1024` | 最多读 1024 字节 |
| | `pdMS_TO_TICKS(100)` | 等待超时 = 100ms |
| | **返回 `len`** | 实际读取字节数；超时返回 0 |
| `uart_write_bytes` | `UART_NUM_1` | 从 UART1 发送 |
| | `uart_buf` | 发送这个数组的内容 |
| | `len` | 发送 len 个字节（只发实际读到的量） |
| `uart_flush` | `UART_NUM_1` | 清空 UART1 缓冲区（确保发完） |
| `vTaskDelay` | `5` | FreeRTOS 延时 5 ticks，防止空转占满 CPU |

> **运行效果**：用串口工具发送任意数据，ESP32 会将相同数据原样返回，实现"回环测试"。

### 串口接收数据的两种显示模式

`uart_read_bytes` 读到的是原始字节数据（`uint8_t` 数组），显示方式决定了我们看到什么。

#### 文本模式（ASCII）

将每个字节当作 ASCII 字符直接打印。人可直接阅读，适合调试字符串协议（如 AT 指令）。

```c
uint8_t rx_buf[128];
int len = uart_read_bytes(UART_NUM_0, rx_buf, sizeof(rx_buf), pdMS_TO_TICKS(500));
if (len > 0) {
    rx_buf[len] = '\0';                        // 添加字符串结束符
    printf("收到文本: %s\n", (char *)rx_buf);  // 直接按字符串打印
}
```

**示例**：发送 "AT\r\n" → 终端显示 `收到文本: AT`

> **注意**：如果收到的不是可打印字符（如 0x00~0x1F 控制字符），终端会显示乱码或不显示。

#### Hex 模式（十六进制）

将每个字节转换为两位十六进制字符串显示。适合调试二进制协议、传感器原始数据。

```c
uint8_t rx_buf[128];
int len = uart_read_bytes(UART_NUM_0, rx_buf, sizeof(rx_buf), pdMS_TO_TICKS(500));
if (len > 0) {
    printf("收到 Hex (%d 字节): ", len);
    for (int i = 0; i < len; i++) {
        printf("%02X ", rx_buf[i]);            // 每字节显示为 2 位大写 Hex
    }
    printf("\n");
}
```

**示例**：发送 0x55 0xAA 0x01 → 终端显示 `收到 Hex (3 字节): 55 AA 01`

#### 同时显示 Hex + 文本

两种模式结合，左侧 Hex、右侧 ASCII，类似 Wireshark/逻辑分析仪的显示方式：

```c
void uart_dump_hex(const uint8_t *data, int len)
{
    printf("--- Hex Dump (%d bytes) ---\n", len);
    for (int i = 0; i < len; i++) {
        // 换行：每 16 字节一行
        if (i % 16 == 0) printf("%04X: ", i);
        printf("%02X ", data[i]);
        if (i % 16 == 15) {
            printf(" |");                       // 右侧 ASCII 区
            for (int j = i - 15; j <= i; j++)
                printf("%c", (data[j] >= 0x20 && data[j] <= 0x7E) ? data[j] : '.');
            printf("|\n");
        }
    }
    // 最后一行不足 16 字节，补齐空格
    int rem = len % 16;
    if (rem > 0) {
        for (int i = 0; i < (16 - rem) * 3 + 2; i++) printf(" ");
        printf("|");
        for (int j = len - rem; j < len; j++)
            printf("%c", (data[j] >= 0x20 && data[j] <= 0x7E) ? data[j] : '.');
        printf("|\n");
    }
    printf("--- End ---\n");
}
```

**输出示例**（收到 "Hello UART!\r\n"）：

```
--- Hex Dump (13 bytes) ---
0000: 48 65 6C 6C 6F 20 55 41 52 54 21 0D 0A          |Hello UART!..|
--- End ---
```

#### 两种模式对比

| 对比 | 文本模式 | Hex 模式 |
|------|---------|----------|
| 显示方式 | `printf("%s")` | `printf("%02X")` |
| 可读性 | 字符直观可见 | 需对照 ASCII 表 |
| 不可打印字符 | 显示乱码/被吞 | 精确显示每个字节 |
| 适用协议 | AT 指令、JSON、NMEA | Modbus、自定义二进制协议 |
| 数据完整性 | 无法确认全部字节 | 每个字节都可见 |

> **原则**：调试时先用 Hex 模式确认数据的完整内容，确认是文本协议后再切换文本模式阅读。

---

### 其他常用 API

| API 函数 | 功能 | 说明 |
|----------|------|------|
| `uart_get_buffered_data_len` | 获取接收缓冲区中待读取的数据长度 | 用于判断是否有数据到达 |
| `uart_flush` | 清空接收/发送缓冲区 | 清除残留数据 |
| `uart_flush_input` | 仅清空接收缓冲区 | |
| `uart_wait_tx_done` | 等待发送完成 | 等待发送 FIFO 清空 + 移位寄存器发送完毕 |
| `uart_set_baudrate` | 动态修改波特率 | 运行时改变通信速率 |
| `uart_get_baudrate` | 获取当前波特率 | |
| `uart_driver_delete` | 卸载 UART 驱动 | 释放资源 |

## 容易踩坑

| 现象 | 常见原因 | 排查方向 |
|------|----------|----------|
| 接收乱码 | 双方波特率、数据位、校验位或停止位不一致 | 统一 115200 8N1 等配置并确认时钟源 |
| 收不到数据 | TX/RX 未交叉连接或 GND 未共地 | 检查 TX→RX、RX←TX、GND 共地和引脚号 |
| 读取长度总为 0 | 超时时间太短或未安装驱动缓冲区 | 增大 `uart_read_bytes()` timeout，检查 `uart_driver_install()` |
| 二进制数据被截断 | 把原始数据当 C 字符串处理 | 按长度处理缓冲区，使用 Hex 模式查看 |

---

## Tips

### 波特率的本质——时钟同步的替代品

UART 是"异步"通信，没有单独的时钟线。那么接收方怎么知道每个 bit 从哪里开始、在哪里结束？答案就是**波特率**——双方事先约定好同一个速率，接收方用这个速率作为"隐式时钟"。

```
发送方 @ 9600 bps:
  ┌──┐  ┌──┐  ┌──┐
──┘  └──┘  └──┘  └──   每 bit = 1/9600 ≈ 104μs
  104μs

接收方 @ 9600 bps:
  1. 检测到起始位（下降沿）
  2. 等待 104/2 = 52μs（到 bit 中心，最佳采样点）
  3. 每间隔 104μs 采样一次，连续 8 次
  4. 检查停止位是否为高
```

**波特率误差的容忍度**：如果双方时钟偏差过大，到第 8 个数据位时采样点已经偏离到相邻 bit 的区域，就会读错。一般要求双方误差 < 3%~5%。ESP32-S3 使用 APB 时钟源（80 MHz）分频得到波特率，精度远高于外部晶振直接分频。

**为什么 115200 是常用值？** 这是 9600 的 12 倍，可以整除常见晶体频率（如 1.8432 MHz），分频误差为零。自定义波特率（如 1000000）可能导致双方的分频系数不同，误差累积。

### 中断接收 vs 轮询接收——如何选择

本项目 `main.c` 中的 `uart_read_bytes()` 是**轮询方式**——主循环每 5ms 来问一次"有数据吗？"。在数据量小、实时性要求不高时完全够用。

但如果数据量大（如每秒几千字节），轮询就可能丢数据：

```
轮询方式：
  main loop 每 5ms 读一次
  如果 5ms 内来了 512 字节 → 环形缓冲区可能溢出 → 丢数据

中断方式：
  每收到一个字节 → ISR 自动存到环形缓冲区
  main loop 只需从缓冲区读 → 不会丢数据
```

如何在 ESP-IDF 中启用 UART 中断：

```c
// 安装驱动时启用事件队列
QueueHandle_t uart_queue;
uart_driver_install(UART_NUM_1, 1024, 1024, 10, &uart_queue, 0);
//                                  RX  TX 队列深度 ↑

// 在 FreeRTOS 任务中等待事件
void uart_event_task(void *pvParameters) {
    uart_event_t event;
    while (1) {
        if (xQueueReceive(uart_queue, &event, portMAX_DELAY)) {
            switch (event.type) {
                case UART_DATA:
                    // 有数据到达，现在去读
                    uart_read_bytes(UART_NUM_1, buf, event.size, 0);
                    break;
                case UART_FIFO_OVF:
                    // 缓冲区溢出！增大 rx_buffer_size
                    break;
            }
        }
    }
}
```

**选择建议**：
| 场景 | 推荐方式 |
|------|---------|
| 每秒几十字节、偶尔通信 | **轮询**（简单） |
| 每秒几百字节、持续通信 | **中断 + 事件队列** |
| 每秒几千字节以上 | **中断 + DMA**（UART 也支持 DMA） |

### 踩坑实录

1. **波特率对了但数据错了**：检查是否双方数据位/停止位/校验位完全一致。8-N-1 是最通用的配置，任何一方改了校验方式都会导致另一边收到乱码。

2. **printf 和 UART 混用**：ESP32-S3 的 `printf` 默认输出到 `UART_NUM_0`（USB 串口）。如果你在代码中同时用 `printf` 和 `uart_write_bytes(UART_NUM_0, ...)`，两者会竞争同一个 UART 的发送缓冲区。建议 printf 用 UART0（调试日志），自定协议用 UART1/UART2。

3. **vTaskDelay(5) 太小**：本项目的 5 ticks（约 50ms @100Hz）是合理的。但如果设为 `vTaskDelay(1)`，CPU 会几乎 100% 空转轮询 UART，导致其他任务饿死。

### 知识衔接

- **下一章**：[02_IIC](02_IIC.md)——从"异步无时钟"的 UART 过渡到"同步有时钟"的 I2C。UART 用波特率隐式同步，I2C 用 SCL 显式同步，这是两者最本质的区别。
- **高级用法**：UART 的 DMA 模式（`uart_write_bytes` 内部超过阈值会自动使用 DMA）、RS485 方向控制（用 RTS 引脚自动切换收发方向）。
