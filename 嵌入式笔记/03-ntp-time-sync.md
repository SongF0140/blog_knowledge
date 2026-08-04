---
title: "NTP 网络校时与时间显示 — ESP32-S3 获取网络时间并显示到 LCD"
published: 2026-08-28
description: "本章项目 23NTCTime 是一个 ESP32-S3 通过 Wi-Fi 连接路由器，再通过 SNTP 向 NTP 服务器校时，最后把当前日期时间显示到 LCD 上 的示例工程。"
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","无线通信","NTP"]
category: 物联网
draft: false
---
# NTP 网络校时与时间显示 — ESP32-S3 获取网络时间并显示到 LCD

## 本章重要 API 函数

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `nvs_flash_init()` | 初始化 NVS，为 Wi-Fi 驱动提供底层参数存储 | `void`：无参数；返回 `esp_err_t`，`ESP_OK` 表示成功 | `"nvs_flash.h"` |
| `esp_wifi_connect()` | STA 模式主动连接路由器 | `void`：无参数；返回 `esp_err_t` | `"esp_wifi.h"` |
| `esp_sntp_setoperatingmode()` | 设置 SNTP 工作模式 | `esp_sntp_operatingmode_t mode`：如 `ESP_SNTP_OPMODE_POLL`；无返回值 | `"esp_sntp.h"` |
| `esp_sntp_setservername()` | 配置 NTP 服务器域名 | `u8_t idx`：服务器下标；`const char *server`：服务器域名；无返回值 | `"esp_sntp.h"` |
| `esp_sntp_init()` | 启动 SNTP 客户端 | `void`：无参数；无返回值 | `"esp_sntp.h"` |
| `setenv()` | 设置环境变量 | `const char *name`：变量名；`const char *value`：变量值；`int overwrite`：是否覆盖；返回 `int` | `<stdlib.h>` |
| `tzset()` | 让 C 库重新加载时区设置 | `void`：无参数 | `<time.h>` |
| `time()` | 获取当前 Unix 时间戳 | `time_t *tloc`：可为 `NULL`；返回 `time_t` 秒数 | `<time.h>` |
| `localtime()` | 将 Unix 时间戳转换为本地时间结构体 | `const time_t *timer`：时间戳指针；返回 `struct tm *` | `<time.h>` |

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数](#本章重要-api-函数)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. 本章目录结构](#2-本章目录结构)**
- **[3. 软件架构传递树](#3-软件架构传递树)**
  - [3.1 模块调用关系](#31-模块调用关系)
  - [3.2 时间同步与显示流程](#32-时间同步与显示流程)
- **[4. 代码讲解](#4-代码讲解)**
  - [4.1 `main/main.c` — 主程序入口](#41-main/mainc-—-主程序入口)
  - [4.2 `components/MYNTC/myntc.c` — SNTP 校时模块](#42-components/myntc/myntcc-—-sntp-校时模块)
  - [4.3 `components/WIFISTA/wifista.c` — STA 联网模块](#43-components/wifista/wifistac-—-sta-联网模块)
  - [4.4 为什么 `myntc_init()` 必须放在 `wifista_init()` 后面](#44-为什么-myntc_init-必须放在-wifista_init-后面)
  - [4.5 为什么先设置 `TZ` 再用 `localtime()`](#45-为什么先设置-tz-再用-localtime)
- **[5. 时序流程图](#5-时序流程图)**
  - [5.1 上电到时间显示的完整时序](#51-上电到时间显示的完整时序)
- **[6. NTP 校时的本质](#6-ntp-校时的本质)**
- **[容易踩坑](#容易踩坑)**
- **[7. 本章小结](#7-本章小结)**
- **[Tips](#tips)**

---

## 学习前后依赖

| 方向 | 内容 |
|------|------|
| 前置基础 | `20_WIFI_STA` 或 `21_WIFI_STA` 中的 Wi-Fi STA 联网流程、`0-1_under_Loop_stu/wifi_notes.md` 中的 IP 与客户端/服务器基础、LCD 显示函数使用方式 |
| 本章核心 | 掌握 ESP32 如何先联网，再通过 SNTP 从 NTP 服务器获取标准时间，最后用 `time()` / `localtime()` 转为本地时间并显示到 LCD |
| 后续承接 | 为网络时钟、定时任务、日志时间戳、带 RTC 的联网设备、MQTT 带时间上报等应用打基础 |

---

## 1. 概述

本章项目 `23_NTCTime` 是一个 **ESP32-S3 通过 Wi-Fi 连接路由器，再通过 SNTP 向 NTP 服务器校时，最后把当前日期时间显示到 LCD 上** 的示例工程。

这个工程把三件事串到了一起：

1. **联网**：通过 STA 模式连上路由器；
2. **校时**：通过 SNTP 向公网时间服务器同步标准时间；
3. **显示**：每秒读取一次本地时间并刷新 LCD。

因此它既是一个 Wi-Fi 工程，也是一个时间系统工程。

---

## 2. 本章目录结构

```text
23_NTCTime/
├── main/
│   └── main.c                     // 程序入口：联网 + 校时 + 显示时间
├── components/
│   ├── WIFISTA/
│   │   ├── wifista.c              // Wi-Fi STA 模块
│   │   └── wifista.h
│   ├── MYNTC/
│   │   ├── myntc.c                // SNTP 校时模块
│   │   └── myntc.h
│   ├── LCD/
│   ├── SPI/
│   └── LED/
└── notes.md
```

本章真正与"网络时间"直接相关的核心文件只有三个：

- `main.c`
- `myntc.c`
- `wifista.c`

---

## 3. 软件架构传递树

### 3.1 模块调用关系

```text
app_main()                                      [main/main.c]
  ├── nvs_flash_init()                          初始化 NVS
  ├── lcd_init()                                初始化 LCD
  ├── wifista_init()                            初始化 STA 并连接路由器
  │     ├── esp_netif_init()
  │     ├── esp_event_loop_create_default()
  │     ├── 注册 WIFI_EVENT / IP_EVENT 回调
  │     ├── esp_netif_create_default_wifi_sta()
  │     ├── esp_wifi_init()
  │     ├── esp_wifi_set_mode(WIFI_MODE_STA)
  │     ├── esp_wifi_set_config()
  │     └── esp_wifi_start()
  │
  ├── myntc_init()                              初始化 SNTP
  │     ├── esp_sntp_setoperatingmode(POLL)
  │     ├── esp_sntp_setservername(...)
  │     ├── esp_sntp_init()
  │     ├── setenv("TZ", "CST-8", 1)
  │     └── tzset()
  │
  └── while(1)
        ├── time(NULL)                          获取当前 Unix 时间戳
        ├── localtime(&timer)                   转换为本地时间结构体
        └── lcd_show_num()                      刷新年月日时分秒
```

### 3.2 时间同步与显示流程

```text
ESP32 上电
   │
   ▼
连接路由器（STA 模式）
   │
   ▼
SNTP 后台向 NTP 服务器请求标准时间
   │
   ▼
系统时间被校正为 UTC 时间
   │
   ▼
通过 TZ=CST-8 转为北京时间
   │
   ▼
time() 取得时间戳
   │
   ▼
localtime() 转成 struct tm
   │
   ▼
LCD 显示：年 月 日 时 分 秒
```

---

## 4. 代码讲解

### 4.1 `main/main.c` — 主程序入口

`main.c` 的核心顺序非常清晰：

```c
nvs_flash_init();
lcd_init();
wifista_init();
myntc_init();
```

顺序含义：

| 步骤 | 作用 |
|------|------|
| `nvs_flash_init()` | Wi-Fi 驱动依赖 NVS，必须放最前面 |
| `lcd_init()` | 初始化显示硬件 |
| `wifista_init()` | 先把网络打通，否则没有办法访问 NTP 服务器 |
| `myntc_init()` | 联网后启动 SNTP，并设置时区 |

然后程序在 `while(1)` 中每秒做一次：

```c
timer = time(NULL);
local_time = localtime(&timer);
```

再从 `local_time` 中取出：

- `tm_year + 1900`
- `tm_mon + 1`
- `tm_mday`
- `tm_hour`
- `tm_min`
- `tm_sec`

分别显示到 LCD 上。

这里有两个细节：

1. `tm_year` 不是实际年份，而是"从 1900 年开始的偏移量"，所以要加 `1900`；
2. `tm_mon` 取值范围是 `0~11`，所以显示月份时要加 `1`。

### 4.2 `components/MYNTC/myntc.c` — SNTP 校时模块

`myntc.c` 是本章最关键的新增模块。

```c
esp_sntp_setoperatingmode(ESP_SNTP_OPMODE_POLL);
esp_sntp_setservername(0, "pool.ntp.org");
esp_sntp_setservername(1, "cn.pool.ntp.org");
esp_sntp_setservername(2, "ntp1.aliyun.com");
esp_sntp_init();
setenv("TZ", "CST-8", 1);
tzset();
```

逐项解释：

| 代码 | 含义 |
|------|------|
| `ESP_SNTP_OPMODE_POLL` | 轮询模式，SNTP 客户端会定期发请求校时 |
| `pool.ntp.org` | 国际公共 NTP 服务器池 |
| `cn.pool.ntp.org` | 中国区域公共 NTP 池 |
| `ntp1.aliyun.com` | 阿里云 NTP 服务器 |
| `esp_sntp_init()` | 真正启动 SNTP 后台任务 |
| `setenv("TZ", "CST-8", 1)` | 设置本地时区为北京时间 |
| `tzset()` | 通知 C 库重新加载时区参数 |

这里的设计思路很实用：

- 配置多个 NTP 服务器，提高可用性；
- 先拿到标准 UTC 时间；
- 再用时区规则把 UTC 转成中国本地时间。

### 4.3 `components/WIFISTA/wifista.c` — STA 联网模块

`wifista.c` 负责给 SNTP 提供网络基础。

和前面 STA 工程相比，这里最大的区别是：

- 保留了 `STA_CONNECTED`、`STA_DISCONNECTED`、`GOT_IP` 的处理框架；
- 但 LCD 上显示连接状态和 IP 的代码被注释掉了；
- 这说明本工程的展示重点已经从"联网状态"切换成了"时间显示"。

也就是说，它保留了联网能力，但把 LCD 显示资源让给了时间内容。

### 4.4 为什么 `myntc_init()` 必须放在 `wifista_init()` 后面

因为 SNTP 本质上是一个网络客户端。

它要做的事情是：

```text
ESP32 → 发送 UDP/NTP 请求 → NTP 服务器 → 返回标准时间
```

如果没有先连接路由器：

- 没有 IP 地址；
- 没有默认网关；
- 没法解析域名；
- 也就无法访问 `pool.ntp.org` 之类的服务器。

所以顺序必须是：

```text
先联网
  ↓
再校时
```

但需要注意：`myntc_init()` 调用时，STA 可能还没有拿到 IP（因为 Wi-Fi 连接和 DHCP 是异步的）。不过 SNTP 本身有重试机制——它启动后会定期轮询，第一次失败没关系，等网络就绪后下一次轮询就会成功。这就是为什么 `myntc_init()` 可以放在 `wifista_init()` 之后直接调用，而不需要等到 `IP_EVENT_STA_GOT_IP` 事件。

### 4.5 为什么先设置 `TZ` 再用 `localtime()`

NTP 返回的是标准 UTC 时间，不带"中国时间"概念。

例如：

- UTC：`2026-07-15 00:00:00`
- 北京时间：`2026-07-15 08:00:00`

两者相差 8 小时。

`localtime()` 的工作逻辑是：

1. 先读取系统时间戳；
2. 再根据当前时区规则，转换为本地时间结构体。

因此如果不先设置：

```c
setenv("TZ", "CST-8", 1);
tzset();
```

那么 `localtime()` 返回的结果往往就是 UTC，而不是北京时间。

---

## 5. 时序流程图

### 5.1 上电到时间显示的完整时序

```text
app_main()
   │
   ├── nvs_flash_init()
   ├── lcd_init()
   ├── wifista_init()
   │      │
   │      ├── esp_wifi_start()
   │      ├── WIFI_EVENT_STA_START
   │      ├── esp_wifi_connect()
   │      ├── WIFI_EVENT_STA_CONNECTED
   │      └── IP_EVENT_STA_GOT_IP
   │
   ├── myntc_init()
   │      ├── 配置 NTP 服务器
   │      ├── esp_sntp_init()
   │      ├── 设置 TZ=CST-8
   │      └── tzset()
   │
   └── while(1)
          ├── time(NULL)
          ├── localtime(&timer)
          ├── 刷新 LCD 日期
          ├── 刷新 LCD 时间
          └── vTaskDelay(1000)
```

---

## 6. NTP 校时的本质

NTP（Network Time Protocol）本质上就是：

```text
设备本地没有准确时钟
   ↓
向公网时间服务器询问"现在几点"
   ↓
服务器返回标准时间
   ↓
设备校正自己的系统时间
```

SNTP（Simple NTP）是 NTP 的简化实现，特别适合嵌入式设备。

它的优势是：

- 协议开销小；
- 实现简单；
- 精度足够大多数物联网项目使用。

ESP32 中常见应用包括：

- 网络时钟
- 日志带时间戳
- 定时上报
- 证书有效期校验
- OTA 校验时间

---

## 容易踩坑

| 现象 | 常见原因 | 排查方向 |
|------|----------|----------|
| 时间一直是 1970 年 | 还没成功联网，或 SNTP 尚未校时完成 | 先确认 STA 已拿到 IP，再给 SNTP 留出同步时间 |
| 时间差 8 小时 | 没设置 `TZ` 或没调用 `tzset()` | 检查 `setenv("TZ", "CST-8", 1)` 与 `tzset()` 是否执行 |
| 年份显示异常 | 忘了 `tm_year + 1900` | `tm_year` 不是自然年份 |
| 月份显示少 1 | 忘了 `tm_mon + 1` | `tm_mon` 范围是 0~11 |
| 工程明明联网了，但 LCD 不显示 IP | 本工程刻意把显示 IP 的代码注释掉了 | 这是设计选择，不是 bug |
| NTP 域名访问失败 | DNS、网关或外网不通 | 优先检查路由器联网状态和 STA 配置 |

---

## 7. 本章小结

本章真正建立的是一条完整链路：

```text
Wi-Fi 联网
   ↓
SNTP 网络校时
   ↓
系统时间更新
   ↓
UTC 转本地时区
   ↓
localtime() 分解成年月日时分秒
   ↓
LCD 实时显示
```

从工程角度看，`23_NTCTime` 的重点不是"LCD 显示"，而是 **ESP32 如何把网络时间引入本地系统时间**。

掌握这一章之后，后续凡是和"当前时间"相关的功能，例如：

- 定时采样
- 日志时间戳
- 到点执行任务
- 联网设备时间同步

都已经有了可靠基础。

---

## Tips

### 本章最易忽视的细节

1. **SNTP 默认不会等待校时完成才返回**：`esp_sntp_init()` 是非阻塞的——它只是创建了一个后台任务，并不保证调用后系统时间就已经校准了。如果你在 `myntc_init()` 之后立刻 `time(NULL)`，大概率拿到的是 1970 年（Unix epoch）。正确的做法是：要么在主循环中检查 `time(NULL) > 1000000000`（大约是 2001 年的阈值），要么用 SNTP 的通知回调来判断校时是否完成。
2. **`tm_year` 和 `tm_mon` 的偏移量不是 bug，是 POSIX 规范**：`localtime()` 返回的 `struct tm` 遵循 POSIX 标准，`tm_year` 是从 1900 年起的年数，`tm_mon` 是 0-11。这不是 C 库的 bug，所有 Unix 系统都是这个约定。
3. **LCD 显示时间的位置可能不够**：年份最多 4 位（如 2026），月份和日期各 2 位，时分秒各 2 位。合计至少需要 16 个字符宽度。如果 LCD 只有 16 列，刚好塞满一行；如果是 8x2 的小屏幕，就需要分两行显示（上行日期、下行时间）。
4. **SNTP 的精度是毫秒级的，但不适合做毫秒级对时应用**：SNTP 使用单个 UDP 请求/响应，不测量网络往返延迟，精度通常在 10-100ms 范围内。如果需要微秒级精度（如金融交易、电网同步），必须用完整 NTP 协议，或者用 GPS/PTP。

### 踩坑记录

- **校时后 LCD 显示的时间偶尔跳变**：如果 SNTP 第一次校时在程序启动后 5 秒完成，系统时间从 1970 年跳到 2026 年，你的 LCD 会突然显示一个巨大的时间跳跃。这是正常的，但如果你在显示逻辑里做了"时间单调递增"的假设，可能会触发异常。解决方案是在 `time(NULL) < 某个合理阈值` 时显示"--:--:--"而不是实际时间。
- **SDK 配置中禁用了 SNTP**：ESP-IDF 的 `menuconfig` 中有一个 `LWIP_SNTP` 选项。如果它被禁用了，`esp_sntp_init()` 仍然可以编译通过，但 SNTP 功能不会真正运行。确认 `CONFIG_LWIP_SNTP=y`。
- **NTP 服务器不可达但程序不报错**：SNTP 在 UDP 层面没有"连接"的概念，它只是发送一个请求然后等回复。如果服务器不可达，请求包会在网络中丢失，SNTP 客户端不会收到任何错误回调——它只会静默失败。这就是为什么配置多个 NTP 服务器很重要。

### SNTP vs NTP 的区别

虽然日常交流中经常混用这两个词，但它们在协议层面是有明确区别的：

| 维度 | NTP（完整） | SNTP（简化） |
|------|-----------|-------------|
| 协议复杂性 | 复杂，有完整的状态机（8 种状态） | 简单，通常只有单次请求/响应 |
| 网络延迟补偿 | 有——通过多次交换测量往返时间并做统计滤波 | 无——假设网络延迟对称且很小 |
| 精度 | 微秒级（局域网）到毫秒级（广域网） | 10-100ms 级 |
| 时钟漂移补偿 | 有——持续跟踪本地时钟的频率误差 | 无——每次独立校正 |
| 服务器端 | 需要专门的 ntpd 守护进程 | 可与标准 NTP 服务器兼容 |
| 内存/代码开销 | 较大（几十 KB） | 较小（几 KB） |
| 适用场景 | 数据中心、金融、电信基站 | 嵌入式设备、IoT 传感器 |

ESP-IDF 使用的 `esp_sntp` 组件实现的是 SNTP 协议，它向标准 NTP 服务器发送 SNTP 格式的请求包（NTP 服务器同时兼容 NTP 和 SNTP 请求）。对于物联网设备而言，SNTP 的精度完全足够——你的传感器数据时间戳差 50ms 并不会影响数据分析结果。

### 时区设置 `TZ` 环境变量的格式

`setenv("TZ", "CST-8", 1)` 中的 `CST-8` 看起来很简单，但背后有一套完整的 POSIX TZ 格式规范。理解它对你后续处理夏令时、非整数时区等情况很有帮助。

**基本格式：**

```text
std offset [dst [offset] [,start[/time],end[/time]]]
```

**最简形式（本章使用）：**

```text
CST-8
│   │
│   └── UTC 偏移量（负数表示本地时间比 UTC 快）
│        即 UTC+8 = 北京时间
└────── 时区缩写（任意 3 个以上字母）
```

**解析规则（容易搞反）：**

`CST-8` 的 `-8` 表示"本地时间 = UTC - (-8) = UTC + 8"。POSIX TZ 的符号方向和你直觉中的 UTC 偏移相反：
- `CST-8` → 北京时间（UTC+8）
- `EST5` → 美国东部时间（UTC-5）
- `GMT0` → 格林威治时间（UTC+0）

**常用中国时区写法：**

| 写法 | 含义 |
|------|------|
| `CST-8` | 中国标准时间，UTC+8 |
| `Asia/Shanghai` | 部分嵌入式 C 库支持 IANA 时区名（ESP-IDF 默认 newlib 不支持） |
| `CST-8CST,M-10.5.0/2` | 不太适用，中国已取消夏令时 |

**带夏令时的示例（美国东部）：**

```c
setenv("TZ", "EST5EDT,M3.2.0/2,M11.1.0/2", 1);
// EST5: 标准时间 UTC-5
// EDT: 夏令时缩写
// M3.2.0/2: 夏令时开始（3月第2个周日凌晨2点）
// M11.1.0/2: 夏令时结束（11月第1个周日凌晨2点）
```

**如果 TZ 设置错误会怎样：**

- 只设置了 `setenv` 但忘了 `tzset()` → `localtime()` 仍然使用旧时区
- `TZ` 写成 `CST+8` → 时间会差 16 小时（因为 `+8` 表示 UTC-8，即太平洋时间）
- 写成了不存在的时区名 → C 库回退到 UTC（即 `time()` 和 `localtime()` 返回相同值）

建议在调试时打印 `time()` 和 `localtime()` 的结果对比，确认时区转换是否正确生效。

### 与后续章节的知识衔接

- **RTC 时钟**：ESP32-S3 内置 RTC 模块，但精度有限（晶振误差通常每天几秒）。SNTP 校时的价值在于定期纠正 RTC 的漂移。实际项目中，可以在 SNTP 校时完成后将时间写入 RTC，这样即使设备断网，也能维持一段时间的准确时间。
- **MQTT + 时间戳**：MQTT 消息中经常需要携带时间戳。有了本章的基础，你可以在 `while(1)` 中取 `time(NULL)` 作为消息的 `timestamp` 字段，发送给云平台。
- **TLS 证书校验**：TLS 证书有有效期，系统时间错误（如显示 1970 年）会导致证书校验失败，进而 HTTPS/MQTTS 连接失败。因此在做 TLS 连接之前，最好先确保 SNTP 校时完成。
- **定时任务**：有了可靠的系统时间，你可以用 FreeRTOS 软件定时器 + `time()` 做"每天 8 点上报数据"、"每小时记录日志"等定时任务。

### 设计决策思考：为什么 SNTP 使用轮询而不是事件驱动

你可能注意到，Wi-Fi 章节强调"事件驱动优于轮询"，但 SNTP 却用了 `ESP_SNTP_OPMODE_POLL`（轮询模式）。这不是矛盾，而是设计目标的差异：

Wi-Fi 事件（连接、断开、获取 IP）是**不可预测的异步事件**。如果用轮询，要么延迟大，要么功耗高。

SNTP 校时是**周期性任务**。即使有"时间已同步"的事件，你仍然需要定期重新校时（因为本地时钟会漂移）。SNTP 轮询模式的本质是"定时器驱动的周期性同步"，默认间隔是 1 小时。这和"在 while(1) 里死循环检查 Wi-Fi 状态"是两种不同的轮询：

```text
坏轮询：while(1) { if (wifi_connected()) ... }  → 高频无效检查
好轮询：每 1 小时发一次 NTP 请求                   → 低频有效同步
```

SNTP 还提供了 `ESP_SNTP_OPMODE_LISTENONLY`（仅监听广播/组播模式），但这依赖局域网内有 NTP 广播服务器，大多数家庭和办公网络没有这个服务，所以轮询模式是最通用的选择。
