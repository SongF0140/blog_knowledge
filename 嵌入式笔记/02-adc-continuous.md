---
title: "ESP32-S3 ADC 连续转换（Continuous）——DMA 高速采样"
published: 2026-08-19
description: "ADC 连续转换配置结构体关键字段："
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","ADC","模拟外设"]
category: 物联网
draft: false
---
# ESP32-S3 ADC 连续转换（Continuous）——DMA 高速采样

## 本章重要 API 函数（仅 ADC Continuous 相关）

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `adc_continuous_new_handle()` | 创建 ADC 连续转换句柄 | `const adc_continuous_handle_cfg_t *hdl_config`：句柄配置（帧大小、缓冲区大小）<br>`adc_continuous_handle_t *ret_handle`：返回句柄 | `"esp_adc/adc_continuous.h"` |
| `adc_continuous_config()` | 配置 ADC 连续转换参数 | `adc_continuous_handle_t handle`：句柄<br>`const adc_continuous_config_t *config`：转换配置（通道数组、采样频率、格式） | `"esp_adc/adc_continuous.h"` |
| `adc_continuous_register_event_callbacks()` | 注册转换完成事件回调 | `adc_continuous_handle_t handle`：句柄<br>`const adc_continuous_evt_cbs_t *cbs`：回调结构体（`on_conv_done`）<br>`void *user_data`：用户数据 | `"esp_adc/adc_continuous.h"` |
| `adc_continuous_start()` | 启动 ADC 连续转换 | `adc_continuous_handle_t handle`：句柄 | `"esp_adc/adc_continuous.h"` |
| `adc_continuous_stop()` | 停止 ADC 连续转换 | `adc_continuous_handle_t handle`：句柄 | `"esp_adc/adc_continuous.h"` |
| `adc_init()` | 初始化 ADC 连续转换模式（自定义封装） | `void`：无参数 | `"myadc.h"` |

**ADC 连续转换配置结构体关键字段：**

| 字段 | 说明 |
|------|------|
| `.adc_pattern` | 通道配置数组指针（`adc_digi_pattern_config_t[]`），每个元素含衰减、分辨率、通道号、单元号 |
| `.conv_mode` | 转换模式：`ADC_CONV_SINGLE_UNIT_1`（仅ADC1）/ `ADC_CONV_BOTH_UNIT` |
| `.format` | 数据输出格式：`ADC_DIGI_OUTPUT_FORMAT_TYPE2`（每采样4字节，自描述） |
| `.pattern_num` | 每次扫描的通道数量（本项目=2） |
| `.sample_freq_hz` | 总采样频率（本项目=20000Hz），多通道共享 |

**TYPE2 数据格式（每采样 4 字节）：**

| 字节 | 内容 | 说明 |
|------|------|------|
| Byte 0 | ADC_Result[7:0] | 12bit 结果的低 8 位 |
| Byte 1 | ADC_Result[11:8]（低4bit有效） | 12bit 结果的高 4 位 |
| Byte 2 | Channel_Number | 通道编号 |
| Byte 3 | Unit | ADC 单元编号 |

**12bit 合成公式：** `adc_value = ((data[n+1] & 0x0F) << 8) | data[n]`

**注意：SPI、LCD 相关 API 已在前面章节讲解。**

---

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数（仅 ADC Continuous 相关）](#本章重要-api-函数仅-adc-continuous-相关)**
- **[学习前后依赖](#学习前后依赖)**
- **[一、从 Oneshot 到 Continuous——为什么需要 DMA](#一从-oneshot-到-continuous——为什么需要-dma)**
  - [1.1 速率瓶颈在哪](#11-速率瓶颈在哪)
  - [1.2 DMA 的角色：CPU 的"搬运工"](#12-dma-的角色cpu-的"搬运工")
- **[二、运行链路树](#二运行链路树)**
- **[三、ADC 数据帧解析（TYPE2 格式）](#三adc-数据帧解析type2-格式)**
  - [TYPE1 vs TYPE2 格式对比](#type1-vs-type2-格式对比)
  - [TYPE1 格式（未使用）](#type1-格式未使用)
  - [TYPE2 格式（本项目使用）](#type2-格式本项目使用)
  - [为什么选择 TYPE2](#为什么选择-type2)
  - [双通道不是真正"同时"采样](#双通道不是真正"同时"采样)
  - [1. SAR ADC 硬件限制](#1-sar-adc-硬件限制)
  - [2. 时间复用机制](#2-时间复用机制)
  - [3. 帧的概念](#3-帧的概念)
  - [4. 为什么看起来像"同时"](#4-为什么看起来像"同时")
  - [5. 真正的"同时采样"方案](#5-真正的"同时采样"方案)
- **[四、重点行注释](#四重点行注释)**
  - `[#[myadc.c L12`|`myadc.c L12`]]
  - `[#[myadc.c L15-17`|`myadc.c L15-17`]]
  - `[#[myadc.c L25-26`|`myadc.c L25-26`]]
  - `[#[myadc.c L32-35`|`myadc.c L32-35`]]
  - `[#[myadc.c L38-42`|`myadc.c L38-42`]]
  - `[#[myadc.c L46-50`|`myadc.c L46-50`]]
  - `[#[myadc.c L55`|`myadc.c L55`]]
- **[五、整体架构](#五整体架构)**
- **[六、模块实现说明](#六模块实现说明)**
  - [6.1 MYADC 组件](#61-myadc-组件)
  - [6.2 main 模块](#62-main-模块)
- **[七、文件简单注释](#七文件简单注释)**
  - `[#[myadc.c`|`myadc.c`]]
  - `[#[main.c`|`main.c`]]
- **[容易踩坑](#容易踩坑)**
- **[八、关键要点总结](#八关键要点总结)**
- **[Tips](#tips)**
  - [采样率、帧大小、缓冲区的关系](#采样率帧大小缓冲区的关系)
  - [回调中的线程安全问题](#回调中的线程安全问题)
  - [TYPE1 or TYPE2：选择建议](#type1-or-type2选择建议)
  - [踩坑实录](#踩坑实录)
  - [知识衔接](#知识衔接)

---

## 学习前后依赖

| 方向 | 内容 |
|------|------|
| 前置基础 | ADC 基本概念、模拟电压范围、LCD 显示和中断/DMA 回调安全意识。 |
| 本章核心 | 掌握 ADC Continuous 连续采样、DMA 搬运、TYPE2 数据帧解析和多通道轮转。 |
| 后续承接 | 承接 `13_ADC` 的 Oneshot 单次采样对比，并为高速传感器数据采集打基础。 |

## 一、从 Oneshot 到 Continuous——为什么需要 DMA

### 1.1 速率瓶颈在哪

在 [01_低速ADC](01_低速ADC.md) 中，我们用 Oneshot 模式每 500ms 读一次 ADC，轻松愉快。但如果需要 **20 kHz 采样率**——也就是每秒 20000 次转换——Oneshot 就崩溃了：

```text
Oneshot @ 20 kHz:
  CPU 每秒要调用 adc_oneshot_read() 20000 次
  每次阻塞等待 ~10μs
  总 CPU 占用 ≈ 200ms/s = 20% → 太高了

Continuous @ 20 kHz:
  DMA 每 50μs 搬一次数据（无需 CPU）
  CPU 只在缓冲区满时处理一次回调
  总 CPU 占用 ≈ 可以忽略不计
```

这就是两种模式的本质区别：**Oneshot 的 CPU 开销与采样率成正比，Continuous 的 CPU 开销几乎恒定。**

![Pasted image 20260704144823](https://raw.githubusercontent.com/SongF0140/image_place/main/images/b729a2ba5526397a9ee7c0e211548c51.png)

ESP32-S3 片上集成的 SAR ADC 主要参数：

| 参数 | 说明 |
|------|------|
| 分辨率 | 12 bit（可选 9~12 bit） |
| 量程 | 0 ~ 3.3 V（配合衰减配置） |
| 通道数 | ADC1/ADC2 多通道 |
| 输出格式 | ADC_DIGI_OUTPUT_FORMAT_TYPE1 / TYPE2 |
| 最高采样率 | 约 100 kHz（DMA Continuous 模式下） |

12 bit 分辨率意味着数字量范围是 `0 ~ 4095`，对应电压：

```text
电压 = (ADC值 / 4095) × 3.3V
```

### 1.2 DMA 的角色：CPU 的"搬运工"

DMA（Direct Memory Access，直接存储器访问）是理解 Continuous 模式的关键。它允许外设**不经过 CPU** 直接把数据写入内存：

```text
         没有 DMA:                     有 DMA:
         
    ADC 完成转换                    ADC 完成转换
         │                              │
         ▼                              ▼
    中断 CPU                      DMA 直接写内存
         │                              │
         ▼                              ▼
    CPU 暂停当前任务              去下一轮采样
    CPU 把数据搬到内存                  │
         │                              ▼
         ▼                        缓冲区满 → 回调
    CPU 恢复之前的任务             CPU 批量处理
         
    "每转换一次，CPU 停一次"        "搬完了再叫 CPU"
```

本项目使用 **Continuous 连续转换模式**，ADC1 的通道 3 和通道 4 以 20 kHz 频率循环采样，结果通过 DMA 存入缓冲区，转换完成触发回调。

---

## 二、运行链路树

```text
app_main()
    │
    ├─ lcd_init()           // 初始化 LCD 显示
    │
    └─ adc_init()           // 初始化 ADC 连续转换
            │
            ├─ adc_continuous_new_handle()        // 创建 ADC 连续转换句柄
            │       参数：conv_frame_size = 8      // 每帧 8 字节
            │             max_store_buf_size = 1024 // 环形缓冲区 1024 字节
            │
            ├─ adc_continuous_config()            // 配置 ADC 模式与通道
            │       参数：pattern_num = 2          // 2 个通道
            │             conv_mode = ADC_CONV_SINGLE_UNIT_1  // 仅使用 ADC1
            │             format = ADC_DIGI_OUTPUT_FORMAT_TYPE2
            │             sample_freq_hz = 20000   // 采样频率 20 kHz
            │             adc_pattern[0] = {UNIT_1, CHANNEL_3, 12bit, 11dB}
            │             adc_pattern[1] = {UNIT_1, CHANNEL_4, 12bit, 11dB}
            │
            ├─ adc_continuous_register_event_callbacks()  // 注册事件回调
            │       参数：on_conv_done = continuous_callback
            │
            └─ adc_continuous_start()             // 启动连续转换

                    │
                    ▼ 硬件自动循环采样
        DMA 搬运 conv_frame_size(8字节) 到缓冲区
                    │
                    ▼ 一帧数据准备好
        continuous_callback() 被调用
            │
            ├─ data_value = edata->conv_frame_buffer  // 取帧首地址
            │
            ├─ if (edata->size == 8)
            │       │
            │       ├─ adc_value_rl = ((data_value[1] & 0x0F) << 8) | data_value[0]
            │       │                       // 解析通道 3（RL）的 12 bit ADC 值
            │       │
            │       └─ adc_value_rp = ((data_value[5] & 0x0F) << 8) | data_value[4]
            │                               // 解析通道 4（RP）的 12 bit ADC 值
            │
            └─ return true 表示已处理，缓冲区可释放

app_main() 主循环（500 ms 周期）
    │
    ├─ lcd_show_num(1, 8, adc_value_rl, ...)         // 显示 RL 原始 ADC 值
    ├─ lcd_show_float(2, 8, adc_value_rl/4095*3.3)   // 显示 RL 电压
    ├─ lcd_show_num(3, 8, adc_value_rp, ...)         // 显示 RP 原始 ADC 值
    ├─ lcd_show_float(4, 8, adc_value_rp/4095*3.3)   // 显示 RP 电压
    │
    └─ vTaskDelay(500)                               // 阻塞 500 ms
```

---

## 三、ADC 数据帧解析（TYPE2 格式）

本项目配置为 `ADC_DIGI_OUTPUT_FORMAT_TYPE2`，其数据排布如下（每个采样结果占 4 字节）：

#### 局部图：TYPE2 双通道帧格式

```text
一帧 8 字节 = CH3 采样 + CH4 采样

CH3 / RL：
byte0 -> DATA[7:0]
byte1 -> DATA[11:8]
byte2 -> channel = CH3
byte3 -> unit = ADC1

CH4 / RP：
byte4 -> DATA[7:0]
byte5 -> DATA[11:8]
byte6 -> channel = CH4
byte7 -> unit = ADC1
```

```text
单个 TYPE2 采样 = 4 字节
[data low] -> [data high] -> [channel] -> [unit]
```

帧大小 `conv_frame_size = 8` 正好容纳 2 个通道 × 4 字节 = 8 字节。

12 bit ADC 值的合成公式：

```c
adc_value = ((data_value[n+1] & 0x0F) << 8) | data_value[n]
```

- `data_value[n]`：低 8 位
- `data_value[n+1] & 0x0F`：高 4 位（TYPE2 中 bit11~bit8 放在该字节低 4 位）

因此：

| 变量 | 对应通道 | 字节位置 |
|------|----------|----------|
| `adc_value_rl` | ADC1_CHANNEL_3 | `data_value[0..1]` |
| `adc_value_rp` | ADC1_CHANNEL_4 | `data_value[4..5]` |

### TYPE1 vs TYPE2 格式对比

ESP-IDF 提供两种输出格式，本项目选择 TYPE2：

| 特性 | TYPE1 | TYPE2 |
|------|-------|-------|
| 每采样字节数 | 2 字节 | 4 字节 |
| 通道/单元信息 | 无（需要查表确定） | 每帧自带，自描述 |
| 数据量 | 紧凑 | 冗余但灵活 |
| 适用场景 | 单通道固定采样 | 多通道混合采样 |

#### TYPE1 格式（未使用）

每个采样只占 **2 字节**：

```text
字节索引：  0        1
          ├────────┼────────┤
          │DATA[7:0]│DATA[11:8]│
          │  低8位  │  高4位  │
          └────────┴────────┘
```

需要根据采样顺序查表确定数据来自哪个通道，代码较复杂。

#### TYPE2 格式（本项目使用）

每个采样占 **4 字节**，自带通道号和 ADC 单元信息：

```text
字节索引：  0        1        2        3
          ├────────┼────────┼────────┼────────┤
          │DATA[7:0]│DATA[11:8]│ channel│  unit  │
          │  低8位  │  高4位  │ (通道号)│(ADC单元)│
          └────────┴────────┴────────┴────────┘
```

### 为什么选择 TYPE2

1. **自描述性**：每帧数据自带通道号和 ADC 单元信息，即使采样顺序变化也能正确解析
2. **灵活性**：支持混采 ADC1 和 ADC2 的通道（本项目只用了 ADC1）
3. **调试友好**：可以直接从原始数据判断来自哪个通道，便于调试

![Pasted image 20260714144321](https://raw.githubusercontent.com/SongF0140/image_place/main/images/aedb908d0720c17f7f68111fe65e6d42.png)

如果使用 TYPE1 格式，每个采样只占 2 字节，数据更紧凑，但需要根据采样顺序查表确定通道，代码会更复杂。

### 双通道不是真正"同时"采样

#### 1. SAR ADC 硬件限制

ESP32-S3 的 ADC 是 **SAR（逐次逼近寄存器）型 ADC**，每个 ADC 单元（如 ADC1）内部只有**一个转换核心**：

#### 总览图：ADC 单元内部路径

```text
GPIO4(CH3) ─┐
            ├─> MUX ─> SAR 核心 ─> 数字输出
GPIO5(CH4) ─┘

MUX 负责快速切换通道，SAR 核心一次只转换一个通道。
```

#### 2. 时间复用机制

通过 **MUX（多路选择器）快速切换**，在很短的时间内轮流采样多个通道：

#### 时序图：连续采样轮转

```text
时间轴 ->

|-- CH3 --|-- CH4 --|-- CH3 --|-- CH4 --|
    │         │         │         │
    ▼         ▼         ▼         ▼
  SAR转换   SAR转换   SAR转换   SAR转换
```

#### 状态图：缓冲区满/读取/继续采样

```text
连续采样
  │
  ▼
DMA 填充 Frame Buffer
  │
  ▼
一帧满 8 字节
  │
  ▼
on_conv_done 回调
  │
  ▼
任务解析 TYPE2 数据
  │
  ▼
缓冲区释放/复用 -> 继续采样
```

在本项目配置中：

```c
.pattern_num = 2,                         // 每次循环扫描 2 个通道
.sample_freq_hz = 20000,                  // 总采样率 20 kHz
```

- **总采样率 20 kHz**：ADC1 在 1 秒内完成 20000 次转换
- **两通道共享**：每个通道实际采样率约 **10 kHz**
- **切换时间极短**：MUX 切换和 SAR 转换在硬件层面完成，微秒级

#### 3. 帧的概念

"一帧"就是**所有通道各采样一次**的完整周期：

```text
一帧时间 = 通道1转换时间 + 通道2转换时间 + MUX切换时间

本项目一帧 = CH3采样 + CH4采样 = 2次转换
```

当一帧数据（8 字节）收集完成后，DMA 触发中断，调用 `continuous_callback`，此时回调中拿到的是"一组"数据，看起来像是同时采集的。

#### 4. 为什么看起来像"同时"

虽然物理上是**时序轮询**，但在应用层看来像是同时的，原因：

1. **速度快**：20 kHz 采样率下，两次采样间隔只有约 50 μs（微秒），远小于肉眼可见的时间（毫秒级）
2. **打包交付**：一帧数据同时包含两个通道的结果，应用层拿到的就是"一组"数据
3. **DMA 自动搬运**：不需要 CPU 干预通道切换，CPU 只在帧完成后处理一次

#### 5. 真正的"同时采样"方案

如果需要**真正意义上的同时采样**（两个通道在同一时刻采样），需要：

| 方案 | 说明 |
|------|------|
| 使用 ADC1 + ADC2 | 两个独立的 ADC 单元可以并行采样 |
| 外接多通道 ADC | 通过 I2C/SPI 外接专业的多通道同步采样 ADC |

本项目只用了 `ADC_CONV_SINGLE_UNIT_1`，所以是时间复用方式。

---

## 四、重点行注释

### `myadc.c L12`

```c
data_value = edata->conv_frame_buffer;
```

**作用**：在 `on_conv_done` 回调中，拿到当前已转换完成的一帧数据的缓冲区首地址。

**说明**：`edata->conv_frame_buffer` 是 DMA 填充好的原始字节数组，`edata->size` 表示本帧有效字节数。这里用指针直接引用，避免拷贝。

---

### `myadc.c L15-17`

```c
adc_value_rl = ((data_value[1] & 0x0F) << 8) | data_value[0];
adc_value_rp = ((data_value[5] & 0x0F) << 8) | data_value[4];
return true;
```

**作用**：把 TYPE2 原始字节解析成两个 12 bit ADC 值。

- `adc_value_rl`：ADC1 通道 3 的结果。
- `adc_value_rp`：ADC1 通道 4 的结果。
- `return true`：通知驱动层，应用层已消费该帧，缓冲区可以释放/复用。

---

### `myadc.c L25-26`

```c
.conv_frame_size = 8,
.max_store_buf_size = 1024,
```

| 字段 | 值 | 含义 |
|------|-----|------|
| `conv_frame_size` | 8 | 每次连续转换产生 8 字节数据（2 通道 × 4 字节/通道） |
| `max_store_buf_size` | 1024 | DMA/驱动内部环形缓冲区最大 1024 字节，可缓存约 128 帧 |

---

### `myadc.c L32-35`

```c
.atten = ADC_ATTEN_DB_11,
.bit_width = ADC_BITWIDTH_12,
.channel = ADC_CHANNEL_3,
.unit = ADC_UNIT_1,
```

第一个采样通道配置：

| 字段          | 值                 | 含义                       |
| ----------- | ----------------- | ------------------------ |
| `atten`     | `ADC_ATTEN_DB_11` | 11 dB 衰减，输入量程约 0~3.3 V   |
| `bit_width` | `ADC_BITWIDTH_12` | 12 bit 分辨率（0-4095）       |
| `channel`   | `ADC_CHANNEL_3`   | 使用 ADC1 通道 3（对应 GPIO4）   |
| `unit`      | `ADC_UNIT_1`      | 属于 ADC1（esp32s3只有ADC1可用） |

---

### `myadc.c L38-42`

```c
.atten = ADC_ATTEN_DB_11,
.bit_width = ADC_BITWIDTH_12,
.channel = ADC_CHANNEL_4,
.unit = ADC_UNIT_1,
```

第二个采样通道配置：

| 字段 | 值 | 含义 |
|------|-----|------|
| `channel` | `ADC_CHANNEL_4` | 使用 ADC1 通道 4（对应 GPIO5） |
| 其他 | 同上 | 衰减 11 dB、12 bit、ADC1 |

---

### `myadc.c L46-50`

```c
.adc_pattern = adc_digi_arr,
.conv_mode = ADC_CONV_SINGLE_UNIT_1,
.format = ADC_DIGI_OUTPUT_FORMAT_TYPE2,
.pattern_num = 2,
.sample_freq_hz = 20000,
```

| 字段               | 值                              | 含义                                 |
| ---------------- | ------------------------------ | ---------------------------------- |
| `adc_pattern`    | `adc_digi_arr`                 | 指向通道配置数组                           |
| `conv_mode`      | `ADC_CONV_SINGLE_UNIT_1`       | 只使用 ADC1 进行转换                      |
| `format`         | `ADC_DIGI_OUTPUT_FORMAT_TYPE2` | 使用 TYPE2 数据格式（带通道号/单元信息）           |
| `pattern_num`    | 2                              | 每次循环扫描 2 个通道                       |
| `sample_freq_hz` | 20000                          | ADC 总采样率 20 kHz，两通道共享，即每通道约 10 kHz |

---

### `myadc.c L55`

```c
.on_conv_done = continuous_callback,
```

**作用**：注册"转换完成"事件回调函数。

**说明**：当驱动内部累积到一帧（8 字节）数据时，会在中断/任务上下文调用 `continuous_callback`，把最新数据交给应用层解析。

---

## 五、整体架构

#### 总览图：ADC 到 LCD 显示链路

```text
ADC1 CH3/CH4
   │
   ▼
DMA
   │
   ▼
Frame Buffer
   │
   ▼
continuous_callback()
   │
   ▼
adc_value_rl / adc_value_rp
   │
   ▼
app_main() -> LCD 显示
```

#### 局部图：软件层次

```text
main/main.c
  │ 初始化 LCD、ADC，循环显示数值
  ▼
components/MYADC/myadc.c
  │ 封装 ADC 连续转换和回调解析
  ▼
ESP-IDF adc_continuous
  │ 配置 SAR ADC、DMA、事件回调
  ▼
ESP32-S3 ADC1 + DMA
```

---

## 六、模块实现说明

### 6.1 MYADC 组件

文件：

- `myadc.h`
- `myadc.c`

**头文件职责**：

- 导出两个 16 bit 全局变量：`adc_value_rl`、`adc_value_rp`，供 `main.c` 直接读取。
- 导出 `adc_init()` 初始化函数。

**源文件职责**：

- 创建并配置 `adc_continuous_handle`。
- 配置双通道采样模式、TYPE2 输出格式、20 kHz 采样率。
- 注册 `on_conv_done` 回调，在回调中解析 TYPE2 帧并更新全局变量。

### 6.2 main 模块

文件：

- `main.c`

**职责**：

- 初始化 LCD 和 ADC。
- 在 LCD 上固定显示两行标签：RL、RP 的 ADC 值和电压值。
- 主循环每 500 ms 读取一次 `adc_value_rl` / `adc_value_rp`，更新显示。
- 电压换算公式：

```c
(float)adc_value / 4095 * 3.3
```

---

## 七、文件简单注释

### `myadc.c`

```c
#include "myadc.h"
#include "hal/adc_types.h"
#include "esp_adc/adc_continuous.h"

adc_continuous_handle_t adc_continuous_handle;  // ADC 连续转换句柄
uint8_t *data_value;                              // 当前帧缓冲区指针
uint16_t adc_value_rl;                            // 通道 3 的 ADC 值（RL）
uint16_t adc_value_rp;                            // 通道 4 的 ADC 值（RP）

// 转换完成回调：解析一帧 TYPE2 数据，更新两个全局 ADC 值
bool continuous_callback(adc_continuous_handle_t handle,
                         const adc_continuous_evt_data_t *edata,
                         void *user_data)
{
    data_value = edata->conv_frame_buffer;
    if (edata->size == 8)
    {
        // 组合 12 bit 数据：高 4 位在 data_value[n+1] 低 4 位，低 8 位在 data_value[n]
        adc_value_rl = ((data_value[1] & 0x0F) << 8) | data_value[0];
        adc_value_rp = ((data_value[5] & 0x0F) << 8) | data_value[4];
        return true;  // 已处理该帧
    }
    return false;
}

// 初始化 ADC1 两通道连续转换
void adc_init(void)
{
    // 句柄配置：每帧 8 字节，环形缓冲区 1024 字节
    adc_continuous_handle_cfg_t adc_contunuous_structure = {
        .conv_frame_size = 8,
        .max_store_buf_size = 1024,
    };
    adc_continuous_new_handle(&adc_contunuous_structure, &adc_continuous_handle);

    // 采样通道配置：ADC1 CH3、CH4，12 bit，11 dB 衰减
    adc_digi_pattern_config_t adc_digi_arr[] = {
        { .atten = ADC_ATTEN_DB_11, .bit_width = ADC_BITWIDTH_12,
          .channel = ADC_CHANNEL_3, .unit = ADC_UNIT_1 },
        { .atten = ADC_ATTEN_DB_11, .bit_width = ADC_BITWIDTH_12,
          .channel = ADC_CHANNEL_4, .unit = ADC_UNIT_1 }
    };

    // 连续转换配置：单 ADC1、TYPE2 格式、2 通道、20 kHz
    adc_continuous_config_t continuous_config_structure = {
        .adc_pattern = adc_digi_arr,
        .conv_mode = ADC_CONV_SINGLE_UNIT_1,
        .format = ADC_DIGI_OUTPUT_FORMAT_TYPE2,
        .pattern_num = 2,
        .sample_freq_hz = 20000,
    };
    adc_continuous_config(adc_continuous_handle, &continuous_config_structure);

    // 注册转换完成回调
    adc_continuous_evt_cbs_t evt_structure = {
        .on_conv_done = continuous_callback,
    };
    adc_continuous_register_event_callbacks(adc_continuous_handle,
                                            &evt_structure, NULL);

    // 启动连续转换
    adc_continuous_start(adc_continuous_handle);
}
```

### `main.c`

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "lcd.h"
#include "myadc.h"

void app_main(void)
{
    lcd_init();  // 初始化 LCD 显示
    adc_init();  // 初始化 ADC 连续转换

    // 在 LCD 上显示固定标签
    lcd_show_string(1, 1, "adc_rl:", GREEN, BLACK);
    lcd_show_string(2, 1, "vol_rl:", GREEN, BLACK);
    lcd_show_string(3, 1, "adc_rp:", GREEN, BLACK);
    lcd_show_string(4, 1, "vol_rp:", GREEN, BLACK);

    while (1)
    {
        // 显示 RL 的 ADC 值和换算电压
        lcd_show_num(1, 8, adc_value_rl, 4, YELLOW, BLACK);
        lcd_show_float(2, 8, (float)adc_value_rl / 4095 * 3.3, 3, YELLOW, BLACK);

        // 显示 RP 的 ADC 值和换算电压
        lcd_show_num(3, 8, adc_value_rp, 4, YELLOW, BLACK);
        lcd_show_float(4, 8, (float)adc_value_rp / 4095 * 3.3, 3, YELLOW, BLACK);

        vTaskDelay(500);  // 每 500 ms 刷新一次
    }
}
```

---

## 容易踩坑

| 现象 | 常见原因 | 排查方向 |
|------|----------|----------|
| 采样数据错位 | 未按 TYPE2 帧格式解析或通道号判断错误 | 逐字段解析 unit/channel/data，确认 `conv_frame_size` |
| 回调中数据失效 | DMA 帧缓冲区生命周期理解错误 | 在有效上下文内复制或立即解析数据 |
| 采样丢帧 | 处理速度慢于 DMA 填充速度 | 增大缓冲区、降低采样率或减少回调耗时 |
| 电压换算不准 | 衰减、位宽、校准方式与输入范围不匹配 | 确认 attenuation、bitwidth 和 ADC 校准曲线 |

## 八、关键要点总结

1. **连续 ADC 通过 DMA 自动搬运数据**，CPU 只在回调里解析，适合高速采样。
2. **`conv_frame_size` 必须匹配 `pattern_num × 每通道字节数`**。本例 2 通道 × 4 字节 = 8 字节。
3. **`ADC_DIGI_OUTPUT_FORMAT_TYPE2` 每个采样占 4 字节**，12 bit 有效值跨前两个字节，高 4 位需要 `& 0x0F` 提取。
4. **全局变量 `adc_value_rl/rp` 在回调中更新，主循环读取**，是典型的"生产者-消费者"模型；若对实时性要求高，建议加临界区保护。
5. **电压换算**默认按 3.3 V 满量程、12 bit 计算；实际精度受参考电压、衰减、PCB 噪声影响。

---

## Tips

### 采样率、帧大小、缓冲区的关系

这三者是一套精密的时间-空间联动，设计不当就会丢数据：

```text
采样率 20 kHz → 每秒 20000 次转换
帧大小 8 字节 → 每帧包含 2 通道 × 4 字节
每秒帧数 = 20000 / 2 = 10000 帧/秒
每帧触发一次 on_conv_done 回调 → 每秒 10000 次回调！

缓冲区 1024 字节 → 可缓存 1024/8 = 128 帧
缓冲区填满只需 128/10000 ≈ 12.8 ms
→ 如果回调在 12.8ms 内没有处理完，数据就开始覆盖了
```

**设计准则**：
- `max_store_buf_size` ≥ `conv_frame_size` × 预期回调处理的最大帧数
- 回调中只做最低限度的工作（解析数据、更新全局变量），耗时操作放到任务中
- `sample_freq_hz` 不是越高越好——选择刚好满足需求的频率，给 DMA 和 CPU 留余量

### 回调中的线程安全问题

`continuous_callback` 可能在**中断上下文**中执行（取决于 `intr_alloc_flags` 配置），这意味着：

1. **不能使用 `printf`**——printf 内部可能触发任务调度，在 ISR 中是不允许的
2. **不能使用阻塞 API**——`vTaskDelay`、`xQueueSend`（阻塞版本）全都在 ISR 中无效
3. **全局变量的原子性**——`adc_value_rl` 是 16 bit 变量，ESP32-S3 是 32 位 CPU，单次写入是原子的；但如果扩展为结构体，需要加锁

**本项目正确做法**：回调中只做简单的位操作 + 赋值，`return true` 释放缓冲区。所有显示操作在 `app_main()` 的主循环中完成。

### TYPE1 or TYPE2：选择建议

虽然本项目用 TYPE2（更灵活），但在实际项目中：

| 场景 | 推荐 |
|------|------|
| 单通道大量采集（如音频） | **TYPE1**——每采样 2 字节，缓冲区利用率翻倍 |
| 多通道混合采样 | **TYPE2**——自描述，不会错位 |
| 需要确定数据来源进行调试 | **TYPE2**——直接看 channel/unit 字段 |
| 对缓冲区内存敏感 | **TYPE1**——节省 50% 内存 |

### 踩坑实录

1. **`conv_frame_size` 不对导致数据错位**：这是最常见的问题。TYPE2 下每采样 4 字节，2 通道 = 8 字节/帧。如果配成 `conv_frame_size = 4`（以为 2 个 TYPE1 采样），驱动只会搬 4 字节——结果是每帧只拿到 CH3 的数据，CH4 永远读不到。

2. **采样频率设置过高导致驱动 assert**：ESP-IDF 内部有采样频率上限检查。如果设 `sample_freq_hz = 200000`（200 kHz），可能触发 `assert(sample_freq_hz <= max)`。查看芯片数据手册确定上限。

3. **`return false` 的后果**：回调返回 `false` 表示"我没处理这帧数据"，驱动会**保留该帧不释放**。连续返回 `false` 会导致缓冲区耗尽，后续 DMA 传输全部丢弃。确保正常路径 `return true`。

### 知识衔接

- **上一章**：[01_低速ADC](01_低速ADC.md)——Oneshot 模式下 CPU 主动触发每次转换。建议先理解 SAR ADC 的基本原理和衰减器概念。
- **下一站**：I2C 通信（BH1750 光传感器）——一个"自带 ADC 的数字传感器"如何通过总线输出光照值。从"单片机 ADC 读模拟电压"过渡到"传感器内部集成 ADC，我只管读数字总线"。
- **高级扩展**：用 Continuous ADC 采集音频数据 → 存入环形缓冲区 → 配合 I2S 或 DAC 输出，构成一个简单的音频处理链路。
