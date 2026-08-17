---
title: "Qt 11 布局管理器（QHBoxLayout / QVBoxLayout / QGridLayout / QFormLayout / QSplitter）学习笔记"
published: 2026-08-17
description: "掌握 Qt 四大布局管理器与 QSplitter 的用法：用布局而非绝对坐标自适应排列控件，理解嵌套布局的层级关系、伸缩因子 stretch 与窗体伙伴 buddy。"
tags: ["Qt","C++","GUI","布局","QHBoxLayout","QVBoxLayout","QGridLayout","QFormLayout","QSplitter"]
category: Qt学习
draft: false
---

# Qt 11 布局管理器（布局而非绝对坐标）学习笔记

## 本章重要 API 函数

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `QHBoxLayout(QWidget *parent)` | 水平布局 | 控件从左到右排列；`parent` 通常是承载它的窗口/容器 | `<QHBoxLayout>` |
| `QVBoxLayout(QWidget *parent)` | 垂直布局 | 控件从上到下排列 | `<QVBoxLayout>` |
| `QLayout::addWidget(w, stretch, alignment)` | 加入控件 | `w`：控件；`stretch`：伸缩因子（占比）；`alignment`：对齐 | `<QLayout>` |
| `QBoxLayout::setStretch(index, stretch)` | 设置某项伸缩比 | `index`：子项序号；`stretch`：权重。源码 `buttonLayout->setStretch(0,1)` | `<QBoxLayout>` |
| `QGridLayout::addWidget(w, row, col, rowSpan, colSpan)` | 网格放置 | 行、列、跨行数、跨列数。源码 `grp_layout->addWidget(btn1,0,0,1,1)` | `<QGridLayout>` |
| `QFormLayout::setWidget(row, role, widget)` | 表单行 | `role`：`LabelRole`/`FieldRole`；源码用它放标签与输入框 | `<QFormLayout>` |
| `QLabel::setBuddy(QWidget*)` | 设置伙伴控件 | 标签里 `&n` 快捷键聚焦到 buddy（如输入框） | `<QLabel>` |
| `QSplitter(Qt::Orientation, parent)` | 可拖动分隔条 | 用户可拖拽改变子区域大小，比固定布局更灵活 | `<QSplitter>` |

> 💡 Tips：布局的 `stretch` 是**比例**不是像素。例如 `setStretch(0,1); setStretch(1,2); setStretch(2,1)` 表示三项按 1:2:1 瓜分剩余空间，窗口缩放时等比例变化。还有：把布局设给 `this`（窗口）时，布局成为窗口的顶层布局，自动管理所有子控件几何尺寸——你**不应再手动 `setGeometry` 定位子控件**。

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数](#本章重要-api-函数)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. 核心概念 / 原理](#2-核心概念--原理)**
- **[3. 布局层级与嵌套结构](#3-布局层级与嵌套结构)**
- **[4. 核心代码解析](#4-核心代码解析)**
- **[5. 运行效果 / 验证](#5-运行效果--验证)**
- **[容易踩坑（注意点）](#容易踩坑注意点)**
- **[学习要点总结](#学习要点总结)**
- **[Q&A（常见问题）](#qa常见问题)**
- **[Tips（全局速记）](#tips全局速记)**

---

## 学习前后依赖

| 方向 | 内容 |
| ---- | ---- |
| 前置基础 | **qt08 对象树**：所有 `new QPushButton(name, panel)` 的父参数决定控件归属与生命周期；**qt09/10 字符串**：按钮中文文案。 |
| 本章核心 | 四种布局管理器的用法、嵌套布局、伸缩因子、QSplitter、窗体伙伴。 |
| 后续承接 | 所有后续 GUI 章节（qt13 信号槽界面、qt14 键盘、qt20 绘图等）的界面都靠布局搭建；复杂界面 = 布局的层层嵌套。 |

> 💡 Tips：布局管理器**强依赖对象树（qt08）**——布局本身也是 `QObject`，控件作为子对象交给父窗口。如果你没理解"父参数 = 内存归属"，就会困惑为什么布局里的控件不用手动 delete。先 qt08，再 qt11，顺序固定。

## 1. 概述

早期 GUI 程序常用"绝对坐标"摆放控件：用 `setGeometry(x, y, w, h)` 写死每个按钮的位置和大小。这有两个致命问题：① 窗口缩放时控件不会跟着变，留白或溢出难看；② 多语言/高分屏下文字长度变化，布局直接错位。Qt 给出的解法是**布局管理器（Layout）**：你只声明"控件之间的相对关系"（水平排、垂直排、网格排、表单排），Qt 在窗口大小变化、字体变化时**自动计算并刷新每个控件的位置与尺寸**。

`qt11` 用 `Widget::show1()~show4()` 四个函数演示了全部主流布局：水平/垂直嵌套（`show1`）、网格 `QGridLayout`（`show2`）、表单 `QFormLayout` 配 `buddy` 快捷键（`show3`）、以及可拖拽的 `QSplitter` 嵌套（`show4`）。构造函数里调用 `show4()`，运行即见一个可拖动三区划分的窗口。此外 `widget.ui` 还用 Designer 拖出了一个 `QFormLayout` 内嵌 `QVBoxLayout` 的示例，展示"可视化布局"与"代码布局"两种等价路径。

> 💡 Tips：概述里最该记住的一句话——**"布局管的是'关系'不是'坐标'；把控件交给布局后，就别再手动 setGeometry 它们了。"** 布局和绝对坐标二选一，混用会导致布局被覆盖或控件重叠。

## 2. 核心概念 / 原理

### 2.1 布局如何工作

布局本质是一个**几何求解器**。当窗口 `show()` 或尺寸变化，Qt 触发"布局重算"：布局询问每个子项的 `sizeHint()`（理想尺寸）与 `sizePolicy()`（拉伸意愿），结合自身的 `stretch` 权重，把可用空间分配给各子项，最后调用子项的 `setGeometry` 落实像素位置。这一切由事件循环自动驱动，你无需手写重绘。

### 2.2 父子与归属

- 布局对象本身用 `new QHBoxLayout(this)` 挂到窗口（或容器控件）下，成为其顶层布局。
- 被 `addWidget` 的控件，其父对象应是**承载布局的容器**（如 `new QPushButton(name, buttonPanel)` 父是 `buttonPanel`），这样控件随容器销毁而销毁（qt08）。
- 嵌套布局：一个容器控件（如 `QWidget`、`QGroupBox`）拥有自己的子布局，再被父布局 `addWidget`——形成布局树。

### 2.3 伸缩因子 stretch

`addWidget(w, stretch)` 与 `setStretch(i, stretch)` 控制"剩余空间"的分配比例。 stretch=0 表示尽量按 `sizeHint` 不抢空间；stretch>0 表示按比例瓜分多余空间。多个 stretch 之比即最终宽度之比。

> 💡 Tips：原理层最易误解的是"布局会自动让控件等宽/等高"——**不会**，除非你设了 stretch 且 `sizePolicy` 允许扩展。`QPushButton` 默认 `sizePolicy` 是 `Preferred`，水平可扩展、垂直固定；源码 `show1` 里 `button->setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding)` 就是显式告诉布局"我要占满可用空间"。

## 3. 布局层级与嵌套结构

下面用 classDiagram 展示 `show1` 的嵌套层级（窗口 → 水平布局 → 按钮面板(垂直布局) + 文本编辑），以及 `show4` 的 `QSplitter` 嵌套：

```mermaid
classDiagram
    class Widget {
        QWidget (this)
    }
    class MainHLayout {
        QHBoxLayout
    }
    class ButtonPanel {
        QWidget
        含 QVBoxLayout
    }
    class VLayout {
        QVBoxLayout
        三项 stretch 1:2:1
    }
    class Edit {
        QTextEdit
        stretch 2
    }
    class SplitterH {
        QSplitter 水平
        左1 : 右2
    }
    class SplitterV {
        QSplitter 垂直
        上1 : 下2
    }
    Widget *-- MainHLayout : 顶层布局
    MainHLayout *-- ButtonPanel : addWidget stretch1
    MainHLayout *-- Edit : addWidget stretch2
    ButtonPanel *-- VLayout
    VLayout *-- "3x QPushButton"
    Widget *-- SplitterH : show4 顶层
    SplitterH *-- "功能区 GroupBox"
    SplitterH *-- SplitterV
    SplitterV *-- "输入区 GroupBox"
    SplitterV *-- "输出区 GroupBox"
```

图中 `*`--` 表示"拥有/包含"的布局归属关系。`show1` 是"水平布局里嵌套一个带垂直布局的面板 + 一个编辑框"；`show4` 是"水平 `QSplitter` 里再嵌套一个垂直 `QSplitter`"，形成可拖拽的田字/三区结构。

> 💡 Tips：图里最关键的是 `Widget *-- MainHLayout`（窗口拥有顶层布局）与 `ButtonPanel *-- VLayout`（容器控件拥有自己的子布局）——**布局必须挂在一个具体控件上才有意义**。常见错误是把布局 `new` 出来却没设给任何 `this`/容器，结果控件全堆在 (0,0)。另外 `QSplitter` 既是布局容器也是可见分隔条，与纯 `QLayout` 不同，它自身是一个 `QWidget`。

## 4. 核心代码解析

`qt11_tst_LayOut/widget.cpp` 的 `show1()`（构造函数实际调用 `show4()`，但 `show1~3` 是同等重要的教学代码）：

```cpp
void Widget::show1() {
    resize(360, 480);
    auto *mainLayout = new QHBoxLayout(this);          // 顶层水平布局挂在窗口
    auto *buttonPanel = new QWidget(this);             // 容器控件（父=窗口）
    auto *buttonLayout = new QVBoxLayout(buttonPanel); // 面板内垂直布局
    auto *edit = new QTextEdit(this);
    QSizePolicy sizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
    const QString buttonNames[] = {"按钮1","按钮2","按钮3"};
    for (const auto &name : buttonNames) {
        auto *button = new QPushButton(name, buttonPanel); // 父=面板！
        button->setSizePolicy(sizePolicy);
        buttonLayout->addWidget(button);
    }
    buttonLayout->setStretch(0, 1);
    buttonLayout->setStretch(1, 2);
    buttonLayout->setStretch(2, 1);   // 三按钮高度比 1:2:1
    mainLayout->addWidget(buttonPanel, 1);
    mainLayout->addWidget(edit, 2);   // 面板:编辑框 宽度比 1:2
}
```

- `new QPushButton(name, buttonPanel)`：**父对象是 buttonPanel 而非窗口**，所以按钮随面板销毁，且被 `buttonLayout` 管理几何。删掉这个父参数，按钮虽被布局 add 但仍然要自己管内存——典型漏写。
- `setStretch` 在 `buttonLayout` 上设 1:2:1，在 `mainLayout` 上 `addWidget(panel,1)`、`addWidget(edit,2)` 设 1:2，形成双层比例。

```cpp
void Widget::show2() {
    auto *main_layout = new QHBoxLayout(this);
    auto *grp = new QGroupBox(this);
    auto *grp_layout = new QGridLayout(grp);
    auto *btn1 = new QPushButton("按钮1", grp);
    auto *btn2 = new QPushButton("按钮2", grp);
    auto *cmbbox = new QComboBox(grp);
    auto *edit = new QTextEdit(grp);
    grp_layout->addWidget(btn1, 0, 0, 1, 1);
    grp_layout->addWidget(btn2, 0, 1, 1, 1);
    grp_layout->addWidget(cmbbox, 1, 0, 1, 2);   // 第1行跨2列
    grp_layout->addWidget(edit, 2, 0, 1, 2);     // 第2行跨2列
    main_layout->addWidget(grp);
}
```

- `QGridLayout::addWidget(w, row, col, rowSpan, colSpan)`：后两参是**跨行/跨列数**。`cmbbox` 在 `row=1,col=0` 跨 2 列，占满整行。`QGroupBox` 是带标题边框的容器，常作网格布局的外框。

```cpp
void Widget::show3() {
    auto *grp_layout = new QFormLayout(grp);
    auto *userlabel = new QLabel("用户名(&n):", grp);  // &n 定义快捷键 Alt+N
    auto *useredit = new QLineEdit(grp);
    auto *typelabel = new QLabel("密码(&p):", grp);
    auto *typeedit = new QLineEdit(grp);
    grp_layout->setWidget(0, QFormLayout::LabelRole, userlabel);
    grp_layout->setWidget(0, QFormLayout::FieldRole, useredit);
    grp_layout->setWidget(1, QFormLayout::LabelRole, typelabel);
    grp_layout->setWidget(1, QFormLayout::FieldRole, typeedit);
    userlabel->setBuddy(useredit);   // Alt+N 聚焦用户名框
    typelabel->setBuddy(typeedit);   // Alt+P 聚焦密码框
}
```

- `QFormLayout` 专为"标签—输入"两列表单设计；`LabelRole`/`FieldRole` 明确每行的标签与字段。`setBuddy` 让标签的 `&n`/`&p` 快捷键把焦点送到对应输入框——提升可访问性。

```cpp
void Widget::show4() {                 // 构造函数实际调用
    auto *main_layout = new QHBoxLayout(this);
    auto *main_spliter = new QSplitter(Qt::Horizontal, this);
    main_spliter->setChildrenCollapsible(false);
    auto lefy_grp = new QGroupBox("功能区", main_spliter);
    auto right_spliter = new QSplitter(Qt::Vertical, main_spliter);
    auto edit = new QGroupBox("输入区", right_spliter);
    auto show_grp = new QGroupBox("输出区", right_spliter);
    main_spliter->setStretchFactor(0, 1);
    main_spliter->setStretchFactor(1, 2);
    right_spliter->setStretchFactor(0, 1);
    right_spliter->setStretchFactor(1, 2);
    main_layout->addWidget(main_spliter);
}
```

- `QSplitter` 与布局不同：它是**可见的、可拖拽**的分隔控件。`setStretchFactor` 设定初始比例（`QSplitter` 用 `setStretchFactor` 而非布局的 `setStretch`）。`setChildrenCollapsible(false)` 禁止把子区域拖到完全消失。`widget.h` 里 `#include <QSplitter>` 不可少，否则编译不过。

> 💡 Tips：这段代码藏着三个细节——① `addWidget` 的 `stretch` 参数（`mainLayout->addWidget(panel,1)`）与 `setStretch(index,stretch)` **作用相同但调用点不同**：前者在加入时指定，后者事后按序号改；② `QSplitter` 不是 `QLayout` 子类，它是 `QWidget`，所以既能当容器也能被 `addWidget` 进布局；③ `widget.ui` 里 Designer 生成的 `QFormLayout` 内嵌 `QVBoxLayout`（stretch="0,1,2"），与 `show3` 代码布局是**同一概念的两种实现方式**——可视化 vs 手写。

## 5. 运行效果 / 验证

编译运行 `qt11_tst_LayOut`，构造函数调用 `show4()`，应看到一个窗口：

- 左侧"功能区" `QGroupBox`，右侧被垂直 `QSplitter` 分成上下"输入区""输出区"，中间有可拖拽的分隔条；左右比例初始约 1:2，上下约 1:2。
- 拖动分隔条，区域大小实时变化；缩放窗口，比例保持。
- 若把 `show4()` 换成 `show1()`：窗口左边三按钮（高度 1:2:1）、右边文本框（宽占 2 份），缩放自适应。
- `show2`：网格内按钮、跨列下拉框与文本框；`show3`：表单两行标签+输入框，按 `Alt+N`/`Alt+P` 焦点跳转。

> 💡 Tips：验证时最容易误判"成功"的假象——窗口能显示就以为布局对了。应**拖动分隔条/缩放窗口**确认控件真的在重排；若控件大小纹丝不动、重叠或挤在左上角，说明布局没真正设给 `this`（可能忘了 `new QHBoxLayout(this)` 的 `this`，或控件父对象与布局宿主不一致）。

## 容易踩坑（注意点）

- 坑 1：**布局没设给 `this`/容器**。现象：控件全堆在窗口左上角、不自适应。原因：`new QHBoxLayout()` 漏传父对象，布局成了孤儿。规避：始终 `new QHBoxLayout(this)` 或 `widget->setLayout(layout)`。
- 坑 2：**控件父对象与布局宿主不一致**。现象：内存不释放或几何错乱。原因：控件 `new QPushButton(name)` 没给父，却 `addWidget` 进某布局。规避：控件父对象应是承载该布局的容器（如 `buttonPanel`）。
- 坑 3：**布局与 `setGeometry` 混用**。现象：手动设的几何被布局覆盖或控件重叠。原因：交给布局后几何由布局算。规避：布局管理的控件不要再 `setGeometry`；要固定大小用 `setFixedSize`/`sizePolicy`。
- 坑 4：**`stretch` 设了但 `sizePolicy` 不允许扩展**。现象：比例无效、控件不撑开。原因：`sizePolicy` 默认 `Preferred`/`Minimum` 可能拒绝扩展。规避：需要撑满设 `QSizePolicy::Expanding`。
- 坑 5：**`QSplitter` 当布局用 `setStretch`**。现象：比例无效。原因：`QSplitter` 用 `setStretchFactor`，不是布局的 `setStretch`。规避：记清 API 差异。
- 坑 6：**`QGridLayout` 跨行列参数理解错**。现象：控件错位/覆盖。原因：后两参是 `rowSpan/colSpan`（跨度）不是结束行列。规避：`addWidget(w, r, c, 1, 2)` 表示占 1 行 2 列。

## 学习要点总结

### 核心知识点
1. 布局管理"相对关系"，窗口缩放/字体变化自动重排，优于绝对坐标。
2. 四大布局：`QHBoxLayout`(水平)、`QVBoxLayout`(垂直)、`QGridLayout`(网格)、`QFormLayout`(表单)。
3. 布局可嵌套（容器控件持有子布局）；`stretch` 控制比例；`sizePolicy` 决定扩展意愿。
4. `QSplitter` 是可拖拽分隔的 `QWidget`，用 `setStretchFactor` 设比例。

### 关键 API
- `QHBoxLayout/QVBoxLayout::addWidget(w, stretch)`
- `QBoxLayout::setStretch(index, stretch)`
- `QGridLayout::addWidget(w, row, col, rSpan, cSpan)`
- `QFormLayout::setWidget(row, role, w)` + `QLabel::setBuddy`
- `QSplitter::setStretchFactor` / `setChildrenCollapsible`

### 下一步学习
- **qt13 信号槽**：在布局好的按钮上连接点击事件。
- **qt14 键盘 / qt20 绘图**：自定义控件放进布局。
- **qt22 SQLite**：用 `QFormLayout` 做数据录入表单。

> 💡 Tips：本篇直接挂钩 **qt08（对象树，布局依赖父子归属）** 与 **qt13（在布局控件上挂信号槽）**。口诀："布局设给 this，控件父给容器，比例用 stretch，扩展看 sizePolicy。"

## Q&A（常见问题）

**Q1：布局和 `QSplitter` 有什么区别？该用哪个？**  A1：布局（`QLayout`）是不可见的几何求解器，自动排列但不能由用户拖拽调整；`QSplitter` 是可见的、可拖拽的分隔控件，用户能实时改变区域大小。静态自适应用布局，需要用户手动调比例用 `QSplitter`（它本身也能被放进布局里嵌套）。

**Q2：`stretch` 到底怎么算？设了为什么没效果？**  A2：`stretch` 是剩余空间的分配权重，比值决定最终占比（如 1:2:1）。若没效果，多半是子控件的 `sizePolicy` 不允许扩展（如 `Fixed`/`Minimum` 水平方向拒绝拉伸），或你用了 `setFixedSize` 锁死。把对应方向 `sizePolicy` 设为 `Expanding` 再试。

**Q3：Qt5 和 Qt6 在布局上有差异吗？**  A3：布局 API 几乎完全兼容。Qt6 主要变化：① `QWidget::setLayout` 等行为一致；② High-DPI 支持更强，布局 `sizeHint` 按设备像素比更准；③ `QFormLayout` 在 Qt6 对 `RowWrapPolicy` 等细节有微调。代码层面 `show1~4` 在 Qt5/Qt6 都能编译运行。

## Tips（全局速记）

### 最容易忽视的细节
1. **布局对象本身也要有父（`this`）**，否则不生效——这是新手第一坑。
2. **控件父对象应是布局宿主容器**，而非永远传窗口；嵌套布局时尤要注意。
3. **`QSplitter` 是 QWidget 不是 QLayout**，嵌套进布局用 `addWidget`，比例用 `setStretchFactor`。

### 踩坑记录
- **"窗口能显示就以为布局对了"的错觉**：实际可能所有控件挤在 (0,0)。真验证 = 缩放窗口看是否重排、拖分隔条看是否响应。
- **`setStretch` 无效查 sizePolicy**：曾花时间调 stretch 发现无效，根因是按钮默认不扩展；改 `Expanding` 后立刻生效。两者必须配合。

### 与后续章节的衔接
1. **qt08 对象树**：布局里的控件父参数依赖它，否则内存/生命周期混乱。
2. **qt13 信号槽**：布局好的按钮、输入框是信号发射源，槽函数更新界面——布局是信号槽的"舞台"。
3. **qt20 绘图 / qt21 仪表**：自定义 `QWidget` 子类放进布局，配合 `resizeEvent` 自适应。

### 设计决策思考
- **为什么 Qt 用布局而不是 CSS 式约束？** 早期 Qt（1990s）目标是跨平台桌面，CSS Flex/Grid 那时不存在。布局管理器用 C++ 对象表达"关系"，编译期类型安全、运行时自动求解，比字符串样式表更可靠；Qt 后来也加了 QSS（类似 CSS）做外观，但**几何仍靠布局**，分工清晰。
- **为什么 `QSplitter` 是控件而非布局？** 因为它需要提供可见的、可拖拽的分隔条手柄，必须参与绘制与鼠标事件——这超出了纯几何求解器的职责。把它设计成 `QWidget`，既能独立使用也能嵌入布局，复用 Qt 已有的控件/事件体系，比硬塞进 `QLayout` 体系更自然。
