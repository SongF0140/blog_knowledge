---
title: "Qt 学习知识地图"
published: 2026-08-17
description: "Qt 学习项目（qt01–qt22，缺 qt18）的全局导航：21 篇带 mermaid 图的学习笔记的学习顺序、依赖关系与跳转索引。"
tags: ["Qt","C++","GUI","知识地图","导航"]
category: Qt学习
draft: false
---

# Qt 学习知识地图

> 本目录由 `D:\QT_place\QT_study_pjt` 中各 Qt 示例工程（qt01→qt22，缺 qt18）逐篇提炼而来，每篇均为详细 mermaid 笔记，含「每章 Tips」与「Q&A」。本页是总入口。

## 学习路线总览

```mermaid
flowchart LR
    subgraph 基础与环境["一、基础与环境（qt01–qt07）"]
        A1["01 第一个程序"] --> A2["02 QWidget 基础"]
        A2 --> A3["03 CMake 构建"]
        A3 --> A4["04 CMake+Qt6"]
        A4 --> A5["05 VSCode 模板"]
        A5 --> A6["06 CLion"]
        A6 --> A7["07 RCC 资源"]
    end
    subgraph 核心机制["二、核心机制（qt08–qt17）"]
        A7 --> B1["08 对象树"]
        B1 --> B2["09 QString 编码"]
        B2 --> B3["10 char/字符串"]
        B3 --> B4["11 布局"]
        B4 --> B5["12 随机/定时器"]
        B5 --> B6["13 信号槽"]
        B6 --> B7["14 键盘面板"]
        B7 --> B8["15 事件系统"]
        B8 --> B9["16 鼠标事件"]
        B9 --> B10["17 键盘事件"]
    end
    subgraph 进阶应用["三、进阶应用（qt19–qt22）"]
        B10 --> C1["19 线程 moveToThread"]
        C1 --> C2["20 绘图基础"]
        C2 --> C3["21 速度表"]
        C3 --> C4["22 SQLite 数据库"]
    end
```

> 💡 Tips：路线不是强制顺序，但**强烈建议先走完"基础与环境"再进"核心机制"**——对象树/信号槽/事件都依赖你已经理解 `QApplication` + `exec()` 的事件循环（01）。"进阶应用"三篇（线程/绘图/SQLite）是核心机制的落地组合技，建议放在最后。

## 各篇导航表

| 序号 | 文件 | 主题 | 一句话要点 | 关键 mermaid |
| ---- | ---- | ---- | --------- | ------------ |
| 01 | [01-qt-first-program.md](01-qt-first-program.md) | 第一个 Qt 程序 | `QApplication` + `exec()` 事件循环 | 程序启动/事件循环 flowchart |
| 02 | [02-qt-widget-basic.md](02-qt-widget-basic.md) | QWidget 基础 | 控件显示、父子关系、`w.show()` | 控件层级 flowchart |
| 03 | [03-cmake-build.md](03-cmake-build.md) | CMake 构建 | `find_package(Qt5)` + `target_link_libraries` | 构建流程 flowchart |
| 04 | [04-cmake-qt6.md](04-cmake-qt6.md) | CMake + Qt6 | Qt5→Qt6 的 CMake 差异 | 差异对比 flowchart |
| 05 | [05-vscode-cmake-template.md](05-vscode-cmake-template.md) | VSCode + CMake 模板 | VSCode 开发 Qt 的配置模板 | 工具链 flowchart |
| 06 | [06-clion-project.md](06-clion-project.md) | CLion 工程 | CLion 配置 Qt 工具链 | 工程结构 flowchart |
| 07 | [07-rcc-resource.md](07-rcc-resource.md) | RCC 资源系统 | `.qrc` + `:/` 前缀访问资源 | 资源编译 flowchart |
| 08 | [08-object-tree.md](08-object-tree.md) | 对象树 | 父对象析构自动析构子对象 | 父子归属/析构 classDiagram |
| 09 | [09-qstring-char.md](09-qstring-char.md) | QString 与编码 | `toUtf8`/`fromUtf8` 解决中文乱码 | 编码转换 flowchart |
| 10 | [10-char-string.md](10-char-string.md) | char / 字符串 | C++ 字符编码基础 | 层次 classDiagram |
| 11 | [11-layout.md](11-layout.md) | 布局管理器 | `H/V/Grid/Form` 布局替代绝对坐标 | 嵌套层级 classDiagram |
| 12 | [12-random-timer.md](12-random-timer.md) | 随机数 / 定时器 | `QRandomGenerator` + `QTimer` 动画 | 随机/属性协作 flowchart |
| 13 | [13-signal-slot.md](13-signal-slot.md) | 信号与槽 | `connect`/`emit`/`Q_OBJECT` 宏 | emit→槽 sequenceDiagram |
| 14 | [14-keypad.md](14-keypad.md) | 键盘面板 | 用信号槽把按键接到逻辑 | 信号流向 flowchart |
| 15 | [15-event-system.md](15-event-system.md) | 事件系统 | `event()`/事件过滤器分发链 | 事件分发 flowchart |
| 16 | [16-mouse-event.md](16-mouse-event.md) | 鼠标事件 | `mousePress/Move/ReleaseEvent` | 事件处理 flowchart |
| 17 | [17-keyboard-event.md](17-keyboard-event.md) | 键盘事件 | `keyPressEvent`/`QKeyEvent` | 事件处理 flowchart |
| 19 | [19-thread-movetothread.md](19-thread-movetothread.md) | 线程 moveToThread | Worker 移到子线程、信号跨线程 | 对象归属线程迁移 flowchart |
| 20 | [20-painter-base.md](20-painter-base.md) | 绘图基础 | `paintEvent` + `QPainter` | 重绘流程 flowchart |
| 21 | [21-speed-meter.md](21-speed-meter.md) | 速度表 | 自定义控件绘制 | 绘制流程 flowchart |
| 22 | [22-sqlite.md](22-sqlite.md) | SQLite 数据库 | Qt SQL 模块增删改查 | 建库/CRUD flowchart |

> 💡 Tips：本表"关键 mermaid"一列可直接定位每篇最值得看的图。如果某篇概念记不清，先回来看对应图，再点文件名进正文。

## 知识依赖速查

- **想懂"按钮点了为什么有反应"** → 先 01（事件循环）→ 13（信号槽）→ 14（面板示例）
- **想懂"窗口关掉为什么内存不漏"** → 08（对象树）
- **想懂"界面卡死"** → 19（线程，把耗时逻辑移出 GUI 线程）
- **想懂"自定义好看控件"** → 20（绘图）→ 21（速度表实战）
- **想懂"本地存数据"** → 22（SQLite）
- **想懂"中文乱码"** → 09（QString 编码）→ 10（字符基础）

## Q&A（常见问题）

**Q1：qt18 去哪了？**
A1：源工程 `D:\QT_place\QT_study_pjt` 里本来就没有 `qt18_*` 文件夹（编号从 qt17 直接跳到 qt19），因此本知识地图只有 21 篇，编号中间缺 18 是原始工程使然，并非遗漏。

**Q2：这些笔记和 `嵌入式笔记` 是什么关系？**
A2：二者并列放在 `gitcode_knowledge` 下，模板风格一致（YAML 头/目录/API 表/mermaid/踩坑/Q&A/Tips），但内容无关——`嵌入式笔记` 是 ESP32 嵌入式方向，`QT学习笔记` 是 Qt GUI 方向。

**Q3：为什么有的文件夹（如 qt22）笔记是"标准示例"而非工程源码解析？**
A3：qt22 文件夹实际只含 `.idea` IDE 配置、没有可参考源码，因此该篇采用 Qt SQLite 标准示例写法并已在"核心代码解析"中注明；其余各篇均引用了对应工程真实源码。

**Q4：想让这些笔记也能被"笔记搜索"检索怎么办？**
A4：可用 qmd 建立集合：`qmd collection add D:/GITHUB_PLACE/gitcode_knowledge/QT学习笔记 --name qt-notes` 再 `qmd embed`（需先安装 bun）。这是可选增强，不影响笔记本身使用。

## Tips（全局速记）

### 最容易忽视的细节
- 每篇都带「每章 Tips」与「Q&A」，遇到概念混淆先翻这两处。
- 编号缺 18 是源工程事实，不是笔记缺失。

### 踩坑记录
- 构建类（03–07）最容易卡在工具链/CMake 配置，问题多在环境而非 Qt API。

### 与后续章节的衔接
- Qt 笔记与 `嵌入式笔记` 可互为"GUI 端 / 设备端"参考：例如 SQLite 笔记的数据，可来自嵌入式端采集。

### 设计决策思考
- 学习顺序按"基础→核心→进阶"三阶段推进，比按文件夹字母序更顺；本地图即按此组织。
