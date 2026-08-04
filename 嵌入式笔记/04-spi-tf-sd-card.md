---
title: "ESP32-S3 SPI 驱动 TF 卡与 FATFS 文件系统"
published: 2026-08-23
description: "SD 卡和 TF 卡本质上是同一种东西的两个外观版本："
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","通信协议","SPI","TF卡","SD卡"]
category: 物联网
draft: false
---
# ESP32-S3 SPI 驱动 TF 卡与 FATFS 文件系统

## 本章重要 API 函数

### ESP-IDF SDSPI + FATFS API

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `SDSPI_HOST_DEFAULT()` | 生成默认 SD SPI 主机配置 | `void`：无参数，返回 `sdmmc_host_t` 结构体，含 `do_transaction`、`set_card_clk` 等函数指针 | `"driver/sdspi_host.h"` |
| `SDSPI_DEVICE_CONFIG_DEFAULT()` | 生成默认 SD SPI Slot 配置 | `void`：无参数，返回 `sdspi_device_config_t` 结构体，默认 CS=GPIO10、host_id=SPI2_HOST | `"driver/sdspi_host.h"` |
| `esp_vfs_fat_sdspi_mount()` | 五合一：初始化 SPI 总线 + 识别 SD 卡 + 注册 VFS + 挂载 FAT 文件系统 | `const char *base_path`：挂载点路径（如 `"/sdcard"`）；<br>`const sdmmc_host_t *host_config`：从 `SDSPI_HOST_DEFAULT()` 获取；<br>`const sdspi_device_config_t *slot_config`：从 `SDSPI_DEVICE_CONFIG_DEFAULT()` 获取；<br>`const esp_vfs_fat_mount_config_t *mount_config`：FAT 挂载参数；<br>`sdmmc_card_t **out_card`：输出 SD 卡信息句柄。<br>返回 `esp_err_t`：`ESP_OK` 成功 | `"esp_vfs_fat.h"` |
| `esp_vfs_fat_sdcard_unmount()` | 卸载 SD 卡并释放 SPI 总线 | `const char *base_path`：挂载点路径；<br>`sdmmc_card_t *card`：卡句柄。<br>返回 `esp_err_t` | `"esp_vfs_fat.h"` |
| `sdmmc_card_print_info()` | 通过串口打印 SD 卡信息（容量、速度等级等） | `FILE *stream`：输出流（如 `stdout`）；<br>`const sdmmc_card_t *card`：卡句柄 | `"driver/sdmmc_types.h"` |

### esp_vfs_fat_mount_config_t 关键字段

| 字段 | 类型 | 含义 |
|------|------|------|
| `format_if_mount_failed` | `bool` | 挂载失败时是否自动格式化（首次使用设 `true`） |
| `max_files` | `int` | 最大同时打开文件数（含目录遍历句柄，建议 ≥ 5） |
| `allocation_unit_size` | `size_t` | 簇大小，填 `0` 则自动选择 |
| `disk_status_check_enable` | `bool` | 是否启用磁盘状态轮询（省 CPU 可设 `false`） |

### 项目封装 API（`spi_sdcard.c`）

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `sd_spi_init()` | 初始化 SD 卡并挂载到 `/sdcard` | `void`：无参数，返回 `esp_err_t`（`ESP_OK` 成功） | `"spi_sdcard.h"` |

### 本章复用 SPI 封装 API（来自 `spi.c`）

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `spi2_init()` | 初始化 SPI2 总线（60 MHz，Mode 0，DMA 自动） | `void`：无参数 | `"spi.h"` |
| `spi2_write_data()` | 通过 SPI2 批量写数据（轮询模式） | `uint8_t *data`：数据缓冲；`int len`：字节数 | `"spi.h"` |
| `spi2_transfer_byte()` | SPI2 全双工收发单字节 | `uint8_t data`：待发字节，返回 `uint8_t`：接收字节 | `"spi.h"` |

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[学习前后依赖](#学习前后依赖)**
- **[一、概述](#一概述)**
  - [1.1 从 SD 卡到 TF 卡](#11-从-sd-卡到-tf-卡)
  - [1.2 为什么用 SPI 而非 SDIO？](#12-为什么用-spi-而非-sdio)
- **[二、TF 卡数据存储与容量体系](#二tf-卡数据存储与容量体系)**
  - [2.1 NAND Flash 内部结构](#21-nand-flash-内部结构)
  - [2.2 三大容量规格：SDSC／SDHC／SDXC](#22-三大容量规格sdsc／sdhc／sdxc)
  - [2.3 顺序 vs 随机读写](#23-顺序-vs-随机读写)
- **[三、文件系统与 FATFS](#三文件系统与-fatfs)**
  - [3.1 为什么需要文件系统？](#31-为什么需要文件系统)
  - [3.2 FAT 文件系统结构](#32-fat-文件系统结构)
  - [3.3 FATFS 是什么？](#33-fatfs-是什么)
- **[四、FATFS 与 C 标准库 I／O 的桥梁：VFS](#四fatfs-与-c-标准库-i／o-的桥梁vfs)**
  - [4.1 传统裸机读 SD 卡 vs VFS 模式](#41-传统裸机读-sd-卡-vs-vfs-模式)
  - [4.2 ESP-IDF VFS 注册流程](#42-esp-idf-vfs-注册流程)
- **[五、TF 卡与 ESP32-S3 的 SPI 联动](#五tf-卡与-esp32-s3-的-spi-联动)**
  - [5.1 硬件连接总览](#51-硬件连接总览)
  - [5.2 SPI 模式下的 SD 命令](#52-spi-模式下的-sd-命令)
  - [5.3 SD 卡初始化流程](#53-sd-卡初始化流程)
- **[六、软件架构传递树](#六软件架构传递树)**
- **[七、代码逐段解析](#七代码逐段解析)**
  - [7.1 sd_spi_init()](#71-sd_spi_init)
  - [7.2 如何与 main.c 集成](#72-如何与-mainc-集成)
- **[八、完整通信时序图](#八完整通信时序图)**
  - [8.1 初始化阶段](#81-初始化阶段)
  - [8.2 读写阶段](#82-读写阶段)
- **[容易踩坑](#容易踩坑)**
- **[九、本章小结](#九本章小结)**
- **[Tips](#tips)**
  - [VFS 的魔法——fopen 为什么能打开 SD 卡上的文件](#vfs-的魔法——fopen-为什么能打开-sd-卡上的文件)
  - [SD 卡容量规格的寻址陷阱](#sd-卡容量规格的寻址陷阱)
  - [SPI 总线共享的设计智慧](#spi-总线共享的设计智慧)
  - [踩坑实录](#踩坑实录)
  - [知识衔接](#知识衔接)

---

## 学习前后依赖

| 依赖方向     | 章节                                                   | 知识点                             |
| -------- | ---------------------------------------------------- | ------------------------------- |
| **前置基础** | 01_LED                         | GPIO 输出配置                       |
| **前置基础** | 14_IICBH1750             | 双线通信协议原理、推挽/开漏                  |
| **前置基础** | 16_SPILCD                   | SPI 四线通信、Mode 0 时序、CS 片选、DMA    |
| **前置基础** | 0-1_under_Loop_stu | 轮询 vs 中断 vs DMA 三阶段递进           |
| **本章核心** | 17_SPITF（本章）                                         | SD 卡容量体系 + FATFS + VFS + SPI 联动 |
| **后续承接** | 后续项目                                                 | 日志存储、配置文件、OTA 固件升级、MQTT + SD 缓存 |

---

## 一、概述

### 1.1 从 SD 卡到 TF 卡

SD 卡和 TF 卡本质上**是同一种东西的两个外观版本**：

```
        SD 大卡（32×24×2.1mm）      TF 卡 = microSD（15×11×1.0mm）
         ┌──────────────┐           ┌─────┐
         │  ████████████│           │░░░░░│
         │  ██  SD ██   │           │░░░░░│
         │  ████████████│           └─────┘
         └──────────────┘             ↑
           9-pin 金手指              8-pin 金手指
```

| 对比维度 | SD 大卡 | TF 卡（microSD） |
|---------|--------|-----------------|
| 尺寸 | 32 × 24 × 2.1 mm | 15 × 11 × 1.0 mm |
| 引脚数 | 9 根（含写保护检测） | 8 根（无写保护） |
| 电气协议 | 完全一致（SD 1.0/2.0/3.0） | 完全一致 |
| 转换卡套 | — | TF → SD 卡套（纯物理转接，无芯片） |
| 容量上限 | SDUC 支持到 128 TB | 同左 |
| ESP32 常用 | 较少（体积大） | 主流（模块自带 TF 卡座） |

> **关键结论**：把 TF 卡插入 SD 卡套就变成 SD 卡。电气上完全相同，驱动代码完全共用。

### 1.2 为什么用 SPI 而非 SDIO？

SD 卡支持两种通信模式：

| 模式 | 信号线 | 速度 | ESP32-S3 支持 | 复杂度 |
|------|--------|------|--------------|--------|
| **SPI** | 4 线（MOSI、MISO、CLK、CS） | ~25 MB/s（高速 SPI） | 任何 SPI2/3 通道 | 低 |
| **SDIO 1-bit** | 5 线（CMD、CLK、DATA0） | ~25 MB/s | 需要专用 SDMMC 外设 | 中 |
| **SDIO 4-bit** | 8 线（CMD、CLK、DATA0~3） | ~100 MB/s | 需要专用 SDMMC 外设 | 高 |

ESP32-S3 选择 SPI 的理由：
- **引脚占用少**（4 根 vs 8 根），屏幕已经占了一个 SPI2，SD 卡还可以复用同一条 SPI 总线
- **SD 卡上电默认进入 SPI 模式**（CMD0 命令后），无需额外配置
- **ESP-IDF 的 `esp_vfs_fat_sdspi_mount()` 一行代码搞定初始化 + 挂载 + VFS 注册**

---

## 二、TF 卡数据存储与容量体系

### 2.1 NAND Flash 内部结构

TF 卡内部是一颗 **NAND Flash** 芯片加一个**微型控制器**：

```
┌─────────────────────────────────────────┐
│              TF 卡内部                    │
│  ┌─────────┐         ┌───────────────┐  │
│  │  NAND   │  ◄───►  │  微型控制器    │  │
│  │  Flash  │         │  (FTL 层)     │  │
│  │  存储体  │         │  坏块管理     │  │
│  │         │         │  磨损均衡     │  │
│  │         │         │  ECC 纠错     │  │
│  └─────────┘         └───────┬───────┘  │
│                               │          │
│                        SPI / SDIO 接口   │
└───────────────────────────────┼──────────┘
                                │
                     MOSI MISO CLK CS
```

微型控制器做的事：
- **FTL**（Flash Translation Layer）：把逻辑扇区地址映射到 NAND 物理页
- **磨损均衡**（Wear Leveling）：平均分配擦写次数，避免某个块过早损坏
- **坏块管理**：出厂即有坏块标记，使用中新增坏块自动替换
- **ECC 纠错**：每页数据带校验码，读出时自动纠正单 bit 错误

> 这些你都不用管——微型控制器对主机透明，你的代码只管读写扇区。

### 2.2 三大容量规格：SDSC / SDHC / SDXC

| 规格 | 容量范围 | 寻址模式 | 文件系统 | 典型标记 |
|------|---------|---------|---------|---------|
| **SDSC** | ≤ 2 GB | Byte 寻址（CMD17/24） | FAT16 | 无标记或"SD" |
| **SDHC** | 2 GB ~ 32 GB | Block（512B）寻址 | FAT32 | 标"HC" |
| **SDXC** | 32 GB ~ 2 TB | Block 寻址 | exFAT | 标"XC" |
| **SDUC** | 2 TB ~ 128 TB | Block 寻址 | exFAT | 标"UC"（极少见） |

**容量判断示意图：**

```
                  2GB                   32GB                     2TB
     SDSC         │       SDHC          │        SDXC           │
 ─────────────────┼─────────────────────┼───────────────────────┼──►
   Byte 寻址      │     512B Block 寻址  │    512B Block 寻址     │
   FAT16          │     FAT32           │    exFAT              │
```

SDHC/SDXC **强制要求 Block（512 字节）寻址**——`CMD17` 的参数不再是"字节偏移"，而是"扇区号"。

### 2.3 顺序 vs 随机读写

```
顺序读写（Sequential）：
 ┌─┬─┬─┬─┬─┬─┬─┬─┐
 │0│1│2│3│4│5│6│7│  → 连续扇区，速度最快
 └─┴─┴─┴─┴─┴─┴─┴─┘

随机读写（Random）：
 ┌──────────────────────────────────────────┐
 │扇区0 │...│扇区73 │...│扇区512 │...│扇区8│  → 跳来跳去，慢
 └──────────────────────────────────────────┘
```

| 操作类型 | Class 10 卡典型速度 | 用途 |
|---------|-------------------|------|
| 顺序写 | 10~20 MB/s | 日志追加、录视频 |
| 顺序读 | 20~30 MB/s | 加载图片、回放 |
| 随机写 4KB | 0.5~2 MB/s | 数据库、配置文件 |
| 随机读 4KB | 2~5 MB/s | 检索小文件 |

**实战启示**：ESP32 写日志时，尽量用 `fprintf` 连续写入，不要频繁 `fopen`/`fclose`。`mount_config.max_files = 5` 限制同时打开文件数的意义就在这里——避免大量随机寻道。

---

## 三、文件系统与 FATFS

### 3.1 为什么需要文件系统？

如果**没有文件系统**，你对 TF 卡的读写是这样的：

```c
// 裸扇区读写：没有任何"文件"概念
uint8_t buf[512];
sd_read_sector(1000, buf);  // 读第 1000 号扇区
sd_write_sector(2000, buf); // 写第 2000 号扇区
```

问题：
- 你得手动记录"日志写在扇区 1000~9999、图片存在 10000~19999"
- 删除数据时不知道哪些扇区可以重用
- 拔卡插电脑上，Windows 不认（没有 FAT 文件系统）

**有了文件系统**之后：

```c
FILE *f = fopen("/sdcard/log.txt", "a");
fprintf(f, "temperature: %d\n", temp);
fclose(f);
// 拔卡插电脑——直接打开 log.txt 就能看
```

这就是文件系统存在的意义：**把扇区抽象成文件，人类可读写，跨设备可识别。**

### 3.2 FAT 文件系统结构

FAT（File Allocation Table）是 SD 卡最通用的文件系统：

```
┌───────────┬───────────┬───────────┬───────────┬───────────┐
│   MBR     │    FAT 表  │   根目录  │   数据区   │   数据区   │
│  (扇区0)   │  (簇链表)  │ (文件索引) │ (簇2开始)  │   ...     │
└───────────┴───────────┴───────────┴───────────┴───────────┘
```

**FAT 表的本质：一个扇区级的"链表"**

```
假设 log.txt 占用了簇 5、簇 8、簇 12：

FAT 表：   簇5 → 簇8 → 簇12 → EOF
          (5号位置 (8号位置 (12号位置
           写"8")  写"12") 写结束标记)

文件即使碎片化分散在不同扇区，FAT 表能把它们串起来。
```

| 区域 | 大小 | 作用 |
|------|------|------|
| MBR | 512 B | 启动记录 + 分区表（告诉系统这是一张 FAT 盘） |
| 保留区 | 若干扇区 | 备份引导扇区 |
| **FAT 表** | 取决于容量 | 文件系统的"链表"，记录每个簇的下一簇号 |
| **根目录区** | FAT12/16 固定，FAT32 可变 | 存放文件名、大小、创建时间、起始簇号 |
| **数据区** | 剩余空间 | 文件实际内容，按簇（Cluster）分配 |

### 3.3 FATFS 是什么？

**FATFS** 是一个很轻量的开源 FAT 文件系统实现（纯 C 语言），专为嵌入式设计。它不依赖操作系统，只做一件事：**解析 FAT 表、管理目录、读写文件**。

```
 FATFS 在 ESP-IDF 中的位置：

 ┌────────────┐
 │ 你的代码    │  fopen / fread / fwrite / fclose
 └─────┬──────┘
       │  C 标准库调用
 ┌─────▼──────┐
 │   VFS      │  ESP-IDF 虚拟文件系统（分发路由）
 └─────┬──────┘
       │  路由到 /sdcard 路径
 ┌─────▼──────┐
 │   FATFS    │  解析 FAT 表、计算扇区号、管理目录项
 └─────┬──────┘
       │  块读写请求（扇区号 + 缓冲区）
 ┌─────▼──────┐
 │ SDSPI 驱动  │  把扇区号转换成 SPI CMD17/CMD24 命令
 └─────┬──────┘
       │  SPI 帧
 ┌─────▼──────┐
 │  TF 卡     │
 └────────────┘
```

FATFS 的特性：
- **代码极小**：最小配置下 < 10 KB ROM
- **支持 FAT12 / FAT16 / FAT32 / exFAT**
- **支持长文件名（LFN）**
- **支持 RTOS 多任务（FATFS 内部有互斥锁）**
- **ESP-IDF 已内置**：路径 `components/fatfs/`

---

## 四、FATFS 与 C 标准库 I/O 的桥梁：VFS

### 4.1 传统裸机读 SD 卡 vs VFS 模式

这是一段**不用 VFS 的裸 FATFS 代码**：

```c
// 裸 FATFS：用 f_open / f_read / f_write，而不是 fopen / fread / fwrite
FATFS fs;
FIL file;
f_mount(&fs, "", 1);           // 挂载文件系统
f_open(&file, "log.txt", FA_WRITE | FA_CREATE_ALWAYS);
f_write(&file, "hello", 5, &bw);
f_close(&file);
```

这是一段**用了 VFS 后的代码**：

```c
// VFS 模式：用 fopen / fread / fwrite / fprintf 等标准 C 函数
FILE *f = fopen("/sdcard/log.txt", "w");
fprintf(f, "hello\n");
fclose(f);
```

**VFS 做的事**：把 `fopen("/sdcard/log.txt")` 这个 call：
1. 解析路径前缀 `/sdcard` → 找到注册的 FATFS 驱动
2. 把剩余路径 `log.txt` + 标准 C 的 `fopen` 参数翻译成 FATFS 的 `f_open` 调用
3. 返回一个 `FILE *` 指针给你——你感觉不到 FATFS 的存在

### 4.2 ESP-IDF VFS 注册流程

```
esp_vfs_fat_sdspi_mount("/sdcard", &host, &slot, &mount, &card)
    │
    ├─ spi_bus_initialize(SPI2_HOST, ...)    复制 SPI2 驱动初始化后的总线
    │     内部再次初始化 SPI2（如果已经初始化过，跳过）
    │
    ├─ sdspi_host_init_device(&slot, &handle)  初始化 CS 引脚
    │
    ├─ sdmmc_card_init(&host, card)            发送 CMD0→CMD8→ACMD41 序列
    │     识别 SDSC / SDHC / SDXC 类型
    │     读取 CID、CSD、OCR 寄存器
    │     确定扇区大小（512B or 1024B）
    │
    ├─ esp_vfs_fat_sdmmc_mount("/sdcard", &host, card, &mount)
    │     注册 VFS：把 "/sdcard" 路径绑定到 FATFS 驱动
    │     读写请求 → FATFS → sdmmc_host.do_transaction() → SPI CMD17/24
    │
    └─ 返回 ESP_OK（card 指向已填充的卡信息结构体）
```

> `esp_vfs_fat_sdspi_mount()` 实际上是 `esp_vfs_fat_sdmmc_mount` 的一个包装宏。

---

## 五、TF 卡与 ESP32-S3 的 SPI 联动

### 5.1 硬件连接总览

```
          ESP32-S3                          TF 卡模块（microSD）
  ┌──────────────────────┐           ┌────────────────────────┐
  │                      │           │                        │
  │  SPI2                │           │  CS   ←── GPIO 2      │
  │  ┌────────────────┐  │           │  MOSI ←── GPIO 11     │
  │  │ MOSI (GPIO 11) ├──┼───────────┼─► MOSI                │
  │  │ MISO (GPIO 13) │◄─┼───────────┼── MISO                │
  │  │ CLK  (GPIO 12) ├──┼───────────┼─► CLK                 │
  │  │ CS   (GPIO 2)  ├──┼───────────┼─► CS                  │
  │  └────────────────┘  │           │                        │
  │                      │           │  3.3V  ←── ESP 3.3V   │
  │  SPI2                │           │  GND  ←── ESP GND     │
  │  (复用给 LCD)         │           └────────────────────────┘
  │  ┌────────────────┐  │
  │  │ CS_LCD(GPIO 48)│  │   ← LCD 用独立的 CS，与 TF 卡
  │  └────────────────┘  │     共享 SPI2 总线，分时复用
  └──────────────────────┘
```

> 关键设计：LCD 和 TF 卡**共享 SPI2（MOSI 11、MISO 13、CLK 12）**，通过两个独立的 CS 片选线（GPIO 48 给 LCD、GPIO 2 给 TF 卡）分时访问。

### 5.2 SPI 模式下的 SD 命令

SD 卡命令是固定长度的 6 字节帧，通过 SPI MOSI 发送：

```
 SD 命令帧（SPI 模式）
 ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
 │ Byte0    │ Byte1    │ Byte2    │ Byte3    │ Byte4    │ Byte5    │
 │ 0 1 xxxxx│地址[31:24]│地址[23:16]│地址[15:8]│地址[7:0] │ CRC      │
 └──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
   命令索引   32-bit 参数（扇区号 / 字节偏移）
```

常用命令：

| 命令 | 索引 | 参数 | 响应 | 说明 |
|------|------|------|------|------|
| **CMD0** | 0 | 0 | R1 | 复位到 Idle 状态（进入 SPI 模式） |
| **CMD8** | 8 | 0x1AA | R7 | 查询电压范围（SDHC/SDXC 存在检测） |
| **ACMD41** | 41 | HCS 位 | R3 | 初始化（含容量识别） |
| **CMD17** | 17 | 扇区号 | R1 + 512B | 读单个扇区 |
| **CMD24** | 24 | 扇区号 | R1 + 512B | 写单个扇区 |

### 5.3 SD 卡初始化流程

```
上电
 │
 ▼
空闲状态（Idle）
 │
 ├─ 发送 CMD0  ──→  复位，进入 SPI 模式
 │      │
 │      ▼          (R1 响应 ≠ 0x01 则失败)
 │
 ├─ 发送 CMD8  ──→  检查电压 + 识别 SDHC/SDXC
 │      │             参数 0x1AA，期望 R7=0x1AA
 │      ▼
 │   收到 R7=0x01AA?  ──Yes──► SD Ver 2.0+ (SDHC/SDXC)
 │      │ No
 │      ▼
 │   可能是 SDSC (≤2GB) 或极老的 MMC 卡
 │
 ├─ 循环发送 ACMD41 ──→  等待卡进入 Ready 状态
 │      │                 HCS=1 表示支持 SDHC/SDXC
 │      │                 超时通常设 1~2 秒
 │      ▼
 │   OCR 的 bit30=1? ──Yes──► SDHC/SDXC (Block 寻址)
 │      │ No
 │      ▼
 │   SDSC (Byte 寻址，≤2GB)
 │
 ├─ 读取 CID / CSD ──→  获取序列号、容量、速度等级
 │
 ▼
就绪（Ready）→ 可以 CMD17 读 / CMD24 写了
```

---

## 六、软件架构传递树

```
app_main()                                  [main\main.c]
  │
  ├─ lcd_init()                             [components\LCD\lcd.c]
  │    ├─ spi2_init()                        [components\SPI\spi.c]
  │    │    └─ spi_bus_initialize()          ESP-IDF
  │    │         └─ 配置 SPI2: MOSI=11, MISO=13, CLK=12, DMA 自动
  │    │
  │    ├─ spi_bus_add_device()               ESP-IDF
  │    │    └─ 注册 LCD 设备: 60MHz, Mode 0, CS=GPIO48
  │    │
  │    ├─ gpio_config() × 3                  ESP-IDF
  │    │    └─ 配置 DC=GPIO47, BLK=GPIO40, RST=GPIO21
  │    │
  │    └─ lcd_hard_reset() → 初始化序列
  │
  ├─ sd_spi_init()                           [components\SDIO\spi_sdcard.c]
  │    ├─ SDSPI_HOST_DEFAULT()                ESP-IDF 宏
  │    │    └─ 生成 sdmmc_host_t:
  │    │         flags = SPI, slot = SPI2_HOST
  │    │         do_transaction → sdspi_host_do_transaction
  │    │         set_card_clk   → sdspi_host_set_card_clk
  │    │
  │    ├─ SDSPI_DEVICE_CONFIG_DEFAULT()       ESP-IDF 宏
  │    │    └─ 设置 CS = GPIO 2
  │    │
  │    ├─ esp_vfs_fat_mount_config_t          ESP-IDF 结构体
  │    │    └─ format=false, max_files=5
  │    │
  │    └─ esp_vfs_fat_sdspi_mount()          ESP-IDF (五合一函数)
  │         ├─ 内部 spi_bus_initialize()      (如果 SPI2 未初始化则初始化)
  │         ├─ 内部 sdspi_host_init_device()  (注册 CS 信号)
  │         ├─ 内部 sdmmc_card_init()          (发 CMD0→CMD8→ACMD41)
  │         ├─ 内部 esp_vfs_fat_sdmmc_mount() (注册 VFS /sdcard)
  │         └─ 返回 card 句柄 → 全局 sdcard
  │
  └─ while(1)
       ├─ 标准 C fopen/fread/fwrite/fprintf("/sdcard/...")
       │    └─ VFS 路由 → FATFS → sdmmc_host.do_transaction() → SPI2
       │
       └─ LCD 刷新界面
            └─ spi2_write_data() → ESP-IDF SPI2 轮询发送
```

---

## 七、代码逐段解析

### 7.1 `sd_spi_init()` 详解

> 源文件：`components/SDIO/spi_sdcard.c`

```c
#include "spi_sdcard.h"
#include "esp_vfs_fat.h"

sdmmc_card_t* sdcard;  // ★ 全局卡信息句柄，挂载成功后指向卡信息
```

`sdspi_host_init_device` 挂载成功后可通过它获取卡信息：

```c
if (sdcard != NULL) {
    printf("SD Card Size: %llu MB\n",
           ((uint64_t)sdcard->csd.capacity * sdcard->csd.sector_size) / (1024 * 1024));
}
```

**函数体逐行解析：**

```c
esp_err_t sd_spi_init(void)
{
    esp_err_t ret;

    // ① 获取默认 SPI 主机配置
    const sdmmc_host_t host = SDSPI_HOST_DEFAULT();
```

`SDSPI_HOST_DEFAULT()` 展开为一个含 `do_transaction` 和 `set_card_clk` 函数指针的结构体。这些函数 ESP-IDF 已经帮你写好了——初始化阶段用低速（~400 kHz），识别卡后自动切换到高速（~20 MHz）。

```c
    // ② 配置 Slot（主要是改 CS 引脚）
    sdspi_device_config_t slot_config = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot_config.host_id = SPI2_HOST;   // 明确用 SPI2
    slot_config.gpio_cs = GPIO_NUM_2; // 覆盖默认 GPIO 10
```

`SDSPI_DEVICE_CONFIG_DEFAULT()` 内部设置了 `host_id=SPI2_HOST` 和 `gpio_cs=10`。这里重新设一遍是为了明确关系，并把 CS 从 GPIO 10 改成 GPIO 2。

```c
    // ③ 配置 FAT 文件系统参数
    esp_vfs_fat_mount_config_t mount_config = {
        .format_if_mount_failed = false,   // 挂载失败不格式化（保护数据）
        .max_files = 5,                    // 最多同时打开 5 个文件
        .allocation_unit_size = 0,         // 自动选择簇大小
        .disk_status_check_enable = false, // 不轮询磁盘状态
    };
```

| 字段 | 为什么这样设 |
|------|------------|
| `format_if_mount_failed = false` | SD 卡已有数据，不能抹掉 |
| `max_files = 5` | 日志文件 + 配置文件 + 少量目录遍历，5 个够用 |
| `allocation_unit_size = 0` | 让 FATFS 根据卡容量自动选最优簇大小 |
| `disk_status_check_enable = false` | 省 CPU 时间，除非要实现热插拔检测 |

```c
    // ④ 一行搞定：SPI 初始化 + SD 识别 + FAT 挂载 + VFS 注册
    ret = esp_vfs_fat_sdspi_mount("/sdcard", &host, &slot_config,
                                   &mount_config, &sdcard);

    return ret;
}
```

`"/sdcard"` 是挂载点。挂载成功后：
- `fopen("/sdcard/log.txt", "w")` 可以写文件
- `opendir("/sdcard")` 可以遍历目录
- `stat("/sdcard/config.ini", &st)` 可以读文件属性
- 拔卡插电脑，Windows 直接识别 FAT 文件系统

### 7.2 如何与 `main.c` 集成

```c
#include "spi_sdcard.h"
#include "lcd.h"

void app_main(void)
{
    lcd_init();                            // ① 先初始化 LCD（用 SPI2）

    esp_err_t ret = sd_spi_init();         // ② 再挂载 SD 卡（复用 SPI2）
    if (ret == ESP_OK) {
        printf("SD card mounted at /sdcard\n");
    } else {
        printf("SD mount failed: %s\n", esp_err_to_name(ret));
        // 即使挂载失败，LCD 仍然可用
    }

    // ③ 写一句日志到 SD 卡
    FILE *f = fopen("/sdcard/hello.txt", "w");
    if (f) {
        fprintf(f, "ESP32-S3 SPI TF Card Test\n");
        fclose(f);
    }

    // ④ LCD 照常显示
    lcd_show_string(2, 1, "SD Card OK", BLUE, BLACK);

    while(1) { vTaskDelay(1000); }
}
```

> 另外还要修改 `components/CMakeLists.txt`：把 `SDIO` 目录加入 `src_dirs` 和 `include_dirs`，并在 `requires` 里加上 `fatfs`。

---

## 八、完整通信时序图

### 8.1 初始化阶段

```
上电后 SD 卡处于 Idle（仅响应 CMD0）
  │
  ├─ ESP32 发送 CMD0（复位）
  │   主机:  CMD0
  │   SD:     R1(0x01) = In Idle State
  │
  ├─ ESP32 发送 CMD8（查电压 + 判别 SDHC/SDXC）
  │   主机:  CMD8(0x1AA)
  │   SD:     R7(0x01AA) → 是 SD 2.0+ 卡，支持 3.3V
  │
  ├─ ESP32 循环发送 ACMD41（等 Ready）
  │   主机:  ACMD41(HCS=1)
  │   SD:     R1(0x01) = still busy  ... 循环 N 次 ...
  │   SD:     R1(0x00) = Ready!
  │
  ├─ 读 OCR → bit30=1 → SDHC/SDXC，Block 寻址
  │
  ├─ 读 CID（序列号）→ 读 CSD（容量、扇区大小）
  │   主机:  CMD9(读 CSD)
  │   SD:     R1 + 16B CSD 数据
  │
  └─ esp_vfs_fat_sdmmc_mount("/sdcard")
       注册 VFS，FAT 分区已可访问
```

### 8.2 读写阶段

```
写文件流程：fopen("/sdcard/test.txt","w") → fprintf → fclose

  应用层:  fopen              fprintf           fclose
            │                   │                 │
   VFS:     │ 路由 /sdcard      │ 路由 /sdcard     │ 路由 /sdcard
            ▼                   ▼                 ▼
  FATFS:  f_open              f_write           f_close
            创建目录项          计算扇区号         更新目录项
            分配首簇            攒满一个扇区        写回 FAT 表
            写入 FAT 表         CMD24 写扇区
                                 ▼
  SDSPI:   ───────────    CMD24 → 扇区号 → 512B 数据 → SPI2 → TF 卡
           不涉及 SPI
```

---

## 容易踩坑

| 序号 | 坑 | 表现 | 原因与修正 |
|------|----|------|-----------|
| 1 | **仅挂载失败** | `esp_vfs_fat_sdspi_mount` 返回 `ESP_FAIL` | 检查 TF 卡是否插好、是否 FAT32 格式（exFAT 需 `CONFIG_FATFS_LFN`） |
| 2 | **挂载成功但 LCD 花屏** | LCD 显示乱码、白屏 | LCD 和 SD 卡共用 SPI2，SD 卡初始化期间会拉低 CS 线。确保 CS 硬件连接正确 |
| 3 | **SDIO 代码编译不进去** | 链接时找不到 `sd_spi_init` 符号 | `components/CMakeLists.txt` 里没加 `SDIO` 目录，也没加 `fatfs` 到 `requires` |
| 4 | **文件写入后电脑读不到** | SD 卡插入电脑找不到文件 | 写入后记得 `fclose()`，否则数据还在 FATFS 内部缓冲区 |
| 5 | **随机读写特别慢** | 4KB 写入 > 50 ms | TF 卡随机写入本来就慢（~0.5 MB/s），尽量顺序写、攒大数据再 `fwrite` |
| 6 | **中文文件名乱码** | `fopen` 中文路径失败 | FATFS 默认只支持 ASCII，需启用 `CONFIG_FATFS_CODEPAGE=936` |
| 7 | **挂载后 SPI 总线频率太保守** | SD 读写带宽不足 | `SDSPI_HOST_DEFAULT()` 默认 ~20 MHz。手动改 `host.max_freq_khz = 40000` |
| 8 | **TF 卡是 SDSC（≤2GB）老卡但代码当 SDHC 处理** | 读扇区数据偏移错误 | SDSC 用 Byte 寻址，SDHC 用 Block（512B）寻址 |
| 9 | **第一次使用空白卡挂载失败** | `ESP_FAIL` | `mount_config.format_if_mount_failed = true` |

---

## 九、本章小结

| 知识点 | 核心结论 |
|--------|---------|
| **SD 卡 vs TF 卡** | 协议完全相同，仅物理尺寸不同，驱动代码通用 |
| **容量规格** | SDSC ≤ 2GB（Byte 寻址）、SDHC 2~32GB（Block 寻址）、SDXC 32GB~2TB（Block 寻址） |
| **FAT 文件系统** | MBR → FAT 表（簇链表）→ 根目录（文件索引）→ 数据区（实际内容） |
| **FATFS** | 开源轻量 FAT 实现，< 10 KB，支持 RTOS，ESP-IDF 已内置 |
| **VFS 桥接** | `fopen/fread/fwrite` → VFS 路由 → FATFS → SDSPI → TF 卡，开发者只写 C 标准库代码 |
| **SPI 联动** | LCD 和 TF 卡共享 SPI2 总线（MOSI 11、CLK 12、MISO 13），用独立 CS 分时切换 |
| **SD 命令** | CMD0 进 SPI 模式 → CMD8 查电压 → ACMD41 等就绪 → CMD17/CMD24 读写 |
| **初始化** | `esp_vfs_fat_sdspi_mount()` 一行完成 SPI 总线初始化 + 卡识别 + FAT 挂载 + VFS 注册 |
| **关键配置** | `max_files=5`（同时打开文件数）、`format_if_mount_failed=false`（保护数据） |
| **容易踩坑** | 忘 `fclose` 数据未写回、CMakeLists 没加组件导致编译不到、SDSC 用错寻址模式 |

**至此**，你已经学完了从 GPIO 到 SPI 到 I2C 到 UART 到 DMA 再到 TF 卡文件系统这一整条 ESP32-S3 外设链路。

---

## Tips

### VFS 的魔法——fopen 为什么能打开 SD 卡上的文件

这是本章最值得深入理解的设计模式。当你写下 `fopen("/sdcard/log.txt", "w")` 时，这一行代码背后发生了 4 层转发：

```
  你的代码:  fopen("/sdcard/log.txt", "w")
              │
  1. C 标准库 (newlib):  解析路径前缀 "/sdcard"
              │          查找 VFS 注册表中匹配的驱动
              ▼
  2. ESP-IDF VFS 层:    匹配到 FATFS 驱动
              │          将剩余路径 "log.txt" 传入驱动
              ▼
  3. FATFS (ff.c):      解析 FAT 表，分配簇
              │          将"创建文件"翻译成"写 FAT 表 + 写目录项"
              ▼
  4. SDSPI 驱动:         FATFS 说"写扇区 5000"
              │          转换为 CMD24 + SPI 帧 → TF 卡
              ▼
           TF 卡 NAND Flash 物理写入
```

**这就是 VFS 的注册机制**：`esp_vfs_fat_sdspi_mount("/sdcard", ...)` 在初始化时会把 `"/sdcard"` 这个路径前缀和 FATFS 驱动绑定，存在一个全局的 VFS 注册表中。此后任何以 `"/sdcard"` 开头的文件操作（`fopen`、`fread`、`fwrite`、`fprintf`、`fclose`、`stat`、`opendir`……）全部自动转发到 FATFS。

**设计哲学**：VFS 层让"硬件存储"对业务代码完全透明。你可以今天把日志写到 SD 卡（`/sdcard`），明天改写到 SPI Flash（`/spiflash`），业务的 `fprintf` 一行代码都不用改——只需改挂载路径。这就是分层架构的价值。

### SD 卡容量规格的寻址陷阱

这是 SD 卡驱动中最容易出隐蔽 bug 的地方：

```c
// SDSC (≤2GB): 用 Byte 寻址
// 读第 1 个扇区: CMD17 的参数 = 512（字节偏移）
// 读第 2 个扇区: CMD17 的参数 = 1024

// SDHC/SDXC (>2GB): 用 Block 寻址
// 读第 1 个扇区: CMD17 的参数 = 0（扇区号）
// 读第 2 个扇区: CMD17 的参数 = 1
```

如果你拿着一张 16GB 的 SDHC 卡，却用 Byte 寻址去读写，结果是：你读的是正确的扇区号 × 512 的偏移位置，数据全错。更糟糕的是——**卡不会报错**，它只是从错误的地址返回了无意义的数据。

ESP-IDF 的 `sdmmc_card_init()` 会自动识别容量类型，存到 `sdcard->ocr` 寄存器中。FATFS 通过 `sdcard->csd.sector_size` 获取扇区大小（通常是 512），不会再换算。所以**只要用 ESP-IDF 的标准 API，就不用关心这个问题**。但如果你自己写裸驱动，这是第一个要检查的地方。

### SPI 总线共享的设计智慧

LCD 和 SD 卡共享 SPI2 是本章最优雅的工程决策：

```
  SPI2 总线 = 一组 SCLK/MOSI/MISO（物理线缆）
  
  ┌ SPI2 ───┬─── LCD (CS=GPIO48)   ← 拉低 CS48 → LCD 被选中
  │         │
  │         └─── SD  (CS=GPIO2)    ← 拉低 CS2 → SD 被选中
  │
  两个从机永远不会同时被选中（CS 互斥）
```

**前提条件**：LCD 和 SD 卡的速度兼容、SPI 模式兼容。本章 LCD 用 60MHz Mode 0，SD 卡初始化用 400kHz（识别后提至 20MHz），都工作在 Mode 0。两者兼容。

**一个常见问题**：如果在 SD 卡传输数据（CS2=低）的同时，LCD 也拉了 CS48——此时 SPI2 总线上的 SCLK/MOSI/MISO 会被两个从机同时监听。LCD 可能误把 SD 命令当成像素数据。解决方法是**软件互斥锁**——在操作 SD 卡前先加锁，完成后释放，LCD 刷新也加同一把锁。

### 踩坑实录

1. **`fopen` 成功但 `fprintf` 失败**：`mount_config.max_files = 5` 是全局限制。如果已经打开了 5 个文件（包括目录遍历句柄），第 6 个 `fopen` 会失败。关闭不用的文件再试。

2. **拔卡后程序卡死**：FATFS 在读写时如果卡被拔出，会进入无限重试。解决方案：启用 `disk_status_check_enable` 或在任务中加入超时逻辑。

3. **exFAT 卡不识别**：默认的 FATFS 只编译了 FAT12/16/32 支持。需要在 `menuconfig` 中启用 `CONFIG_FATFS_LFN` 和 exFAT 支持。

### 知识衔接

- **上一章**：[03_SPI_LCD](03_SPI_LCD.md)——SPI 总线和 CS 片选机制是理解本章"LCD + SD 卡共享 SPI2"的前提。
- **下一章**：[05_Camera](05_Camera.md)——摄像头通过 DVP 并行接口高速传输图像数据。和 SD 卡一样，摄像头也用 DMA 做数据搬运，用 PSRAM 做帧缓冲。理解了 SPI+DMA+FS 的协作模式，阅读摄像头代码会更顺畅。
- **高级扩展**：基于 SD 卡做离线 OTA（`esp_ota_write` 从 SD 卡读取固件）、MQTT 日志上传（WiFi + SD 卡定时同步）。
