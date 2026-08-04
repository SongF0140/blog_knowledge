---
title: "Wi-Fi STA 模式 — ESP32-S3 连接路由器并显示 IP"
published: 2026-08-26
description: "本章项目 20WIFISTA 是一个 ESP32-S3 以 STA（Station）模式连接路由器 的最小演示工程。"
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","无线通信","Wi-Fi","STA"]
category: 物联网
draft: false
---
# Wi-Fi STA 模式 — ESP32-S3 连接路由器并显示 IP

## 本章重要 API 函数

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `nvs_flash_init()` | 初始化 NVS 非易失存储，为 Wi-Fi 驱动提供参数存储区 | `void`：无参数；返回 `esp_err_t`，`ESP_OK` 表示成功 | `"nvs_flash.h"` |
| `esp_netif_init()` | 初始化 TCP/IP 网络接口抽象层 | `void`：无参数；返回 `esp_err_t` | `"esp_netif.h"` |
| `esp_event_loop_create_default()` | 创建默认事件循环 | `void`：无参数；返回 `esp_err_t` | `"esp_event.h"` |
| `esp_event_handler_register()` | 注册事件回调函数 | `esp_event_base_t event_base`：事件基类；`int32_t event_id`：事件 ID；`esp_event_handler_t event_handler`：回调函数；`void *event_handler_arg`：用户参数；返回 `esp_err_t` | `"esp_event.h"` |
| `esp_netif_create_default_wifi_sta()` | 创建默认 STA 网络接口实例 | `void`：无参数；返回 `esp_netif_t *` 指针 | `"esp_wifi.h"` |
| `esp_wifi_init()` | 初始化 Wi-Fi 驱动 | `const wifi_init_config_t *config`：Wi-Fi 初始化配置结构体指针；返回 `esp_err_t` | `"esp_wifi.h"` |
| `esp_wifi_set_mode()` | 设置 Wi-Fi 工作模式 | `wifi_mode_t mode`：如 `WIFI_MODE_STA`；返回 `esp_err_t` | `"esp_wifi.h"` |
| `esp_wifi_set_config()` | 设置 Wi-Fi 接口参数 | `wifi_interface_t interface`：接口类型；`wifi_config_t *conf`：Wi-Fi 配置结构体指针；返回 `esp_err_t` | `"esp_wifi.h"` |
| `esp_wifi_start()` | 启动 Wi-Fi 驱动 | `void`：无参数；返回 `esp_err_t` | `"esp_wifi.h"` |
| `esp_wifi_connect()` | 发起 STA 连接请求 | `void`：无参数；返回 `esp_err_t` | `"esp_wifi.h"` |

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数](#本章重要-api-函数)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. 本章目录结构](#2-本章目录结构)**
- **[3. 软件架构传递树](#3-软件架构传递树)**
  - [3.1 模块调用关系](#31-模块调用关系)
  - [3.2 Wi-Fi 事件流](#32-wi-fi-事件流)
- **[4. 代码讲解](#4-代码讲解)**
  - [4.1 `main/main.c` — 程序入口](#41-main/mainc-—-程序入口)
  - [4.2 `components/WIFISTA/wifista.h` — 账号配置头文件](#42-components/wifista/wifistah-—-账号配置头文件)
  - [4.3 `components/WIFISTA/wifista.c` — 事件驱动的 STA 模块](#43-components/wifista/wifistac-—-事件驱动的-sta-模块)
  - [4.4 为什么 Wi-Fi 必须先初始化 NVS](#44-为什么-wi-fi-必须先初始化-nvs)
  - [4.5 为什么获取 IP 要走 IP_EVENT，而不是 WIFI_EVENT_STA_CONNECTED](#45-为什么获取-ip-要走-ip_event而不是-wifi_event_sta_connected)
- **[5. 时序流程图](#5-时序流程图)**
  - [5.1 启动到联网的完整时序](#51-启动到联网的完整时序)
  - [5.2 断开连接后的处理逻辑](#52-断开连接后的处理逻辑)
- **[6. STA 模式的本质](#6-sta-模式的本质)**
- **[7. 与前一章 Wi-Fi 原理笔记的衔接](#7-与前一章-wi-fi-原理笔记的衔接)**
- **[容易踩坑](#容易踩坑)**
- **[8. 本章小结](#8-本章小结)**
- **[Tips](#tips)**

---

## 学习前后依赖

| 方向   | 内容                                                                                                             |
| ---- | -------------------------------------------------------------------------------------------------------------- |
| 前置基础 | `0-1_under_Loop_stu/wifi_notes.md` 中的 Wi-Fi 基础原理、STA/AP/STA+AP 概念、IP 地址与 DHCP 基础；以及 LCD 显示函数的基本使用方式            |
| 本章核心 | 掌握 ESP-IDF 中 STA 联网的最小工程框架：`NVS → esp_netif → event loop → Wi-Fi init → STA config → start → connect → got IP` |
| 后续承接 | 为后续 MQTT、HTTP、TCP/UDP、WebServer、OTA 等章节打基础。后面所有联网应用，本质上都建立在本章的 STA 成功拿到 IP 之上                                  |

---

## 1. 概述

本章项目 `20_WIFI_STA` 是一个 **ESP32-S3 以 STA（Station）模式连接路由器** 的最小演示工程。

程序启动后会完成三件事：

1. 初始化 NVS，为 Wi-Fi 驱动提供配置存储区；
2. 初始化 LCD 屏幕，用于显示联网状态；
3. 初始化 Wi-Fi STA 模块，连接指定路由器，并在获取到 IP 后把 IPv4 地址显示到 LCD 上。

这个工程的重点不在网络通信业务本身，而在于 **ESP-IDF 的 Wi-Fi 事件驱动框架**。它展示了一个标准的最小联网骨架，后续几乎所有联网项目都会以它为基础扩展。

---

## 2. 本章目录结构

本项目中与 Wi-Fi STA 相关的核心文件如下：

```text
20_WIFI_STA/
├── main/
│   └── main.c                     // 应用入口
├── components/
│   ├── WIFISTA/
│   │   ├── wifista.c              // STA 初始化与事件回调
│   │   └── wifista.h              // SSID / 密码宏与接口声明
│   ├── LCD/
│   │   ├── lcd.c                  // LCD 驱动
│   │   └── lcd.h
│   ├── SPI/
│   │   ├── spi.c                  // LCD 底层 SPI
│   │   └── spi.h
│   └── LED/
│       ├── led.c
│       └── led.h
└── notes.md                       // 本章笔记
```

其中本章最核心的代码只有两个位置：

- `main.c`
- `wifista.c`

LCD 相关代码只是把联网结果可视化显示出来，不改变 Wi-Fi 框架本身。

---

## 3. 软件架构传递树

### 3.1 模块调用关系

```text
app_main()                                      [main/main.c]
  ├── nvs_flash_init()                          [ESP-IDF: nvs_flash]
  │     └── 初始化 NVS 分区，供 Wi-Fi 驱动读写参数
  │
  ├── lcd_init()                                [components/LCD/lcd.c]
  │     └── 初始化 LCD 显示屏
  │
  ├── wifista_init()                            [components/WIFISTA/wifista.c]
  │     ├── esp_netif_init()                    初始化网络接口层
  │     ├── esp_event_loop_create_default()     创建默认事件循环
  │     ├── esp_event_handler_register()        注册 Wi-Fi / IP 事件回调
  │     ├── esp_netif_create_default_wifi_sta() 创建默认 STA 接口
  │     ├── esp_wifi_init()                     初始化 Wi-Fi 驱动
  │     ├── esp_wifi_set_mode(WIFI_MODE_STA)    设为 STA 模式
  │     ├── esp_wifi_set_config()               写入 SSID / 密码
  │     └── esp_wifi_start()                    启动 Wi-Fi 驱动
  │
  └── while(1)
        └── vTaskDelay(500)                     主任务空转等待事件
```

可以看出，本工程采用的是 **"主任务做初始化，后续工作交给事件回调"** 的结构。真正的联网动作不是在 `while(1)` 中轮询完成的，而是由 ESP-IDF 内部的事件系统推动的。

### 3.2 Wi-Fi 事件流

```text
esp_wifi_start()
   │
   ▼
WIFI_EVENT_STA_START
   │
   └── 回调中调用 esp_wifi_connect()
          │
          ▼
WIFI_EVENT_STA_CONNECTED
   │
   └── LCD 显示 "connected"
          │
          ▼
DHCP 获取 IP
          │
          ▼
IP_EVENT_STA_GOT_IP
   │
   └── 解析 event_data 中的 IP 地址
       并显示到 LCD 第二行
```

断开连接时则会走另一条分支：

```text
WIFI_EVENT_STA_DISCONNECTED
   │
   ├── LCD 显示 "disconnected"
   └── esp_wifi_stop()
```

---

## 4. 代码讲解

### 4.1 `main/main.c` — 程序入口

`main.c` 很短，但顺序非常关键：

```c
void app_main(void)
{
    nvs_flash_init();
    lcd_init();
    wifista_init();
    while (1)
    {
        vTaskDelay(500);
    }
}
```

执行顺序解释如下：

| 步骤 | 代码 | 作用 |
|------|------|------|
| 1 | `nvs_flash_init();` | 初始化 NVS。Wi-Fi 驱动会依赖它保存 PHY 校准信息、网络参数等 |
| 2 | `lcd_init();` | 初始化 LCD，保证后续事件回调里能直接显示联网状态 |
| 3 | `wifista_init();` | 初始化 STA 模块，创建事件循环并启动 Wi-Fi |
| 4 | `while(1) { vTaskDelay(500); }` | 主任务不轮询 Wi-Fi，仅保持任务存活，后续全部由事件回调驱动 |

这里体现了 ESP-IDF 的一个非常重要的风格：

- **初始化阶段在 `app_main()` 完成**；
- **运行阶段由驱动任务、事件循环、协议栈接管**；
- 用户主循环往往只是一个空闲等待，不负责轮询网络状态。

### 4.2 `components/WIFISTA/wifista.h` — 账号配置头文件

`wifista.h` 只定义了三个内容：

```c
#define DEFAULT_SSID "your_ssid"
#define DEFAULT_PWD  "your_password"

void wifista_init(void);
```

这说明当前项目把路由器账号信息直接硬编码进了头文件中。

优点：
- 示例工程简单直观；
- 不需要菜单配置或串口输入。

缺点：
- 账号密码直接写在源码里，不适合正式项目；
- 改 Wi-Fi 时必须重新编译固件；
- 安全性差，源码泄露就等于密码泄露。

本章作为教学示例是可以接受的，但后续正式项目更推荐：

- 用 `menuconfig` 配置；
- 或用 NVS 保存配网结果；
- 或通过 AP 配网页面动态写入。

### 4.3 `components/WIFISTA/wifista.c` — 事件驱动的 STA 模块

`wifista.c` 可以分成两部分：

1. **事件处理函数** `wifista_event_handler()`
2. **初始化函数** `wifista_init()`

#### 4.3.1 事件处理函数

```c
void wifista_event_handler(void* event_handler_arg,
                           esp_event_base_t event_base,
                           int32_t event_id,
                           void* event_data)
```

这是 ESP-IDF 标准事件回调签名。它不是专门给 Wi-Fi 用的，而是通用事件系统的统一接口。

参数含义：

| 参数 | 含义 |
|------|------|
| `event_handler_arg` | 用户自定义参数，本工程传入的是 `NULL` |
| `event_base` | 事件类型大类，如 `WIFI_EVENT`、`IP_EVENT` |
| `event_id` | 具体事件号，如 `WIFI_EVENT_STA_START`、`IP_EVENT_STA_GOT_IP` |
| `event_data` | 附加事件数据指针，不同事件类型指向不同结构体 |

#### 4.3.2 `WIFI_EVENT_STA_START`

```c
if(event_id == WIFI_EVENT_STA_START)
{
    esp_wifi_connect();
}
```

含义：Wi-Fi 驱动已经启动完成，此时可以发起连接请求。

这一步很关键：

- `esp_wifi_start()` 只是启动驱动；
- **并不会自动联网**；
- 真正去连路由器，需要再调用一次 `esp_wifi_connect()`。

因此，本工程把"启动后立刻连接"的动作写在 `STA_START` 事件回调里。

#### 4.3.3 `WIFI_EVENT_STA_CONNECTED`

```c
else if(event_id == WIFI_EVENT_STA_CONNECTED)
{
    lcd_show_string(1,1,"connected   ",YELLOW,BLACK);
}
```

这表示 **链路层已经成功关联到 AP**。换句话说：

- SSID 和密码正确；
- AP 接受了该 STA；
- 802.11 层面的连接已经建立。

但此时 **还没有 IP 地址**。DHCP 还没跑完，所以这里只显示 `connected`，而不能立刻显示 IP。

#### 4.3.4 `WIFI_EVENT_STA_DISCONNECTED`

```c
else if(event_id == WIFI_EVENT_STA_DISCONNECTED)
{
    lcd_show_string(1,1,"disconnected",YELLOW,BLACK);
    esp_wifi_stop();
}
```

表示 STA 已断开连接。

当前项目的处理策略是：

1. LCD 显示 `disconnected`；
2. 直接调用 `esp_wifi_stop()` 停止 Wi-Fi。

这是一种**演示型处理**，好处是逻辑简单，便于观察断连事件。

但它也意味着：
- 一旦掉线，程序不会自动重连；
- 后续网络功能全部停止；
- 需要手动重启系统才能重新联网（或者额外写重新启动逻辑）。

正式项目中更常见的写法是：

```c
esp_wifi_connect();
```

也就是断线后立即重连，而不是停掉 Wi-Fi 驱动。

#### 4.3.5 `IP_EVENT_STA_GOT_IP`

```c
if(event_id == IP_EVENT_STA_GOT_IP)
{
    esp_netif_ip_info_t *event = (esp_netif_ip_info_t *)event_data;
    lcd_show_num(2,1,esp_ip4_addr1_16(&event->ip),3,GREEN,BLACK);
    lcd_show_string(2,4,".",GREEN,BLACK);
    lcd_show_num(2,5,esp_ip4_addr2_16(&event->ip),3,GREEN,BLACK);
    lcd_show_string(2,8,".",GREEN,BLACK);
    lcd_show_num(2,9,esp_ip4_addr3_16(&event->ip),3,GREEN,BLACK);
    lcd_show_string(2,12,".",GREEN,BLACK);
    lcd_show_num(2,13,esp_ip4_addr4_16(&event->ip),3,GREEN,BLACK);
}
```

这一步表示 DHCP 已成功分配 IP，网络层就绪，可以真正进行 socket 通信了。

这里的 `event_data` 被强制转换为：

```c
esp_netif_ip_info_t *
```

它内部至少包含：

- `ip`：本机 IP 地址
- `netmask`：子网掩码
- `gw`：网关地址

本项目只取了 `ip` 字段，并用以下宏分段显示：

- `esp_ip4_addr1_16(&event->ip)`
- `esp_ip4_addr2_16(&event->ip)`
- `esp_ip4_addr3_16(&event->ip)`
- `esp_ip4_addr4_16(&event->ip)`

如果 DHCP 分到的地址是 `192.168.1.102`，LCD 上就会显示：

```text
192.168.1.102
```

#### 4.3.6 `wifista_init()` 初始化流程

`wifista_init()` 是整个 STA 模块的核心入口：

```c
void wifista_init()
{
    esp_netif_init();
    esp_event_loop_create_default();
    esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID,&wifista_event_handler, NULL);
    esp_event_handler_register(IP_EVENT,IP_EVENT_STA_GOT_IP,&wifista_event_handler,NULL);
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);

    esp_wifi_set_mode(WIFI_MODE_STA);

    wifi_config_t wifista_config = {
        .sta = {
            .ssid = DEFAULT_SSID,
            .password = DEFAULT_PWD,
        }
    };
    esp_wifi_set_config(WIFI_IF_STA, &wifista_config);

    esp_wifi_start();
}
```

可以把它拆成 6 步：

##### 第 1 步：初始化网络接口抽象层

```c
esp_netif_init();
```

作用：初始化 `esp_netif` 组件，它是 ESP-IDF 中对底层 lwIP 网络接口的抽象封装。

没有这一步，后续 STA 接口无法创建。

##### 第 2 步：创建默认事件循环

```c
esp_event_loop_create_default();
```

ESP-IDF 的 Wi-Fi、IP、以太网、OTA 等很多组件都依赖统一事件循环。

这一步相当于创建了一个"系统消息总线"：

- Wi-Fi 驱动发生状态变化时，把事件投递到这里；
- 用户注册的回调函数从这里接收消息。

##### 第 3 步：注册事件回调

```c
esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifista_event_handler, NULL);
esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifista_event_handler, NULL);
```

这里注册了两类事件：

| 注册项 | 含义 |
|--------|------|
| `WIFI_EVENT + ESP_EVENT_ANY_ID` | 所有 Wi-Fi 事件都交给同一个回调处理 |
| `IP_EVENT + IP_EVENT_STA_GOT_IP` | 只关心 STA 拿到 IP 这一类 IP 事件 |

这是一种非常常见的写法：

- Wi-Fi 层事件放在一起处理；
- IP 层事件单独处理；
- 在同一个回调里通过 `event_base` + `event_id` 分发。

##### 第 4 步：创建默认 STA 网络接口

```c
esp_netif_create_default_wifi_sta();
```

这一步会把：

- Wi-Fi 驱动
- TCP/IP 协议栈
- DHCP 客户端
- 默认网络接口对象

关联起来。

如果没有这一步，STA 虽然可能能连接 AP，但不会自动获得 IP，也不能形成完整的 TCP/IP 网络接口。

##### 第 5 步：初始化 Wi-Fi 驱动并写入配置

```c
wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
esp_wifi_init(&cfg);

esp_wifi_set_mode(WIFI_MODE_STA);
```

作用：

- 用默认参数初始化 Wi-Fi 驱动；
- 明确指定工作模式为 `STA`。

然后构造连接参数：

```c
wifi_config_t wifista_config = {
    .sta = {
        .ssid = DEFAULT_SSID,
        .password = DEFAULT_PWD,
    }
};
```

这是 ESP-IDF 标准的 Wi-Fi 参数结构体。此处只配置了最基本的：

- SSID
- 密码

然后应用到 STA 接口：

```c
esp_wifi_set_config(WIFI_IF_STA, &wifista_config);
```

##### 第 6 步：启动 Wi-Fi 驱动

```c
esp_wifi_start();
```

此时驱动开始工作，并在内部触发：

```text
WIFI_EVENT_STA_START
```

接着回调里再调用 `esp_wifi_connect()`，整个连接流程就启动起来了。

### 4.4 为什么 Wi-Fi 必须先初始化 NVS

在 `main.c` 中，`nvs_flash_init()` 被放在最前面，这不是随便写的顺序。

ESP-IDF 的 Wi-Fi 驱动依赖 NVS 保存和读取以下信息：

- PHY 校准数据
- 国家/信道配置
- 某些 Wi-Fi 驱动内部参数
- 后续若开启 Wi-Fi 存储模式，也会保存 AP 信息

因此，正确顺序必须是：

```text
NVS 初始化成功
   ↓
Wi-Fi 驱动初始化
   ↓
STA 配置与连接
```

如果跳过 NVS，常见结果包括：

- `esp_wifi_init()` 失败；
- Wi-Fi 行为异常；
- 获取不到正常的射频校准参数。

### 4.5 为什么获取 IP 要走 `IP_EVENT`，而不是 `WIFI_EVENT_STA_CONNECTED`

这是初学者最容易混淆的一点。

两者的区别如下：

| 事件 | 所属层次 | 含义 |
|------|----------|------|
| `WIFI_EVENT_STA_CONNECTED` | 802.11 链路层 | 已经连上 AP，认证与关联完成 |
| `IP_EVENT_STA_GOT_IP` | 网络层 | DHCP 成功，已经拿到 IP，可以进行 TCP/IP 通信 |

也就是说：

```text
先有 STA_CONNECTED
后有 GOT_IP
```

中间还隔着 DHCP 协议过程。

因此，正确的工程实践是：

- `STA_CONNECTED`：提示"已经连上路由器"
- `GOT_IP`：提示"现在真的能上网了"

本项目正是按照这个逻辑设计的。

---

## 5. 时序流程图

### 5.1 启动到联网的完整时序

```text
app_main()
   │
   ├── nvs_flash_init()
   │
   ├── lcd_init()
   │
   ├── wifista_init()
   │     │
   │     ├── esp_netif_init()
   │     ├── esp_event_loop_create_default()
   │     ├── 注册 Wi-Fi / IP 事件回调
   │     ├── esp_netif_create_default_wifi_sta()
   │     ├── esp_wifi_init()
   │     ├── esp_wifi_set_mode(WIFI_MODE_STA)
   │     ├── esp_wifi_set_config()
   │     └── esp_wifi_start()
   │            │
   │            ▼
   │      WIFI_EVENT_STA_START
   │            │
   │            └── esp_wifi_connect()
   │                     │
   │                     ▼
   │              WIFI_EVENT_STA_CONNECTED
   │                     │
   │                     └── LCD 第一行显示 "connected"
   │
   │              DHCP 自动申请地址
   │                     │
   │                     ▼
   │              IP_EVENT_STA_GOT_IP
   │                     │
   │                     └── LCD 第二行显示 IPv4 地址
   │
   └── while(1)
         └── vTaskDelay(500)
```

### 5.2 断开连接后的处理逻辑

```text
Wi-Fi 已连接
   │
   ▼
发生断开（路由器掉电 / 密码错误 / 信号太弱）
   │
   ▼
WIFI_EVENT_STA_DISCONNECTED
   │
   ├── LCD 显示 "disconnected"
   └── esp_wifi_stop()
          │
          └── Wi-Fi 驱动停止
```

这个项目没有做自动重连，因此它更像一个**一次性联网状态演示**，而不是可长期运行的稳定联网框架。

---

## 6. STA 模式的本质

结合上一章的理论笔记，本章项目中的 `STA` 模式本质上就是：

```text
ESP32 作为客户端（Station）
      │
      ├── 主动扫描周围路由器
      ├── 根据 SSID 找到目标 AP
      ├── 用密码完成认证与关联
      ├── 通过 DHCP 向路由器申请 IP
      └── 成为局域网中的一个普通终端设备
```

这意味着 ESP32 的角色和以下设备是同一类：

- 手机连家里 Wi-Fi
- 笔记本连办公室 Wi-Fi
- 平板连热点

都是 **STA**。

如果成功联网，ESP32 在网络中的地位就是：

```text
             ┌─────────┐
             │ 路由器   │  192.168.1.1
             └────┬────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
 ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
 │ 手机     │ │ 笔记本   │ │ ESP32   │
 │ .1.100  │ │ .1.101  │ │ .1.102  │
 └─────────┘ └─────────┘ └─────────┘
```

本章 LCD 显示的 IP，就是这个局域网地址。

---

## 7. 与前一章 Wi-Fi 原理笔记的衔接

`0-1_under_Loop_stu/wifi_notes.md` 讲的是 Wi-Fi 原理，而本章 `20_WIFI_STA` 则是把这些原理落到 ESP-IDF 实际代码中。

两者的映射关系如下：

| Wi-Fi 原理笔记中的概念 | 本章代码中的体现 |
|-----------------------|------------------|
| STA 是客户端模式 | `esp_wifi_set_mode(WIFI_MODE_STA)` |
| 客户端连接路由器 | `esp_wifi_connect()` |
| 连接成功不等于有 IP | `WIFI_EVENT_STA_CONNECTED` vs `IP_EVENT_STA_GOT_IP` |
| DHCP 自动获取 IP | `IP_EVENT_STA_GOT_IP` 中拿到 `esp_netif_ip_info_t` |
| 星型拓扑中所有终端挂到路由器 | LCD 显示的 IP 就是路由器分配给 ESP32 的局域网地址 |
| lwIP 提供 TCP/IP 协议栈 | `esp_netif_init()` 和默认 STA 接口创建在底层依赖 lwIP |

因此，这一章是前一章理论知识的**第一份落地代码**。

---

## 容易踩坑

| 现象 | 常见原因 | 排查方向 |
|------|----------|----------|
| 程序启动后 Wi-Fi 不工作 | 忘记先调用 `nvs_flash_init()` | 检查 `app_main()` 初始化顺序，NVS 必须在 Wi-Fi 前面 |
| 连接成功但没有 IP | 只看了 `WIFI_EVENT_STA_CONNECTED`，没处理 `IP_EVENT_STA_GOT_IP` | 区分链路层连接成功和 DHCP 完成这两个阶段 |
| 断线后永远不再联网 | 在 `DISCONNECTED` 中调用了 `esp_wifi_stop()`，没有重连逻辑 | 如果是正式项目，应改成 `esp_wifi_connect()` 自动重连 |
| 密码改了还连不上 | SSID/密码写死在头文件中，刷错固件或忘了改 | 检查 `wifista.h` 中的 `DEFAULT_SSID` 和 `DEFAULT_PWD` |
| 代码能编译但 LCD 没显示 IP | DHCP 没拿到地址，或回调没进入 `IP_EVENT_STA_GOT_IP` | 在串口日志中补 `printf` 或用断点确认事件是否触发 |
| 拿到 IP 后仍访问不了外网 | 只拿到了局域网地址，但 DNS / 网关异常 | 后续项目中应同时检查网关和 DNS，而不仅是 `ip` |
| 示例工程泄露 Wi-Fi 密码 | 账号密码直接写在源码宏里 | 教学可接受，正式项目应改为 NVS 配网或 `menuconfig` |

---

## 8. 本章小结

本章的核心不是"连上一个 Wi-Fi"这么简单，而是要建立 **ESP-IDF 联网工程的标准骨架**。

可以把整个项目浓缩成一句话：

```text
NVS 初始化
   ↓
网络接口初始化
   ↓
事件循环建立
   ↓
Wi-Fi 设为 STA 模式
   ↓
写入 SSID / 密码
   ↓
启动驱动
   ↓
在事件回调中连接 AP
   ↓
等待 DHCP 获取 IP
   ↓
网络层真正就绪
```

本章学会之后，你就已经具备了后续所有 ESP32 联网项目的地基。无论是：

- MQTT 上云
- HTTP 请求天气接口
- TCP Socket 客户端
- WebSocket 实时通信
- OTA 在线升级

它们的第一步都不是业务代码，而是：

**先让 STA 成功拿到 IP。**

---

## Tips

### 本章最易忽视的细节

1. **`esp_wifi_start()` 不会自动联网**：很多初学者以为调了 `esp_wifi_start()` 就等于连上 Wi-Fi 了。实际上这只是启动驱动，真正连接需要额外调用 `esp_wifi_connect()`。最佳实践是在 `WIFI_EVENT_STA_START` 回调中调用它——因为这时驱动才真正就绪。
2. **`CONNECTED` 之后不能立刻发 HTTP 请求**：必须等 `IP_EVENT_STA_GOT_IP`。即便 `CONNECTED` 事件触发了，DHCP 可能还需要几百毫秒到几秒才能完成。如果在这期间发起 TCP 连接，socket 会因为没有 IP 而失败。
3. **回调函数签名不能写错**：事件回调必须是 `void handler(void*, esp_event_base_t, int32_t, void*)` 这四个参数，顺序也不能错。如果写错了签名，编译不会报错（因为函数指针强转），但运行时 `event_data` 指向的地址会完全错乱。
4. **`ESP_EVENT_ANY_ID` 意味着所有 Wi-Fi 事件都进同一个回调**：你必须在回调里用 `if-else` 或 `switch` 按 `event_id` 分发。如果忘了判断 `event_id`，可能会导致一个事件的处理逻辑被错误地执行到另一个事件上。

### 踩坑记录

- **STA 连上了但 `IP_EVENT_STA_GOT_IP` 不触发**：如果网段内有多个 DHCP Server，或者路由器 DHCP 池已满，ESP32 可能一直拿不到 IP。用串口工具看日志，Wi-Fi 驱动通常会打印 DHCP 超时信息。
- **回调里调用阻塞函数导致系统卡死**：事件回调是运行在事件循环任务中的，如果你在回调里写 `vTaskDelay(5000)` 或死等某个条件，事件循环会被堵住，后续所有 Wi-Fi/IP 事件都无法处理。事件回调必须尽可能快、不阻塞。
- **密码错误时事件流完全不同**：如果密码错误，事件流是 `STA_START → STA_DISCONNECTED`，中间不会有 `STA_CONNECTED`。Wi-Fi 驱动默认会重试几次再放弃，所以你会看到多次 DISCONNECTED 事件。

### 事件驱动模型的设计原因（为什么不用轮询）

初学者有时候会问：为什么不在 `while(1)` 里轮询 Wi-Fi 状态？像这样：

```c
while(1) {
    if (wifi_is_connected()) { ... }
    vTaskDelay(100);
}
```

ESP-IDF 选择事件驱动而不是轮询的原因有三个层次：

**层次一：功耗。** Wi-Fi 芯片（不管是内置还是外置）查询状态的寄存器操作本身就会唤醒芯片，频繁轮询意味着射频部分永远不能真正休眠。事件驱动下，没有事件时 Wi-Fi 子系统可以进入省电模式。

**层次二：实时性。** Wi-Fi 事件的发生时机不可预测——连接成功可能在 100ms 后，也可能在 5 秒后。轮询周期如果设太长（比如 2 秒），事件响应延迟也至少 2 秒；设太短（比如 10ms），CPU 大部分时间都在做无效轮询。事件驱动在事件发生的那一刻回调，延迟接近 0。

**层次三：数据完整性。** `IP_EVENT_STA_GOT_IP` 携带的 `esp_netif_ip_info_t` 数据结构是瞬时的——如果你不在回调里立刻读取，这个指针指向的内存可能已经被协议栈回收，之后轮询时读到的就是垃圾数据。

所以事件驱动不是"一种风格选择"，而是 **实时性与低功耗的双重需求** 共同决定的架构选择。

### WIFI_EVENT vs IP_EVENT 的层次边界

这是需要牢记的一张分层地图：

```text
        ┌──────────────────────────┐
        │      应用层（HTTP/MQTT）   │ ← 你的业务代码
        ├──────────────────────────┤
        │      传输层（TCP/UDP）     │
        ├──────────────────────────┤
        │   IP_EVENT               │ ← 网络层事件
        │   - STA_GOT_IP           │    DHCP 完成，拿到 IP
        │   - STA_LOST_IP          │    IP 地址失效
        ├──────────────────────────┤
        │   WIFI_EVENT             │ ← 链路层事件
        │   - STA_START            │    驱动启动完成
        │   - STA_CONNECTED        │    802.11 关联成功
        │   - STA_DISCONNECTED     │    断开关联
        ├──────────────────────────┤
        │   硬件层（射频/PHY）       │
        └──────────────────────────┘
```

记住这条规则：
- **WIFI_EVENT 告诉你"射频连上了没"**
- **IP_EVENT 告诉你"网络通了没"**

两者之间隔着 DHCP 协议、ARP 解析、可能还有 802.1X 认证。所以在回调里要分层判断——先看 `event_base` 是 `WIFI_EVENT` 还是 `IP_EVENT`，再看具体 `event_id`。

### 与后续章节的知识衔接

STA 模式是 ESP32 联网的第一步，后续章节的依赖关系如下：

- **Wi-Fi AP 模式**（第 22 章）：与 STA 模式共用同一套 `esp_netif`/`esp_event` 框架，只是角色互换。理解 STA 的事件驱动模型后，AP 模式的事件处理几乎可以照搬。
- **NTP 校时**（第 23 章）：必须在 STA 拿到 IP 之后才能初始化 SNTP 客户端，因为 SNTP 需要向公网 NTP 服务器发 UDP 包。本章的 `IP_EVENT_STA_GOT_IP` 就是 NTP 章节的启动前置条件。
- **MQTT / HTTP**：这两类应用层协议都建立在 TCP 之上，而 TCP 连接的前提是 STA 已有有效 IP。因此它们的初始化代码都应该放在 `IP_EVENT_STA_GOT_IP` 回调中，而不是 `app_main()` 的尾部。

### 设计决策思考：`WIFI_INIT_CONFIG_DEFAULT()` 做了什么

很多教程直接写 `WIFI_INIT_CONFIG_DEFAULT()` 就过去了，但它背后实际上配置了：

- Wi-Fi 任务的核心绑定（通常默认不绑定）
- Wi-Fi 任务的优先级
- Wi-Fi 任务的栈大小
- 射频校准数据的存储方式
- NVS 中 Wi-Fi 参数的命名空间

对于大多数入门项目，默认参数足够。但如果你遇到以下情况就需要自定义：
- **大量并发 TCP 连接**：Wi-Fi 任务栈可能不够，需要调大
- **实时性要求高**：需要提高 Wi-Fi 任务的优先级
- **省电敏感场景**：可能需要配置省电模式参数

这些都可以通过自定义 `wifi_init_config_t` 来实现，而不是永远用默认值。
