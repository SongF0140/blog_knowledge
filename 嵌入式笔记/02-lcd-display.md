---
title: "ESP32-S3 LCD 显示学习笔记"
published: 2026-08-14
description: "注意：SPI 相关 API 将在后续章节详细讲解，本章节仅关注 LCD 显示调用。"
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","中断","定时器","LCD"]
category: 物联网
draft: false
---
# ESP32-S3 LCD 显示学习笔记

## 本章重要 API 函数（仅 LCD 相关）

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `lcd_init()` | LCD 初始化（包含 SPI 初始化） | `void`：无参数 | `"lcd.h"` |
| `lcd_write_cmd()` | 发送 LCD 命令 | `uint8_t cmd`：命令字节 | `"lcd.h"` |
| `lcd_write_data()` | 发送单字节数据 | `uint8_t data`：数据字节 | `"lcd.h"` |
| `lcd_write_data16()` | 发送 16 位像素数据（RGB565） | `uint16_t data`：RGB565 颜色值 | `"lcd.h"` |
| `lcd_write_datan()` | 批量发送数据 | `uint8_t *data`：数据指针<br>`uint16_t length`：数据长度 | `"lcd.h"` |
| `lcd_set_window()` | 设置显示区域窗口 | `uint16_t xstar, ystar`：起始坐标<br>`uint16_t xend, yend`：结束坐标 | `"lcd.h"` |
| `lcd_clear()` | 清屏（填充指定颜色） | `uint16_t color`：RGB565 颜色值 | `"lcd.h"` |
| `lcd_show_char()` | 显示单个字符 | `uint8_t line`：行号<br>`uint8_t column`：列号<br>`uint8_t chr`：字符<br>`uint16_t fontcolor`：字体颜色<br>`uint16_t backgroundcolor`：背景色 | `"lcd.h"` |
| `lcd_show_string()` | 显示字符串 | `uint8_t line`：行号<br>`uint8_t column`：列号<br>`char *string`：字符串指针<br>`uint16_t fontcolor`：字体颜色<br>`uint16_t backgroundcolor`：背景色 | `"lcd.h"` |
| `lcd_show_num()` | 显示数字 | `uint8_t line`：行号<br>`uint8_t column`：列号<br>`uint32_t number`：数字<br>`uint8_t length`：显示位数<br>`uint16_t fontcolor`：字体颜色<br>`uint16_t backgroundcolor`：背景色 | `"lcd.h"` |
| `lcd_show_hexnum()` | 显示十六进制数 | `uint8_t line`：行号<br>`uint8_t column`：列号<br>`uint32_t number`：数字<br>`uint8_t length`：显示位数<br>`uint16_t fontcolor`：字体颜色<br>`uint16_t backgroundcolor`：背景色 | `"lcd.h"` |
| `lcd_show_float()` | 显示浮点数 | `uint8_t line`：行号<br>`uint8_t column`：列号<br>`float number`：浮点数<br>`uint8_t length`：显示位数<br>`uint16_t fontcolor`：字体颜色<br>`uint16_t backgroundcolor`：背景色 | `"lcd.h"` |
| `lcd_show_picture()` | 显示图片 | `uint8_t *img`：图片数据指针（RGB565 格式） | `"lcd.h"` |
| `lcd_hard_reset()` | LCD 硬件复位 | `void`：无参数 | `"lcd.h"` |
| `LCD_DC()` | 设置数据/命令模式（宏） | `x`：0=命令模式，1=数据模式 | `"lcd.h"` |
| `LCD_RST()` | 控制 LCD 复位引脚（宏） | `x`：0=复位，1=正常 | `"lcd.h"` |
| `LCD_BLK()` | 控制 LCD 背光（宏） | `x`：0=关闭，1=开启 | `"lcd.h"` |

**注意：SPI 相关 API 将在后续章节详细讲解，本章节仅关注 LCD 显示调用。**

---

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数（仅 LCD 相关）](#本章重要-api-函数仅-lcd-相关)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. SPI 通信原理](#2-spi-通信原理)**
  - [2.1 SPI 总线架构](#21-spi-总线架构)
  - [2.2 SPI 时序图](#22-spi-时序图)
- **[3. LCD 硬件原理](#3-lcd-硬件原理)**
  - [3.1 LCD 控制信号](#31-lcd-控制信号)
  - [3.2 LCD 控制器（ST7789）](#32-lcd-控制器st7789)
- **[4. 软件架构传递树](#4-软件架构传递树)**
  - [4.1 模块层次结构](#41-模块层次结构)
  - [4.2 LCD 显示函数调用树](#42-lcd-显示函数调用树)
- **[5. 代码详细解析](#5-代码详细解析)**
  - [5.1 SPI 初始化 - spi2_init()](#51-spi-初始化---spi2_init)
  - [5.2 SPI 数据写入 - spi2_write_data()](#52-spi-数据写入---spi2_write_data)
  - [5.3 LCD 基础函数](#53-lcd-基础函数)
  - [5.4 设置显示窗口 - lcd_set_window()](#54-设置显示窗口---lcd_set_window)
  - [5.5 显示字符 - lcd_show_char()](#55-显示字符---lcd_show_char)
- **[6. LCD 初始化序列](#6-lcd-初始化序列)**
  - [6.1 初始化命令序列](#61-初始化命令序列)
  - [6.2 初始化流程图](#62-初始化流程图)
- **[7. 显示函数详解](#7-显示函数详解)**
  - [7.1 显示字符串 - lcd_show_string()](#71-显示字符串---lcd_show_string)
  - [7.2 显示数字 - lcd_show_num()](#72-显示数字---lcd_show_num)
- **[容易踩坑](#容易踩坑)**
- **[8. 学习要点总结](#8-学习要点总结)**
  - [8.1 核心知识点](#81-核心知识点)
  - [8.2 关键 API](#82-关键-api)
  - [8.3 进阶方向](#83-进阶方向)
- **[Tips](#tips)**

---

## 学习前后依赖

| 方向 | 内容 |
|------|------|
| 前置基础 | GPIO 控制信号、SPI 最小背景、组件化调用和 LCD 复位/背光基础。 |
| 本章核心 | 掌握 LCD 初始化、命令/数据区分、窗口设置、字符/数字显示和 RGB565 像素写入。 |
| 后续承接 | 承接 `08_GPTimer`、`12_ADCContinuous`、`13_ADC` 等章节的数据可视化显示。 |

## 1. 概述

本章节学习 SPI 通信和 LCD 驱动，实现文字、数字、图片显示功能。涉及 SPI 总线配置、LCD 初始化序列、像素绘制算法等核心知识点。

本项目的 LCD 模块是学习过程中最重要的调试窗口——后续几乎所有章节（定时器计数、ADC 采样值、PWM 参数）都需要通过 LCD 来做可视化反馈。掌握本章的 LCD 驱动是后续学习的基础。

---

## 2. SPI 通信原理

### 2.1 SPI 总线架构

SPI（Serial Peripheral Interface）是同步串行通信协议，由 Motorola 提出。它的核心设计思路是「主控选通 + 时钟同步」，避免 UART 异步通信需要双方约定波特率的麻烦。

```
SPI（Serial Peripheral Interface）四线通信：

┌─────────────────────────────────────────────────────────┐
│                   SPI 总线信号                           │
├──────────────────┬──────────────────────────────────────┤
│ SCLK (时钟线)     │ 主机产生时钟信号，同步数据传输       │
│ MOSI (数据输出)   │ 主机输出 → 从机输入                 │
│ MISO (数据输入)   │ 从机输出 → 主机输入                 │
│ CS/SS (片选)      │ 选择通信的从机设备                  │
└──────────────────┴──────────────────────────────────────┘

ESP32-S3 SPI2 引脚配置：
- SCLK: GPIO_NUM_12
- MOSI: GPIO_NUM_11
- MISO: GPIO_NUM_13
- CS:   GPIO_NUM_48（LCD）
```

### 2.2 SPI 时序图

SPI 有四种工作模式（Mode 0~3），由时钟极性（CPOL）和时钟相位（CPHA）组合决定。本项目采用 Mode 0（CPOL=0, CPHA=0），这是 ST7789 LCD 控制器支持的模式：

```
SPI Mode 0 时序（本项目采用）：

SCLK:  ───┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌───
          └─┘┘┘┘┘┘┘┘┘┘┘┘┘┘┘
          │←时钟周期→│

MOSI:  ────┬──D7──┬──D6──┬──D5──┬──...
          │      │      │      │
          └──────┘      └──────┘

CS:    ───┐                    ┌───
          │                    │
          └────────────────────┘
          │← 传输期间保持低电平 →│

特点：
- 时钟空闲低电平
- 上升沿采样数据
- 下降沿输出数据
```

---

## 3. LCD 硬件原理

### 3.1 LCD 控制信号

```
LCD 模块控制引脚：

┌─────────────────────────────────────────────┐
│            LCD 控制信号                      │
├─────────────────┬───────────────────────────┤
│ CS (片选)       │ GPIO_NUM_48               │
│                 │ 选择 LCD 进行通信          │
├─────────────────┼───────────────────────────┤
│ DC/WR (数据/命令)│ GPIO_NUM_47              │
│                 │ 0: 发送命令               │
│                 │ 1: 发送数据               │
├─────────────────┼───────────────────────────┤
│ RST (复位)      │ GPIO_NUM_21               │
│                 │ 硬件复位 LCD              │
├─────────────────┼───────────────────────────┤
│ BL (背光)       │ GPIO_NUM_40               │
│                 │ 控制背光亮度              │
└─────────────────┴───────────────────────────┘
```

### 3.2 LCD 控制器（ST7789）

```
ST7789 LCD 控制器特性：

- 分辨率: 240x240 像素
- 颜色深度: 16-bit RGB565
- 接口: SPI
- 内置显存（GRAM）
- 支持部分区域更新

RGB565 颜色格式：
┌─────────────────────────────────────────────┐
│ Bit: 15 14 13 12 11 | 10 9 8 7 6 | 5 4 3 2 1 0 │
│      │←─ R (5bit) ─→│ │←─ G (6bit) ─→│ │←─ B (5bit) ─→│
└─────────────────────────────────────────────┘

颜色计算：
- RED = 0xF800 (11111 000000 00000)
- GREEN = 0x07E0 (00000 111111 00000)
- BLUE = 0x001F (00000 000000 11111)
```

RGB565 格式中绿色占 6 位而非 5 位，是因为人眼对绿色亮度变化最敏感，给绿色多 1 位精度能让颜色过渡更平滑。

---

## 4. 软件架构传递树

### 4.1 模块层次结构

```
┌─────────────────────────────────────────────┐
│                  app_main()                 │
│              (main/main.c)                  │
└───────────────────┬─────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│                 lcd_init()                  │
│           (components/LCD/lcd.c)            │
└───────────────────┬─────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│  spi2_init()  │       │ GPIO 控制引脚 │
│ (components/  │       │ DC/RST/BL     │
│   SPI/spi.c)  │       │               │
└───────┬───────┘       └───────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│           ESP-IDF SPI 驱动                  │
│    spi_bus_initialize()                     │
│    spi_bus_add_device()                     │
└─────────────────────────────────────────────┘
```

### 4.2 LCD 显示函数调用树

```
lcd_show_char()      显示单个字符
    │
    ├── lcd_set_window()  设置显示区域
    │       │
    │       ├── lcd_write_cmd(0x2A)  列地址命令
    │       ├── lcd_write_data16()   列地址数据
    │       ├── lcd_write_cmd(0x2B)  行地址命令
    │       └── lcd_write_data16()   行地址数据
    │
    └── lcd_write_data16()  写入像素颜色
            │
            ├── LCD_DC(1)  设置数据模式
            └── spi2_write_data()  SPI 发送

lcd_show_string()    显示字符串
    │
    └── 循环调用 lcd_show_char()

lcd_show_num()       显示数字
    │
    ├── lcd_pow()    计算幂次
    └── lcd_show_char() 逐位显示

lcd_clear()          清屏
    │
    ├── lcd_set_window(0,0,239,239)  全屏区域
    └── lcd_write_datan()  批量写入背景色
```

---

## 5. 代码详细解析

### 5.1 SPI 初始化 - spi2_init()

**源代码（`spi.c`）：**

```c
void spi2_init(void)
{
    spi_bus_config_t  spibus_structure = {
        .flags = SPICOMMON_BUSFLAG_MASTER,     // 主机模式标志
        .isr_cpu_id = INTR_CPU_ID_AUTO,        // 中断 CPU 自动分配
        .max_transfer_sz = 240 * 240 * 2,      // 最大传输大小（像素数据）
        .miso_io_num = GPIO_NUM_13,            // MISO 引脚
        .mosi_io_num = GPIO_NUM_11,            // MOSI 引脚
        .sclk_io_num = GPIO_NUM_12,            // SCLK 引脚
        .quadhd_io_num = -1,                   // 不使用 Quad SPI
        .quadwp_io_num = -1,                   // 不使用 Quad SPI
    };
    spi_bus_initialize(SPI2_HOST,              // SPI2 主机
                       &spibus_structure,
                       SPI_DMA_CH_AUTO);       // DMA 通道自动分配
}
```

**SPI 配置参数说明：**

| 参数 | 值 | 说明 |
|------|-----|------|
| `max_transfer_sz` | 240*240*2 | 全屏像素数据大小（57600字节） |
| `SPI2_HOST` | - | ESP32-S3 的 SPI2 外设 |
| `SPI_DMA_CH_AUTO` | - | 自动分配 DMA 通道 |

### 5.2 SPI 数据写入 - spi2_write_data()

**源代码（`spi.c`）：**

```c
void spi2_write_data(uint8_t *data, int len)
{
    spi_transaction_t t = {0};
    
    t.length = len * 8;                        // 数据长度（位）
    t.tx_buffer = data;                        // 发送缓冲区指针
    spi_device_polling_transmit(spi2_handle, &t); // 轮询传输
}
```

### 5.3 LCD 基础函数

**写命令（`lcd.c`）：**

```c
void lcd_write_cmd(uint8_t cmd)
{
    LCD_DC(0);                 // DC=0: 命令模式
    spi2_write_data(&cmd,1);   // 发送1字节命令
}
```

**写数据（`lcd.c`）：**

```c
void lcd_write_data(uint8_t data)
{
    LCD_DC(1);                 // DC=1: 数据模式
    spi2_write_data(&data,1);  // 发送1字节数据
}
```

**写16位数据（`lcd.c`）：**

```c
void lcd_write_data16(uint16_t data)
{
    uint8_t databuf[2] = {0,0};
    databuf[0] = data >> 8 ;   // 高字节
    databuf[1] = data & 0xFF ; // 低字节
    LCD_DC(1);
    spi2_write_data(databuf,2);
}
```

### 5.4 设置显示窗口 - lcd_set_window()

**源代码（`lcd.c`）：**

```c
void lcd_set_window(uint16_t xstar, uint16_t ystar,uint16_t xend,uint16_t yend)
{
    lcd_write_cmd(0x2a);           // 列地址设置命令
    lcd_write_data16(xstar);       // 起始列
    lcd_write_data16(xend);        // 结束列
    lcd_write_cmd(0x2b);           // 行地址设置命令
    lcd_write_data16(ystar);       // 起始行
    lcd_write_data16(yend);        // 结束行
    lcd_write_cmd(0x2c);           // 写内存命令
}
```

**窗口设置示意：**

```
LCD 显示区域 (240x240)：

     0              xstar            xend            239
     │                │                │              │
  0  ├────────────────┼────────────────┼──────────────┤
     │                │                │              │
ystar├────────────────┼────────────────┼──────────────┤
     │                │┌───────────────┐│              │
     │                ││               ││              │
     │                ││  显示窗口     ││              │
     │                ││               ││              │
yend ├────────────────┼───────────────┼──────────────┤
     │                │                │              │
239  ├────────────────┼────────────────┼──────────────┤

设置窗口后，写入的数据仅显示在该区域
```

### 5.5 显示字符 - lcd_show_char()

**源代码（`lcd.c`）：**

```c
void lcd_show_char(uint8_t line,uint8_t column,uint8_t chr,uint16_t fontcolor,uint16_t backgroundcolor)
{
    uint8_t i , j = 0 ;
    uint8_t chr_index = 0 ;
    uint8_t chr_temp = 0 ;
    
    // 设置字符显示区域（16x32 像素）
    lcd_set_window( (column - 1) * 16 , (line - 1) * 32 + 8 ,column * 16 - 1 , line * 32 + 7 );
    
    // 遍历字模数据（64字节，每字节8位）
    for( i = 0 ; i < 64 ; i++ )
    {
        chr_temp = ascii_3216[chr - ' '][i];  // 取字模数据
        
        for( j = 0 ; j < 8 ; j++ )
        {
            if( chr_temp & ( 0x01 << j ) )    // 位为1：显示字体颜色
            {
                lcd_write_data16(fontcolor);
            }
            else                               // 位为0：显示背景颜色
            {
                lcd_write_data16(backgroundcolor);
            }
            chr_index++;
            if( chr_index == 16 )              // 每行16像素，换行
            {
                chr_index = 0 ;
                break;
            }
        }
    }
}
```

**字模数据结构：**

```
ASCII 字符 'A' 的字模（32x16 点阵）：

每个字符占 64 字节（32行 × 16列 / 8位）

┌─────────────────┐
│     ██████      │  ← 字模数据按位表示像素
│    ██    ██     │    1: 字体颜色
│    ██    ██     │    0: 背景颜色
│    ████████     │
│    ██    ██     │
│    ██    ██     │
│    ██    ██     │
└─────────────────┘

存储格式：每行2字节（16位）
共32行 × 2字节 = 64字节
```

---

## 6. LCD 初始化序列

### 6.1 初始化命令序列

```c
// LCD 初始化序列（ST7789 控制器）

lcd_write_cmd(0x11);     // Sleep Out（退出睡眠）
vTaskDelay(120);         // 必须延时

lcd_write_cmd(0xB2);     // Porch Setting（门控设置）
lcd_write_data(0x0C);    // Back porch
lcd_write_data(0x0C);    // Front porch
...

lcd_write_cmd(0x36);     // Memory Data Access Control（显示方向）
lcd_write_data(0x70);    // 横屏/竖屏设置

lcd_write_cmd(0x3A);     // Color Format（颜色格式）
lcd_write_data(0x05);    // 16-bit RGB565

lcd_write_cmd(0x21);     // Display Inversion On（反转显示）
lcd_write_cmd(0x29);     // Display On（开启显示）
lcd_write_cmd(0x2C);     // Memory Write（开始写显存）
```

### 6.2 初始化流程图

```
lcd_init() 执行流程：

┌─────────────────┐
│  spi2_init()    │ ◄── SPI 总线初始化
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ 添加 SPI 设备   │ ◄── spi_bus_add_device()
│ (CS=GPIO48)     │
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ 配置 GPIO 控制  │ ◄── DC, BL, RST 引脚
│ (GPIO 47,40,21) │
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ lcd_hard_reset()│ ◄── 硬件复位
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│   lcd_on()      │ ◄── 开启背光
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│ 发送初始化命令  │ ◄── 配置显示参数
│ (0x11,0xB2...)  │
└───────┬─────────┘
        │
        ▼
┌─────────────────┐
│   lcd_clear()   │ ◄── 清屏
└─────────────────┘
```

---

## 7. 显示函数详解

### 7.1 显示字符串 - lcd_show_string()

**源代码（`lcd.c`）：**

```c
void lcd_show_string(uint8_t line,uint8_t column,char *string,uint16_t fontcolor,uint16_t backgroundcolor)
{
    uint8_t i = 0 ;
    for( i = 0 ; string[i] != '\0' ; i++ )
    {
        lcd_show_char( line , column + i , string[i] , fontcolor , backgroundcolor);
    }
}
```

### 7.2 显示数字 - lcd_show_num()

**源代码（`lcd.c`）：**

```c
void lcd_show_num(uint8_t line,uint8_t column,uint32_t number,uint8_t length,uint16_t fontcolor,uint16_t backgroundcolor)
{
    uint8_t i;
    for (i = 0; i < length; i++)
    {
        // 取每一位数字并显示
        lcd_show_char(line, column + i, number / lcd_pow(10, length - i - 1) % 10 + '0',fontcolor,backgroundcolor);
    }
}
```

**数字分解算法：**

```
显示数字 12345678（length=8）：

i=0: 12345678 / 10^7 % 10 = 1 → 显示 '1'
i=1: 12345678 / 10^6 % 10 = 2 → 显示 '2'
i=2: 12345678 / 10^5 % 10 = 3 → 显示 '3'
...
i=7: 12345678 / 10^0 % 10 = 8 → 显示 '8'

lcd_pow(x, y) = x^y（幂函数）
```

---

## 容易踩坑

| 现象 | 常见原因 | 排查方向 |
|------|----------|----------|
| LCD 白屏/黑屏 | 复位、背光或初始化命令顺序不对 | 检查 RST/BL/DC/CS 引脚和 ST7789 初始化延时 |
| 颜色显示异常 | RGB565 高低字节顺序或颜色常量错误 | 确认 `lcd_write_data16()` 先高字节后低字节 |
| 画面偏移 | 窗口坐标、屏幕方向或 MADCTL 参数不匹配 | 核对 `lcd_set_window()` 和控制器方向配置 |
| 大图刷新很慢 | 逐点轮询发送导致传输开销大 | 批量写入像素数据，确认 DMA 缓冲区生命周期 |

## 8. 学习要点总结

### 8.1 核心知识点

| 知识点 | 说明 |
|--------|------|
| SPI 通信 | 四线同步串行通信 |
| DMA 传输 | 大数据量高效传输 |
| LCD 命令序列 | 控制器初始化配置 |
| 窗口设置 | 局部区域更新 |
| 字模显示 | 位图转像素颜色 |

### 8.2 关键 API

| API | 功能 |
|-----|------|
| `spi_bus_initialize()` | 初始化 SPI 总线 |
| `spi_bus_add_device()` | 添加 SPI 设备 |
| `spi_device_polling_transmit()` | 轮询传输数据 |
| `lcd_write_cmd()` | 发送 LCD 命令 |
| `lcd_write_data16()` | 发送 16位像素数据 |
| `lcd_set_window()` | 设置显示区域 |

### 8.3 进阶方向

- 显示图片（lcd_show_picture）
- 触摸屏集成
- 动画/视频显示
- 多种字体支持

---

## Tips

### 1. 本章最易忽视的细节

- **SPI 数据大小端**：RGB565 数据发送时必须先发高字节再发低字节（大端序），因为 ST7789 期望这种字节顺序。如果搞反了，红蓝通道会互换，颜色完全错误。
- **初始化延时不能省略**：ST7789 在收到 Sleep Out 命令（0x11）后需要等待至少 120ms 让内部振荡器稳定。这个延时不是「建议」而是硬性要求——跳过延时会导致后续命令被忽略，屏幕一直白屏。
- **CS（片选）由驱动自动管理**：`spi_bus_add_device()` 中配置的 CS 引脚由 SPI 驱动自动拉低/拉高，不需要手动控制。但 DC（数据/命令）信号需要软件手动切换。
- **窗口设置提升刷新效率**：全屏刷新 240×240 像素需要传输 115,200 字节。如果只更新一个 16×32 的字符区域，只需 1,024 字节——窗口设置能将传输量减少 100 倍以上。

### 2. 踩坑记录

| 场景 | 现象 | 原因 | 教训 |
|------|------|------|------|
| LCD 显示花屏/条纹 | 初始化后画面异常 | `spi_bus_add_device` 的 clock_speed_hz 设为过高（>40MHz），信号质量差 | SPI 频率不要超过模块手册规定值，通常 20~40MHz 稳定 |
| 特定颜色显示不对 | 红色变成蓝色，蓝色变成红色 | RGB565 字节序颠倒 | 对照 ST7789 数据手册确认字节序 |
| 长时间运行后花屏 | 运行几小时后画面抖动 | SPI 信号线过长或未做阻抗匹配 | 缩短杜邦线、降低 SPI 频率 |

### 3. 与后续章节的知识衔接

- **GPTimer/ESP Timer（08/09）**：定时器回调中需要显示计数器值，直接复用本章的 `lcd_show_num()` 即可
- **ADC 章节（12/13）**：采样值需要通过 LCD 实时显示，本质是定时刷新 + 数字显示的组合
- **PWM 舵机控制（11）**：舵机角度显示在 LCD 上，形成「按键→角度计算→PWM 输出→LCD 反馈」的闭环

### 4. 设计决策思考

**为什么通过 SPI 而不是并行接口驱动 LCD？**

ST7789 同时支持 SPI（3 线/4 线）和 8080 并行接口。选择 SPI 的原因：
- **引脚数量**：SPI 最少只需 4 根线（SCLK+MOSI+CS+DC），并行接口需要 8/16 根数据线 + 控制线
- **刷新率够用**：SPI 在 40MHz 下传输 115KB 全屏数据约需 30ms，即约 33 fps，对学习项目完全足够
- **简化布线**：杜邦线连接时 4 根线比 18 根线可靠得多

如果需要高于 60fps 的刷新率（如视频播放），才应考虑切换到并行接口或 QSPI（Quad SPI，一次传 4 位数据）。
