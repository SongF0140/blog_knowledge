---
title: "FreeRTOS 多任务 — 任务创建/删除、时间片调度与延时必要性"
published: 2026-08-30
description: "本章项目 FREERTOS/1MultiTask 是一个 双任务并行 的最小工程："
tags: ["物联网","ESP32-S3","ESP-IDF","嵌入式","FreeRTOS","多任务"]
category: 物联网
draft: false
---
# FreeRTOS 多任务 — 任务创建/删除、时间片调度与延时必要性

## 本章重要 API 函数

| API                         | 功能                | 参数说明                                                                                                                                                                                                                                      | 头文件                 |
| --------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| `xTaskCreatePinnedToCore()` | 创建任务并绑定到指定 CPU 核心 | `TaskFunction_t pvTaskCode`：任务函数；`const char *pcName`：任务名；`uint32_t usStackDepth`：栈大小（字）；`void *pvParameters`：参数；`UBaseType_t uxPriority`：优先级；`TaskHandle_t *pvCreatedTask`：任务句柄输出；`BaseType_t xCoreID`：核心号；返回 `BaseType_t`，`pdPASS` 表示成功 | `"freertos/task.h"` |
| `vTaskDelete()`             | 删除任务（包括自身）        | `TaskHandle_t xTaskToDelete`：要删除的任务句柄，传 `NULL` 表示删除自身；无返回值                                                                                                                                                                                | `"freertos/task.h"` |
| `vTaskDelay()`              | 任务阻塞延时            | `const TickType_t xTicksToDelay`：延时 tick 数；无返回值                                                                                                                                                                                           | `"freertos/task.h"` |
| `key_scan()`                | 按键扫描（自定义）         | `void`：无参数；返回 `uint8_t`，`0` 表示无按下，`1`/`2` 表示对应按键                                                                                                                                                                                          | `"key.h"`           |
| `gpio_toggle()`             | GPIO 电平翻转（自定义）    | `gpio_num_t gpio_num`：引脚号；无返回值                                                                                                                                                                                                            | `"LED.h"`           |

***

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数](#本章重要-api-函数)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. 本章目录结构](#2-本章目录结构)**
- **[3. 软件架构传递树](#3-软件架构传递树)**
  - [3.1 模块调用关系](#31-模块调用关系)
  - [3.2 双任务并行运行示意](#32-双任务并行运行示意)
- **[4. 任务创建与删除](#4-任务创建与删除)**
  - [4.1 `xTaskCreatePinnedToCore` 详解](#41-xtaskcreatepinnedtocore-详解)
  - [4.2 `vTaskDelete` 详解](#42-vtaskdelete-详解)
  - [4.3 任务句柄的作用](#43-任务句柄的作用)
- **[5. 时间片轮转任务调度机制](#5-时间片轮转任务调度机制)**
  - [5.1 调度器与任务状态](#51-调度器与任务状态)
  - [5.2 同优先级时间片轮转](#52-同优先级时间片轮转)
  - [5.3 不同优先级的抢占](#53-不同优先级的抢占)
- **[6. 为什么第 18 行要停顿](#6-为什么第-18-行要停顿)**
  - [6.1 让出 CPU 给调度器](#61-让出-cpu-给调度器)
  - [6.2 喂看门狗（Task Watchdog）](#62-喂看门狗task-watchdog)
  - [6.3 停顿时间与 Hz 的关系](#63-停顿时间与-hz-的关系)
- **[7. FreeRTOS 数据类型对照表与可移植性](#7-freertos-数据类型对照表与可移植性)**
  - [7.1 为什么要自定义类型](#71-为什么要自定义类型)
  - [7.2 常用数据类型对照表](#72-常用数据类型对照表)
  - [7.3 FreeRTOS 函数命名约定](#73-freertos-函数命名约定)
- **[8. 从上电到 app_main：启动原理](#8-从上电到-app_main启动原理)**
  - [8.1 对比 Windows 的 BIOS 启动](#81-对比-windows-的-bios-启动)
  - [8.2 ESP32-S3 的上电启动流程](#82-esp32-s3-的上电启动流程)
  - [8.3 Bootloader 的工作原理](#83-bootloader-的工作原理)
  - [8.4 FreeRTOS 调度器何时启动](#84-freertos-调度器何时启动)
- **[容易踩坑](#容易踩坑)**
- **[9. 本章小结](#9-本章小结)**
- **[Tips](#tips)**

***

## 学习前后依赖

| 方向   | 内容                                                                              |
| ---- | ------------------------------------------------------------------------------- |
| 前置基础 | `01_LED` 的 GPIO 配置与 `vTaskDelay()`、`04_Key_board` 的按键扫描思路                       |
| 本章核心 | 掌握 FreeRTOS 任务的创建与删除、同等优先级的时间片轮转调度、为什么任务循环中必须让出 CPU，以及 FreeRTOS 自定义数据类型与可移植性的关系 |
| 后续承接 | 任务优先级与抢占、队列/信号量/互斥量、任务通知，以及 FreeRTOS 其他章节的进阶同步机制                                |

***

## 1. 概述

本章项目 `FREERTOS/1_MultiTask` 是一个 **双任务并行** 的最小工程：

- `LED_task`：每隔 1 秒翻转一次 GPIO38 上的 LED；
- `key_task`：扫描两个按键，key1 翻转 GPIO39 上的 LED，key2 直接删除 `LED_task`。

它把"多任务"这件事讲清楚了：

1. 如何用 `xTaskCreatePinnedToCore()` 创建任务；
2. 如何用 `vTaskDelete()` 在运行时删除任务；
3. 同等优先级的两个任务是如何被 CPU 轮流执行的；
4. 为什么每个任务循环里都必须有 `vTaskDelay()`，否则系统会被"饿死"甚至触发看门狗复位。

***

## 2. 本章目录结构

```text
FREERTOS/1_MultiTask/
├── main/
│   └── main.c                     // 双任务入口
└── components/
    ├── LED/
    │   ├── LED.c                  // LED 初始化与电平翻转
    │   └── LED.h
    └── KEY/
        ├── key.c                  // 按键初始化与扫描
        └── key.h
```

核心文件是 `main.c`，两个任务函数都写在这里。

***

## 3. 软件架构传递树

### 3.1 模块调用关系

```text
app_main()                                  [main/main.c]
  ├── LED_init()                            [components/LED/LED.c]
  ├── key_init()                            [components/KEY/key.c]
  │
  ├── xTaskCreatePinnedToCore(LED_task,...)  创建 LED 任务 → core 0
  └── xTaskCreatePinnedToCore(key_task,...)  创建 key 任务 → core 0

         ┌──────── CPU core 0 ────────┐
         │                            │
   LED_task()                    key_task()
   while(1)                      while(1)
   ├ gpio_toggle(38)             ├ key_scan()
   └ vTaskDelay(100)             │  ├ key1 → gpio_toggle(39)
                                 │  └ key2 → vTaskDelete(LED_task)
                                 └ vTaskDelay(5)
```

### 3.2 双任务并行运行示意

```text
时间 ──────────────────────────────────►

LED_task  ████░░░░░░░░░░░░████░░░░░░░░░░░░████
          toggle  delay(100)   delay(100)

key_task  ░░░░██████████████░░░░██████████████░░
          scan  delay(5)        scan  delay(5)

          █ = 运行    ░ = 阻塞/让出
```

两个任务优先级都是 1，都绑定在 core 0 上，所以它们是靠 **时间片轮转** 交替运行的。谁处于阻塞态，CPU 就去跑另一个。

***

## 4. 任务创建与删除

### 4.1 `xTaskCreatePinnedToCore` 详解

`main.c` 中的创建调用：

```c
xTaskCreatePinnedToCore(LED_task,"LED_task",1024,NULL,1,&LED_task_handle,0);
xTaskCreatePinnedToCore(key_task,"key_task",1024,NULL,1,&key_task_handle,0);
```

参数逐项对应：

| 位置 | 参数   | 值                  | 含义                                              |
| -- | ---- | ------------------ | ----------------------------------------------- |
| 1  | 任务函数 | `LED_task`         | 任务入口函数指针                                        |
| 2  | 任务名  | `"LED_task"`       | 调试可见的字符串名                                       |
| 3  | 栈大小  | `1024`             | 单位是 **字**（word），ESP32-S3 上 1 字 = 4 字节，即 4096 字节 |
| 4  | 参数   | `NULL`             | 传给任务函数的 `args`                                  |
| 5  | 优先级  | `1`                | 数值越大优先级越高                                       |
| 6  | 句柄输出 | `&LED_task_handle` | 创建后用来操作该任务                                      |
| 7  | 核心号  | `0`                | 绑定到 CPU0                                        |

这里两个关键点：

1. **栈大小单位是字不是字节**：`1024` 对应 4KB，FreeRTOS 在 ESP32 上按 32 位字计算。
2. **`xCoreID = 0`**：ESP32-S3 是双核，本工程把两个任务都钉在 core 0，避免跨核调度带来的初学困惑。可填 `0`、`1` 或 `tskNO_AFFINITY`（不绑定）。

### 4.2 `vTaskDelete` 详解

`key_task` 中：

```c
if(LED_task_handle != NULL)
{
    vTaskDelete(LED_task_handle);
    LED_task_handle = NULL;
    led_flag = false;
}
```

含义：

- `vTaskDelete(LED_task_handle)` 把 LED 任务从调度器中移除，释放它占用的栈和 TCB（任务控制块）；
- 紧接着 `LED_task_handle = NULL`，防止下次按键再次进入这段代码时对一个已经无效的句柄调用删除；
- `led_flag = false` 用来标记 LED 已经不再运行（虽然本工程后续没有用到它）。

删除任务有两种情况：

| 调用方式                | 行为                      |
| ------------------- | ----------------------- |
| `vTaskDelete(某句柄)`  | 删除指定任务，由 IDLE 任务负责回收栈内存 |
| `vTaskDelete(NULL)` | 删除调用者自身                 |

注意：如果被删除任务使用的栈是 **动态分配** 的，FreeRTOS 会安排 IDLE 任务去释放它。所以删完任务后不要立刻进死循环不让 IDLE 跑，否则那块内存不会被回收。

### 4.3 任务句柄的作用

`main.c` 定义了两个全局句柄：

```c
TaskHandle_t LED_task_handle = NULL;
TaskHandle_t key_task_handle = NULL;
```

句柄的本质是任务控制块（TCB）的指针。有了它，上层才能对任务做：

- 删除：`vTaskDelete(handle)`
- 挂起：`vTaskSuspend(handle)`
- 恢复：`vTaskResume(handle)`
- 改优先级：`vTaskPrioritySet(handle, ...)`

初始化为 `NULL` 是一个好习惯，这样在任务还没被创建前，任何对句柄的判空都能安全跳过。

***

## 5. 时间片轮转任务调度机制

### 5.1 调度器与任务状态

FreeRTOS 的调度器在 `vTaskStartScheduler()` 后启动。在 ESP-IDF 里，调度器其实是在调用 `app_main()` **之前**就已经启动了（详见第 8.4 节）。任务有四种状态：

```text
        创建
         │
         ▼
      ┌──────┐
      │ Ready│◄─────────────────┐
      └──┬───┘                  │
         │ 被调度选中             │ 超时/事件到达
         ▼                      │
      ┌──────┐   vTaskDelay() ┌───────┐
      │Running│──────────────►│Blocked│
      └──────┘                └───────┘
         │                      │
         │ vTaskSuspend()       │
         ▼                      │
      ┌────────┐                │
      │Suspended│───────────────┘
      └────────┘  vTaskResume()
```

本工程里：

- `LED_task` 在 `vTaskDelay(100)` 期间处于 **Blocked**；
- `key_task` 在 `vTaskDelay(5)` 期间处于 **Blocked**；
- 谁不阻塞，谁就 Ready，被调度成 Running。

### 5.2 同优先级时间片轮转

本工程两个任务优先级都是 1。当多个任务 **同等优先级** 且都处于 Ready 时，FreeRTOS 用 **时间片轮转（Round-Robin）** 调度：

```text
tick 中断（每 1 tick 一次）
        │
        ▼
   检查同优先级 Ready 队列
        │
        ▼
   切到队列下一个任务
```

也就是说，即使两个任务都不主动 `vTaskDelay()`，它们也会在每个 tick 边界被轮流切换。但实际工程中绝不会这样写，原因见第 6 节。

### 5.3 不同优先级的抢占

如果两个任务优先级不同，调度规则是 **抢占式**：

```text
高优先级任务 Ready
        │
        ▼
   立即抢占低优先级任务
        │
        ▼
   低优先级任务被"挤"回 Ready
```

也就是说：高优先级任务只要不阻塞，低优先级任务就得不到 CPU。这也是为什么设计任务优先级时，**耗时任务要给低优先级，并且必须主动 delay**，否则会饿死其他任务。

本工程两个任务同级，所以主要体现的是时间片轮转，而不是抢占。

***

## 6. 为什么第 18 行要停顿

`main.c#L18` 这一行：

```c
vTaskDelay(100);
```

是本章最容易被初学者忽略、却又最关键的一行。它的作用不是"让 LED 闪慢一点"，而是有三层必要性。

### 6.1 让出 CPU 给调度器

如果不写 `vTaskDelay()`，`LED_task` 会变成：

```c
while(1)
{
    gpio_toggle(GPIO_NUM_38);   // 一直翻转
}
```

此时任务永远处于 Running 态，**永远不让出 CPU**。虽然时间片轮转会在 tick 边界切换到 `key_task`，但：

- 如果两个任务优先级不同，低优先级任务会被彻底饿死；
- 即便同级，任务本身也会一直占满一个核心，白白浪费功耗。

`vTaskDelay(100)` 把当前任务放进 Blocked 队列，调度器立刻切到其他 Ready 任务。这是"让出 CPU"最直接的方式。

### 6.2 喂看门狗（Task Watchdog）

ESP-IDF 默认开启 **Task Watchdog Timer（TWDT）**。它的工作原理是：

```text
TWDT 启动后倒计时
        │
        ▼
   需要 IDLE 任务定期"喂狗"
        │
        ├── IDLE 跑到了 → 喂狗 → 复位计数器
        └── IDLE 长期没跑到 → 超时 → 触发系统复位
```

关键在于：**IDLE 任务优先级最低（0）**。如果某个应用任务一直不让出 CPU，IDLE 任务就永远得不到运行，看门狗没人喂，系统就会重启。

`vTaskDelay(100)` 让 `LED_task` 阻塞 1 秒，在这 1 秒里 CPU 空闲，IDLE 任务得以运行并喂狗。所以这一行同时也是在"保命"。

> 这也是为什么 `key_task` 里也写了 `vTaskDelay(5)`，哪怕它只有 50ms——同样是为了把 CPU 让给 IDLE 和其他任务。

### 6.3 停顿时间与 Hz 的关系

`vTaskDelay(100)` 的 `100` 不是毫秒，而是 **tick 数**。一个 tick 等于多少毫秒，由 `configTICK_RATE_HZ` 决定：

```text
1 tick = 1000ms / configTICK_RATE_HZ
```

ESP-IDF 默认 `configTICK_RATE_HZ = 100`，也就是 1 tick = 10ms。所以：

| 调用                | tick 数 | 实际延时        |
| ----------------- | ------ | ----------- |
| `vTaskDelay(100)` | 100    | 1000ms = 1s |
| `vTaskDelay(5)`   | 5      | 50ms        |
| `vTaskDelay(20)`  | 20     | 200ms       |

这就是为什么"停顿时间和 Hz 有关"：

- 如果把 `configTICK_RATE_HZ` 改成 `1000`，同样 `vTaskDelay(100)` 就只剩 100ms；
- 如果改成 `50`，`vTaskDelay(100)` 就变成 2 秒。

为了代码可读性，ESP-IDF 推荐用宏：

```c
vTaskDelay(pdMS_TO_TICKS(1000));   // 明确表示延时 1000ms
```

这样无论 tick 频率怎么改，延时时间都是 1 秒。本工程直接写 `vTaskDelay(100)` 是因为 tick 频率恰好是 100Hz，能对上，但可移植性不如 `pdMS_TO_TICKS()`。

***

## 7. FreeRTOS 数据类型对照表与可移植性

### 7.1 为什么要自定义类型

在 `main.c` 里能看到这些类型：

```c
TaskHandle_t LED_task_handle = NULL;
```

而 `main.c#L52` 的函数返回值是 `BaseType_t`。

FreeRTOS 不直接用 `int`、`unsigned int`、`uint16_t`，而是自己 typedef 了一层类型。原因只有一个：**可移植性**。

不同 MCU 的 C 编译器对原生类型的位宽定义不一样：

```text
8 位 MCU（如 8051）：int = 16 位
32 位 MCU（如 ESP32-S3）：int = 32 位
64 位 PC：long = 64 位
```

如果 FreeRTOS 内部直接写 `int`，那么同一份代码在不同 MCU 上行为可能不同。所以它通过每个移植层（`portmacro.h`）把类型重新定义成"语义固定"的别名：

- `TickType_t`：能装下 tick 计数的无符号类型；
- `BaseType_t`：该架构最高效的整数类型；
- `UBaseType_t`：无符号版。

这样 FreeRTOS 内核代码完全不用改，只要换一个 `portmacro.h` 就能跑在新的 MCU 上。这就是"自定义类型 = 可移植性"的本质。

### 7.2 常用数据类型对照表

| FreeRTOS 类型     | 典型底层类型（ESP32-S3）       | 用途                         | 为什么要包一层                  |
| --------------- | ---------------------- | -------------------------- | ------------------------ |
| `BaseType_t`    | `int`（32 位）            | 通用整数返回值（`pdPASS`/`pdFAIL`） | 保证返回值位宽与架构匹配             |
| `UBaseType_t`   | `unsigned int`         | 优先级、队列长度等无符号量              | 同上，无符号版                  |
| `TickType_t`    | `uint32_t`             | tick 计数、延时值                | 16 位 MCU 上可能是 `uint16_t` |
| `TaskHandle_t`  | `void *`               | 任务句柄                       | 屏蔽 TCB 结构体细节             |
| `QueueHandle_t` | `void *`               | 队列句柄                       | 同上                       |
| `StackType_t`   | `StackType_t`（通常 32 位） | 任务栈单元                      | 栈以"字"为单位，不同架构字长不同        |
| `portTickType`  | `TickType_t`           | 旧版别名                       | 兼容老代码                    |
| `portBASE_TYPE` | `BaseType_t`           | 旧版别名                       | 兼容老代码                    |

对应关系记忆方法：

```text
BaseType_t   ≈ int          （有符号通用整数）
UBaseType_t  ≈ unsigned int （无符号通用整数）
TickType_t   ≈ uint32_t     （装 tick 的计数器）
TaskHandle_t ≈ 指针         （任务身份）
```

所以在自己写任务代码时，也应该尽量用 `BaseType_t`、`TickType_t` 这类类型，而不是直接写 `int`。这样代码以后从 ESP32-S3 移植到 STM32、甚至 RP2040 上时，类型语义不会变。

### 7.3 FreeRTOS 函数命名约定

除了自定义类型，FreeRTOS 还有一套严格的函数命名前缀约定。初学者经常看到 `xTaskCreatePinnedToCore` 和 `vTaskDelay` 这种开头字母不一样的写法，其实 **前缀就暗示了返回值类型**。

**函数前缀与返回值的关系：**

| 前缀   | 含义                | 返回值               | 本章例子                           |
| ---- | ----------------- | ----------------- | ------------------------------ |
| `v`  | void              | 无返回值              | `vTaskDelay()`、`vTaskDelete()` |
| `x`  | 有返回值              | `BaseType_t` 或句柄等 | `xTaskCreatePinnedToCore()`    |
| `pv` | pointer to void   | 返回指针              | `pvPortMalloc()`               |
| `px` | pointer to struct | 返回结构体指针           | `pxTaskGetStackStart()`        |
| `ul` | unsigned long     | 返回 `uint32_t`     | `ulTaskNotifyTake()`           |
| `pc` | pointer to char   | 返回字符串指针           | `pcTaskGetName()`              |

所以看到函数名第一眼就能判断：

```text
vTaskDelete(...)            → v 开头 → 无返回值，直接调用即可
xTaskCreatePinnedToCore(... → x 开头 → 有返回值，必须检查
```

这就是为什么本章 `main.c` 里：

- `vTaskDelay(100)` 和 `vTaskDelete(LED_task_handle)` 都没接返回值；
- 而 `xTaskCreatePinnedToCore(...)` 严格来说应该检查返回值是否为 `pdPASS`（本工程省略了检查）。

**变量和宏也有前缀：**

| 前缀            | 含义                         | 例子                                           |
| ------------- | -------------------------- | -------------------------------------------- |
| `u` + `c/s/l` | unsigned + char/short/long | `uc`、`us`、`ul`                               |
| `x`           | BaseType\_t 或句柄类型          | `LED_task_handle` 是 `TaskHandle_t`           |
| `e`           | enum 枚举                    | `eTaskState`                                 |
| `pd`          | 宏（prefix define）           | `pdPASS`、`pdFAIL`、`pdMS_TO_TICKS()`、`pdTRUE` |

这套命名约定和上一节的自定义类型一样，都是为了让代码"见名知意、跨平台一致"。`v`/`x` 前缀让你不看文档就知道要不要接返回值，`pdPASS`/`pdFAIL` 让你不用记 `1`/`0` 哪个是成功。养成识别前缀的习惯，阅读 FreeRTOS 源码和 API 手册会快很多。

***

## 8. 从上电到 app_main：启动原理

前面几节讲的都是"任务创建之后"的事。但 ESP32-S3 从按下复位键到 `app_main()` 开始执行，中间还有一段完整的启动链路。把它和 Windows 的 BIOS 启动对比着看，会清晰很多。

### 8.1 对比 Windows 的 BIOS 启动

Windows PC 的传统 BIOS 启动流程：

```text
上电
 │
 ▼
CPU 从 ROM 地址 0xFFFF0 取指（BIOS）
 │
 ▼
POST 加电自检（CPU/内存/外设）
 │
 ▼
扫描启动设备，读第 1 扇区 MBR
 │
 ▼
MBR 引导代码加载 OS Bootloader
 │
 ▼
Bootloader 加载 OS 内核
 │
 ▼
内核初始化 → 进入用户态
```

UEFI 启动是它的现代版：把 MBR 换成 GPT + EFI 分区里的 `.efi` 程序，其余骨架一样。

这条链路的本质是 **多级接力**：

```text
ROM 固件 → 一级引导 → 二级引导 → 操作系统内核
```

每一级只负责把下一级加载进内存并跳过去，自己就功成身退。

### 8.2 ESP32-S3 的上电启动流程

ESP32-S3 的启动也是多级接力，但角色和 PC 不完全一样：

```text
上电
 │
 ▼
片内 ROM 固件执行（≈ PC 的 BIOS）
 │
 ▼
读取 Flash 0x8000 处的 bootloader.bin（≈ 二级引导）
 │
 ▼
bootloader 初始化硬件、读分区表
 │
 ▼
找到 app 分区，加载 app 代码
 │
 ▼
跳转到 app 入口 call_start_cpu0
 │
 ▼
系统级初始化 → 启动 FreeRTOS
 │
 ▼
main 任务里调用 app_main()
```

对应关系：

| 阶段   | Windows PC   | ESP32-S3              |
| ---- | ------------ | --------------------- |
| 第一级  | 主板 BIOS/UEFI | 片内 ROM 固件             |
| 第二级  | MBR / EFI 程序 | `bootloader.bin`      |
| 引导配置 | 分区表（GPT/MBR） | `partition-table.bin` |
| 第三级  | OS 内核        | 应用固件 `app.bin`        |
| 用户入口 | 用户态进程        | `app_main()`          |

最大区别在于：PC 的 BIOS 在主板的 ROM 里，和操作系统是分开的两套东西；而 ESP32-S3 的 ROM 固件是芯片出厂烧死的，`bootloader.bin` 和 `app.bin` 都在片外同一颗 Flash 里，由 ESP-IDF 一起编译、一起烧录。

### 8.3 Bootloader 的工作原理

`bootloader.bin` 是 ESP-IDF 编译时单独生成的一个镜像，默认烧在 Flash 的 `0x8000` 偏移处。它做的事很固定：

```text
从 ROM 接管 CPU
   │
   ├── 1. 初始化时钟树、内存、Flash 控制器
   │
   ├── 2. 读取 partition-table.bin
   │      （位于 0x8000 之后，描述各分区位置）
   │
   ├── 3. 找到标记为 app 的分区
   │
   ├── 4.（可选）校验 app 镜像签名/哈希
   │
   ├── 5. 把 app 的代码段映射/加载到内存
   │
   └── 6. 跳转到 app 入口，交出控制权
```

Flash 布局大致如下：

```text
0x0000  ┌─────────────────────┐
        │  bootloader.bin     │
0x8000  ├─────────────────────┤
        │  partition-table    │
        ├─────────────────────┤
        │  app 分区 (factory)  │
        │    app.bin          │
        ├─────────────────────┤
        │  nvs / spiffs / ... │
        └─────────────────────┘
```

bootloader 的设计意义：

1. **解耦硬件初始化和应用**：app 不用关心最底层时钟、Flash 时序，这些 bootloader 已经配好。
2. **支持 OTA 和多 app 分区**：bootloader 能根据分区表选择从哪个 app 启动，这是 OTA 升级的基础。
3. **支持安全启动**：可以校验 app 签名，防止跑被篡改的固件。

对初学者来说，只要记住一句话：**bootloader 负责把 app 从 Flash 搬进内存并跳过去，之后 CPU 就在跑你的 `app_main()` 了**。它和 app 是两个独立的 `.bin`，只是由 ESP-IDF 帮你一起编译和烧录。

### 8.4 FreeRTOS 调度器何时启动

这是初学者最容易搞混的一点：`app_main()` 里明明没有调用 `vTaskStartScheduler()`，为什么 `xTaskCreatePinnedToCore()` 创建的任务就能跑起来？

答案是：**调度器在 `app_main()` 之前就已经启动了**。

ESP-IDF 的启动代码（位于 `esp_system` 组件里）大致做了这些事：

```text
call_start_cpu0()           ← app 入口
   │
   ├── 系统级初始化
   │   （时钟、堆、中断、NVS 等）
   │
   ├── esp_startup_start_app()
   │     │
   │     ├── xTaskCreatePinnedToCore(main_task,...)
   │     │     创建 main 任务
   │     │
   │     └── vTaskStartScheduler()
   │           启动 FreeRTOS 调度器
   │
   ▼
调度器开始运行，选中 main 任务
   │
   ▼
main_task()
   │
   ├── 调用 app_main()      ← 你写的代码在这里
   │
   └── app_main 返回后
         vTaskDelete(NULL)   删除 main 任务自己
```

也就是说：

1. 启动代码先创建一个名叫 `main` 的任务；
2. 再调用 `vTaskStartScheduler()` 让 FreeRTOS 跑起来；
3. 调度器一启动，立刻切到 `main` 任务；
4. `main` 任务的函数体里调用 `app_main()`；
5. `app_main()` 返回后，`main` 任务把自己删掉。

所以你在 `app_main()` 里用 `xTaskCreatePinnedToCore()` 创建的任务，是直接注册到一个 **已经在运行** 的调度器里的。这也是为什么 `app_main()` 里不需要、也不应该再调用 `vTaskStartScheduler()`——再调一次会失败。

把这个链路和第 5 节的调度机制连起来看，整个多任务系统从上电到运行的完整图景就清楚了：

```text
ROM → bootloader → app 入口
   → 系统初始化 → 创建 main 任务
   → vTaskStartScheduler()
   → app_main() → 创建用户任务
   → 任务被调度运行
```

***

## 容易踩坑

| 现象                             | 常见原因                                          | 排查方向                               |
| ------------------------------ | --------------------------------------------- | ---------------------------------- |
| 任务函数写完不执行                      | 忘了 `xTaskCreatePinnedToCore()`，或返回值非 `pdPASS` | 检查创建返回值和栈大小是否够                     |
| 栈大小填 `1024` 以为是 1KB            | 单位是 **字**，ESP32-S3 上 1 字 = 4 字节               | `1024` 实际是 4KB                     |
| 删除任务后再按按键崩溃                    | 删完没把句柄置 `NULL`，下次又调用 `vTaskDelete`            | 参考本工程 `LED_task_handle = NULL` 的写法 |
| 程序跑一会儿自动重启                     | 任务循环里没 `vTaskDelay()`，IDLE 没机会喂看门狗            | 给每个任务循环加合理的延时                      |
| 同样 `vTaskDelay(100)` 在不同工程延时不同 | `configTICK_RATE_HZ` 不一样                      | 用 `pdMS_TO_TICKS(ms)` 替代裸数字        |
| 两个任务都不 delay，CPU 占用 100%       | 时间片轮转虽然会切换，但谁都不让出，IDLE 饿死                     | 至少给低优先级任务加 delay                   |
| 把任务钉在 core 0 后 Wi-Fi 异常        | Wi-Fi 任务通常在 core 1，核心分配冲突                     | 留意系统任务的核心占用，或用 `tskNO_AFFINITY`    |

***

## 9. 本章小结

本章用一个"LED 闪 + 按键删任务"的小工程，把 FreeRTOS 多任务最核心的几件事讲清楚了：

```text
任务创建  xTaskCreatePinnedToCore()
   ↓
任务调度  同优先级时间片轮转 / 高优先级抢占
   ↓
任务让出  vTaskDelay() → Blocked
   ↓
IDLE 喂狗  让出 CPU 才能避免看门狗复位
   ↓
任务删除  vTaskDelete() + 句柄置 NULL
```

记住三条主线：

1. **创建任务** 时要给够栈、想清楚优先级和核心绑定；
2. **任务循环** 里必须有让出 CPU 的动作（`vTaskDelay` 或阻塞 API），否则会饿死低优先级任务并触发看门狗；
3. **停顿时间** 的单位是 tick，和 `configTICK_RATE_HZ` 直接相关，工程上推荐 `pdMS_TO_TICKS()`。

最后，FreeRTOS 用一套自定义类型（`BaseType_t`、`TickType_t`、`TaskHandle_t` 等）屏蔽了不同 MCU 的原生类型差异，这是它能跑在几十种芯片上的根本原因。在自己写代码时沿用这套类型，也是提升可移植性的好习惯。

---

## Tips

### 本章最易忽视的细节

1. **ESP32-S3 默认 `configTICK_RATE_HZ = 100`，不是 1000**：很多从 STM32 转过来的开发者习惯用 1000Hz，以为 `vTaskDelay(100)` = 100ms。在 ESP-IDF 上 `vTaskDelay(100)` = 1000ms。如果你想用 1000Hz，需要在 `menuconfig` 中改 `CONFIG_FREERTOS_HZ`，但这会增加 tick 中断频率，间接增加功耗。
2. **`app_main()` 返回后 main 任务被删除，这是推荐做法**：不要在 `app_main()` 里写 `while(1) { vTaskDelay(500); }` 来保持任务——这纯粹浪费一个任务栈。如果确实需要一个"主循环"，应该创建一个专门的用户任务。本章在 `app_main()` 中写 `while(1)` 只是为了演示最简单的结构。
3. **栈大小 1024 字 = 4096 字节，对简单任务来说绰绰有余**：但如果任务里有 `printf`、局部大数组、递归调用，栈就可能不够用。ESP-IDF 默认开启了栈溢出检测（`CONFIG_FREERTOS_CHECK_STACK_OVERFLOW`），栈溢出时会触发 panic。
4. **时间片轮转不是"公平平分 CPU 时间"**：FreeRTOS 的时间片轮转在每个 tick 边界切换，但谁先运行是由 Ready 队列的顺序决定的。如果你在 `app_main()` 中先创建 `LED_task`，它就在队列前面，调度器会先执行它。

### 踩坑记录

- **栈以"字"为单位，不是"字节"**：这是本章最经典的坑。`xTaskCreatePinnedToCore(..., 256, ...)` → 实际栈大小是 `256 * 4 = 1024` 字节。如果你以为 `256` 是字节、实际给了 `1024`，那没问题（多给了）；但如果你真需要 256 字节的栈却写了 `256`，实际是 1024 字节，浪费了内存。反过来，如果你需要 4KB 栈却写了 `1024`（以为 1024 字节），那栈严重不足，会触发栈溢出。
- **删完任务后继续用句柄**：`vTaskDelete(LED_task_handle)` 之后句柄指向的内存已经被释放，任何对该句柄的操作（`vTaskSuspend`、`vTaskResume`）都会导致崩溃。本章的 `LED_task_handle = NULL` 就是为了防止这种情况。
- **在 ISR 里调用 `vTaskDelay`**：`vTaskDelay()` 不能在中断服务程序中调用，因为 ISR 没有任务上下文。ISR 中需要使用 `vTaskDelayUntil()` 或其他的 ISR 安全 API。

### `vTaskDelay` 为什么同时完成"让出 CPU / 喂狗 / 延时"三件事

第 6 节已经分别解释了这三件事。这里把它们串起来，从一个统一的视角理解：

`vTaskDelay(100)` 的核心操作是**把当前任务从 Ready 队列移到 Blocked 队列，并设置一个 100 tick 后的唤醒定时器**。这个操作本身就引发了三个连锁反应：

```text
vTaskDelay(100)
   │
   ├── 1. 当前任务进入 Blocked → 调度器选下一个 Ready 任务
   │       这就是"让出 CPU"
   │
   ├── 2. 如果其他任务都在 Blocked → 调度器选 IDLE 任务
   │        IDLE 任务运行 → 喂看门狗
   │       这就是"喂狗"
   │
   └── 3. 100 tick 后，tick 中断唤醒当前任务 →回到 Ready
          这就是"延时"
```

这三个效果其实是一个操作在不同视角下的投影：
- 从**调度器视角**看，是"让出 CPU"；
- 从**系统健康视角**看，是"喂狗"（因为 IDLE 得以运行）；
- 从**任务自身视角**看，是"延时 1 秒后恢复"。

理解这一点后，你会发现所有阻塞 API（`vTaskDelay`、`xQueueReceive`、`ulTaskNotifyTake` 等）都同时具备这三个效果。这就是为什么 FreeRTOS 中"阻塞 = 让出 CPU"是铁律——阻塞是调度器切换任务的唯一触发机制（除了 tick 中断抢断）。

### tick 和 Hz 的深入理解

第 6.3 节已经解释过 `1000ms / configTICK_RATE_HZ = 1 tick` 的基本关系。下面展开更多实际影响：

**tick 频率的 trade-off：**

| 场景 | 低 tick（如 50Hz） | 高 tick（如 1000Hz） |
|------|-------------------|---------------------|
| tick 中断频率 | 低 → 功耗低 | 高 → 功耗高（每 1ms 一次中断） |
| 延时精度 | `vTaskDelay(1)` = 20ms 起跳 | `vTaskDelay(1)` = 1ms 起跳 |
| 调度延迟 | 最坏 20ms 才能切换任务 | 最坏 1ms 就能切换 |
| 时间片粒度 | 粗 | 细 |

**一个常见的困惑：`vTaskDelay(1)` 到底延时多久？**

在 100Hz 下，`vTaskDelay(1)` 理论上延时 10ms，但实际精度受 tick 中断的位置影响。如果刚好在 tick 中断前一刻调用 `vTaskDelay(1)`，实际延时可能是 10ms + 一个 tick 的偏移量（接近 20ms）。这就是为什么 FreeRTOS 推荐 `vTaskDelayUntil()` 用于精确周期性任务——它可以根据上一次唤醒的时间来计算下一次延时，避免累积误差。

**ESP-IDF 默认 100Hz 的原因：**
- 10ms 的调度粒度对大多数物联网应用够用；
- 每秒 100 次 tick 中断对功耗友好；
- 和 FreeRTOS 在其他平台上的默认值保持一致。

如果你需要高精度定时（如控制步进电机），建议用硬件定时器（ESP32-S3 有 4 个通用定时器），而不是依赖 FreeRTOS 的 tick 中断。

### FreeRTOS 数据类型对照表的意义

第 7 节已经列出了对照表。这里补充两点深度思考：

**1. 为什么 TickType_t 在 16 位 MCU 上是 16 位？够用吗？**

在 100Hz 下，16 位无符号整数的最大值是 65535 tick = 655.35 秒 ≈ 11 分钟。也就是说，在 16 位平台上，`vTaskDelay(70000)` 会溢出回绕。FreeRTOS 内部已经处理了这个问题（用取模运算），但用户如果直接比较两个 tick 值就需要注意溢出。
在 32 位平台上（如 ESP32-S3），`TickType_t` 是 `uint32_t`，最大值约 49.7 天。也就是说，系统连续运行 49.7 天后 tick 计数器会回绕。如果你的应用需要连续运行超过这个时间，建议使用 64 位时间戳（如 `esp_timer_get_time()`）。

**2. 为什么 BaseType_t 不是 `int32_t`？**

`BaseType_t` 被设计为"该架构最高效的整数类型"。在 32 位 MCU 上它是 `int32_t`，在 64 位 PC 上它是 `int64_t`，在 16 位 MCU 上它是 `int16_t`。这和 C 语言的 `int` 语义一致，但 FreeRTOS 不信任所有编译器都遵循这个约定（有些嵌入式编译器把 `int` 固定为 16 位），所以自己 typedef 了一层。
在实际开发中，你应该：
- 用 `BaseType_t` 接 FreeRTOS API 的返回值；
- 用 `TickType_t` 存延时值和 tick 计数；
- 用 `UBaseType_t` 存任务优先级和队列长度；
- 用标准 C 的 `uint32_t` 做你自己的数据运算（明确需要 32 位的地方）。

这样既保持了可移植性，又保持了代码意图的清晰性。

### 与后续章节的知识衔接

- **任务挂起与恢复**（第 2 章）：本章用 `vTaskDelete()` 删任務，下一章用 `vTaskSuspend()` / `vTaskResume()` 实现可逆的暂停。理解本章的 Blocked 态后，Suspended 态的语义就很好懂了——Blocked 是"等一等"，Suspended 是"被人关掉了"。
- **队列与信号量**：`xQueueReceive()` 和 `vTaskDelay()` 一样，都会让任务进入 Blocked 态。区别在于唤醒条件：`vTaskDelay` 是"时间到了"，`xQueueReceive` 是"队列有数据了"。
- **软件定时器**：FreeRTOS 的软件定时器回调函数运行在 Timer Task 中，它的实现依赖于 tick 中断。理解本章的 tick 机制后，你就能理解为什么软件定时器的精度受限于 tick 频率。
- **中断管理**：本章的 `vTaskDelay` 不能在 ISR 中使用。后续章节会介绍 ISR 安全的 `xTaskResumeFromISR`、`xQueueSendFromISR` 等 API，它们都带有 `FromISR` 后缀。

### 设计决策思考：为什么 ESP-IDF 用双核但本章只用单核

本章把两个任务都钉在 core 0。这是故意的教学简化。ESP32-S3 的两个核心（core 0 和 core 1）都受 FreeRTOS 调度器管理，理论上你可以把 `LED_task` 放 core 0、`key_task` 放 core 1，实现真正的并行执行。

但在教学阶段，单核绑定有三个好处：
1. **行为可预测**：两个任务在同一个核上，时序关系是确定的（时间片轮转）。跨核时，两个任务可能同时运行，调试时序问题会变得复杂。
2. **避免缓存一致性问题**：跨核共享全局变量（如 `LED_task_handle`）需要考虑内存屏障和数据竞争。本章用 `LED_task_handle = NULL` 这种简单赋值在单核上没问题，在多核上需要 `volatile` 或原子操作保护。
3. **留一个核给系统任务**：ESP-IDF 的 Wi-Fi、lwIP 等系统任务默认运行在 core 0 或 core 1（取决于配置）。本章留出 core 1 给这些任务，避免你的用户任务与系统任务抢 CPU。

正式项目中，推荐用 `tskNO_AFFINITY` 让调度器自动分配核心，或者显式把计算密集型任务放在与 Wi-Fi 不同的核心上。
