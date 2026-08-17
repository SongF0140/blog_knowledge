---
title: "Qt 22 SQLite 数据库 学习笔记"
published: 2026-08-17
description: "使用 Qt SQL 模块操作 SQLite 嵌入式数据库：建库、建表、增删改查、参数绑定、事务与模型视图，以及常见的连接/线程/路径坑。"
tags: ["Qt","C++","GUI","SQL","SQLite","数据库"]
category: Qt学习
draft: false
---

# Qt 22 SQLite 数据库 学习笔记

## 本章重要 API 函数

| API | 功能 | 参数说明 | 头文件 |
|-----|------|----------|--------|
| `QSqlDatabase::addDatabase("QSQLITE")` | 注册/获取一个 SQLite 数据库连接 | `const QString &type`：驱动名，SQLite 固定为 `"QSQLITE"`；可传第二个参数 `const QString &connectionName` 指定连接名（默认连接名 `defaultConnection`） | `<QSqlDatabase>` |
| `db.setDatabaseName(path)` | 设置数据库文件位置 | `const QString &name`：文件路径；传 `":memory:"` 表示内存数据库（进程退出即消失） | `<QSqlDatabase>` |
| `db.open()` | 打开数据库连接 | 无参数，返回 `bool`；**必须判返回值** | `<QSqlDatabase>` |
| `QSqlQuery::exec(sql)` | 执行一条 SQL | `const QString &query`：整条 SQL 字符串；返回 `bool` | `<QSqlQuery>` |
| `QSqlQuery::prepare(sql)` | 预编译带占位符的 SQL | 占位符用 `?`（位置绑定）或 `:name`（命名绑定）；返回 `bool` | `<QSqlQuery>` |
| `query.bindValue(...)` | 绑定参数 | `:name` 用 `bindValue(":name", val)`；`?` 用 `bindValue(pos, val)`（pos 从 0 开始） | `<QSqlQuery>` |
| `query.next()` | 移动到下一条结果 | 无参数，返回 `bool`；首次调用跳到第一条 | `<QSqlQuery>` |
| `query.value(col)` | 取当前行某列值 | `int` 列索引或 `const QString &name`；返回 `QVariant` | `<QSqlQuery>` |
| `db.lastError()` | 取最近错误 | 返回 `QSqlError`，含 `text()` / `driverText()` | `<QSqlDatabase>` / `<QSqlQuery>` |
| `db.transaction()/commit()/rollback()` | 事务控制 | 无参数，返回 `bool` | `<QSqlDatabase>` |
| `QSqlTableModel` | 表数据模型（可直接挂到视图） | 构造后可 `setTable()` + `select()`，支持 `insertRow/setData/submitAll` | `<QSqlTableModel>` |

> 💡 Tips：最容易踩的"API 误用"是忘记给工程文件 `.pro` 加 `QT += sql`（qmake）或在 CMake 里 `find_package(Qt6 COMPONENTS Sql)`——不挂 SQL 模块，`QSqlDatabase` 连头文件都找不到，链接期报 undefined reference。另一个坑：`addDatabase` 的第二个参数（连接名）在多线程下**必须**显式区分，否则多个线程拿到同一个 `defaultConnection` 会互相踩踏。

## 本章目录

> 点击条目可跳转到对应小节。

### 章节导航

- **[本章重要 API 函数](#本章重要-api-函数)**
- **[学习前后依赖](#学习前后依赖)**
- **[1. 概述](#1-概述)**
- **[2. 核心概念 / 原理](#2-核心概念--原理)**
- **[3. Qt SQL 操作 SQLite 总体流程](#3-qt-sql-操作-sqlite-总体流程)**
- **[4. 核心代码解析](#4-核心代码解析)**
- **[5. 运行效果 / 验证](#5-运行效果--验证)**
- **[容易踩坑](#容易踩坑)**
- **[学习要点总结](#学习要点总结)**
- **[Q&A（常见问题）](#qa常见问题)**
- **[Tips（全局速记）](#tips全局速记)**

---

## 学习前后依赖

| 方向 | 内容 |
| ---- | ---- |
| 前置基础 | [13 信号槽](13-signal-slot.md)（模型数据变化常用信号通知视图）、[20 绘图](20-painter-base.md)（把查询结果画到自定义控件）、[02 QWidget 基础](02-qt-widget-basic.md)（数据最终要显示在窗口上）。 |
| 本章核心 | 用 Qt SQL 模块打开 SQLite、建表、用参数化查询做增删改查、用模型绑定表格视图。 |
| 后续承接 | 可延伸至 `QSqlRelationalTableModel` 外键关联、`QSqlQueryModel` 自定义只读视图，以及把数据库与 [19 线程](19-thread-movetothread.md) 结合做后台读写避免界面卡顿。 |

> 💡 Tips：为什么把它放在最后？因为数据库操作本质是把"前面所有机制"串起来用——UI 显示靠 QWidget、交互靠信号槽、大数据渲染靠绘图、后台读写靠线程。先会前面 21 篇，这一篇自然就是"组合技"。

## 1. 概述

SQLite 是一个**进程内嵌入式关系数据库**，整个数据库就是一个文件，不需要独立的数据库服务器进程，非常适合桌面/嵌入式 Qt 程序的本地持久化（配置、缓存、离线数据、小型业务库）。Qt 通过 **Qt SQL 模块**统一封装了对多种数据库（SQLite、MySQL、PostgreSQL、ODBC 等）的访问，且对 SQLite 提供开箱即用的驱动（`QSQLITE`）。

本篇要解决的不是"SQL 语法怎么写"，而是"在 Qt 里怎么正确地打开库、安全地填参数、可靠地读结果、避免连接与线程雷区"。

> 💡 Tips：概述里最该记住的一句话——**`QSqlDatabase` 是"连接"的句柄，`QSqlQuery` 是"一次操作"的执行器；一个连接可以跑无数条 query，但 query 默认依附于当前默认连接的数据库**。把二者关系搞混，是 90% 初学者报 "no such table" 的根源（query 跑到了错误的连接上）。

## 2. 核心概念 / 原理

Qt SQL 的层次可以拆成四层：

1. **驱动层（QSqlDriver）**：真正和数据库对话的底层适配器。SQLite 驱动由 Qt 内置，只要 `QT += sql` 即可用，不需要额外安装客户端库（这点与 MySQL/PostgreSQL 不同）。
2. **连接层（QSqlDatabase）**：代表一条到数据库的连接。通过 `QSqlDatabase::addDatabase("QSQLITE")` 注册，同一个连接名全局唯一；用 `setDatabaseName` 指定文件，用 `open()` 真正建立。
3. **操作层（QSqlQuery）**：执行 SQL、遍历结果。可以 `exec(sql)` 一步执行，也可以 `prepare + bindValue + exec` 做参数化（强烈推荐，防 SQL 注入 + 正确处理引号/二进制）。
4. **模型层（QSqlTableModel / QSqlQueryModel）**：把查询结果包装成 Qt 的 model，直接 `setModel()` 挂到 `QTableView` 上，自动实现增删改与刷新，免手写表格绘制。

> 💡 Tips：原理上最容易误解的是"SQLite 是文件，所以随便哪个线程都能打开同一个文件读写"。错。SQLite 文件可以被多线程访问，但**同一个 `QSqlDatabase` 连接对象不是线程安全的**——每个线程必须拥有自己独立的连接（不同连接名）。Qt 会直接断言/报错如果你跨线程使用同一个连接。

## 3. Qt SQL 操作 SQLite 总体流程

```mermaid
flowchart TD
    A["QSqlDatabase::addDatabase(\"QSQLITE\")<br/>注册一个 SQLite 连接<br/>（多线程需指定 connectionName）"]
    B["db.setDatabaseName(\"demo.db\")<br/>文件路径；:memory: 为内存库"]
    C["db.open()<br/>打开连接，返回 bool"]
    D{"open 成功?"}
    L["db.lastError().text()<br/>打印错误，排查驱动/路径/权限"]
    E["QSqlQuery query(db)<br/>创建查询并绑定到该连接"]
    F["query.exec(\"CREATE TABLE IF NOT EXISTS ...\")<br/>建表 / DDL"]
    G["query.prepare(\"INSERT INTO t VALUES(?,?)\")<br/>+ bindValue + exec 增/改/删"]
    H["query.exec(\"SELECT ...\")<br/>查询"]
    I["while(query.next())<br/>逐行读取 value(col)"]
    J["db.transaction() / commit()<br/>批量写，提升性能"]
    K["db.close()<br/>析构时自动关闭，释放文件锁"]
    A --> B --> C --> D
    D -->|否| L
    D -->|是| E
    E --> F --> G --> H --> I --> K
    E -.可选批量.-> J
    J --> G
```

> 💡 Tips：流程里最关键的两个箭头是 `C --> D`（**一定要判 `open()` 返回值**，很多"打不开"其实是因为路径所在目录不存在或文件被占用）和 `E`（**构造 `QSqlQuery` 时把 `db` 传进去**，否则它默认用 `defaultConnection`，在多线程或你建了多个连接时就会跑错库）。另外 `CREATE TABLE` 几乎总是加 `IF NOT EXISTS`，否则第二次运行直接 `exec` 失败。

## 4. 核心代码解析

> 说明：本仓库 `qt22_tst_SQLite` 文件夹实际只含 `.idea` IDE 配置、**没有可参考的源码**，以下采用 Qt 操作 SQLite 的标准示例写法，覆盖本篇全部要点，可直接照搬进任何 Qt Widgets 工程。

```cpp
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlError>
#include <QVariant>
#include <QDebug>

void demoSqlite()
{
    // 1) 注册并打开连接（显式给连接名，便于多线程扩展）
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE", "conn1");
    db.setDatabaseName("demo.db");          // 文件库；换成 ":memory:" 即内存库
    if (!db.open()) {                        // ⚠️ 必须判返回值
        qDebug() << "打开失败:" << db.lastError().text();
        return;
    }

    // 2) 建表（IF NOT EXISTS 避免重复运行报错）
    QSqlQuery query(db);                     // ⚠️ 把 db 传进去，确保跑在这条连接上
    query.exec("CREATE TABLE IF NOT EXISTS user ("
               "id INTEGER PRIMARY KEY AUTOINCREMENT, "
               "name TEXT NOT NULL, age INTEGER)");

    // 3) 参数化插入（? 占位，bindValue 位置从 0 开始）
    query.prepare("INSERT INTO user(name, age) VALUES(?, ?)");
    query.bindValue(0, "Alice");
    query.bindValue(1, 20);
    if (!query.exec()) qDebug() << "插入失败:" << query.lastError().text();

    // 4) 查询并遍历
    if (query.exec("SELECT id, name, age FROM user")) {
        while (query.next()) {
            int id   = query.value(0).toInt();
            QString n = query.value(1).toString();
            int age  = query.value(2).toInt();
            qDebug() << id << n << age;
        }
    }

    // 5) 命名绑定 + 更新
    query.prepare("UPDATE user SET age = :age WHERE name = :name");
    query.bindValue(":age", 21);
    query.bindValue(":name", "Alice");
    query.exec();

    db.close();   // 析构时也会自动关闭
}
```

> 💡 Tips：这段代码里藏着 4 个细节——① `addDatabase` 的第二个参数 `"conn1"` 在多线程时是救命稻草；② `QSqlQuery query(db)` 的 `db` 实参，漏写就会默认连到 `defaultConnection`，若你之前用默认连接打开过别的库就会串库；③ `AUTOINCREMENT` 让 `id` 自增，查询回来用 `value(0).toInt()` 取；④ `bindValue` 对 `?` 用数字下标（从 0），对 `:name` 用字符串名——**两套不能混用**，否则绑定不生效、SQL 里留着 `?` 直接报错。

## 5. 运行效果 / 验证

正确运行后：控制台按行打印 `1 Alice 20`（插入后查询），执行 UPDATE 后再次查询应为 `1 Alice 21`；工程目录下出现 `demo.db` 文件（用 DB Browser for SQLite / `sqlite3` 命令行打开能看到 `user` 表与数据）。若用 `:memory:`，则程序退出后库消失，文件不会留下。

最低验证步骤：
1. `.pro` 加 `QT += sql` 或 CMake `find_package(Qt6 COMPONENTS Sql REQUIRED)` + `target_link_libraries(xx Qt6::Sql)`；
2. 编译运行，观察 `qDebug` 输出与 `open()` 是否成功；
3. 用外部 SQLite 工具打开 `demo.db` 确认表真实写入。

> 💡 Tips：验证时最容易误判"成功"的假象是——**代码没崩、也没打印 lastError，就以为写入了**。其实 `exec()` 返回 `false` 但你可能没检查返回值，数据根本没进库。务必养成"每个 `exec`/写入后查 `lastError` 或判返回值"的习惯；更稳妥的是用外部工具打开 `.db` 文件做"双保险"验证。

## 容易踩坑（注意点）

- **坑 1：没加 `QT += sql`**。现象：编译报 `QSqlDatabase: No such file or directory` 或链接 `undefined reference to QSqlDatabase`。原因：SQL 模块未引入。规避：qmake 工程 `.pro` 加 `QT += sql`；CMake 工程 `find_package(Qt6 COMPONENTS Sql)` 并链接 `Qt6::Sql`。
- **坑 2：`open()` 不判返回值**。现象：后面 `exec` 全部失败、`lastError` 显示 "out of memory" 或空。原因：连接根本没打开（路径目录不存在、文件被别的进程锁住、权限不足）。规避：永远 `if (!db.open()) { qDebug() << db.lastError().text(); return; }`。
- **坑 3：跨线程共用同一连接**。现象：偶发崩溃或 `QSqlDatabasePrivate::removeDatabase` 断言。原因：`QSqlDatabase` 非线程安全。规避：每个线程 `addDatabase("QSQLITE", uniqueName)` 各自建连接，不要传递连接对象跨线程。
- **坑 4：参数绑定混用 `?` 与 `:name`**。现象：SQL 执行报错 "near ?: syntax error" 或值没写进去。原因：`bindValue` 下标与占位符类型不匹配。规避：`?` 用 `bindValue(int, val)`（从 0），`:name` 用 `bindValue(":name", val)`，二选一保持一致。
- **坑 5：忘记给 `QSqlQuery` 传连接**。现象：`no such table`（表确实建了，但 query 跑在另一条连接上）。原因：没把 `db` 传给 `QSqlQuery` 构造。规避：`QSqlQuery query(db);` 显式绑定。
- **坑 6：文件路径用相对路径结果找不到库**。现象：每次运行都像"新建空库"。原因：相对路径相对于程序工作目录（调试时往往是构建目录而非源码目录）。规避：用 `QCoreApplication::applicationDirPath() + "/demo.db"` 或绝对路径，或用 `:memory:` 明确语义。

## 学习要点总结

### 核心知识点
- Qt SQL 四层：驱动 → 连接(`QSqlDatabase`) → 操作(`QSqlQuery`) → 模型(`QSqlTableModel`)。
- SQLite 是单文件嵌入式库，`QSQLITE` 驱动内置，无需额外服务端。
- 参数化查询（`prepare` + `bindValue`）是写入数据的标准做法，防注入且正确处理类型。
- 连接非线程安全，多线程需各自独立连接名。

### 关键 API
- `QSqlDatabase::addDatabase("QSQLITE"[, name])`、`setDatabaseName`、`open()`
- `QSqlQuery::exec / prepare / bindValue / next / value`、`lastError()`
- `QSqlTableModel` 直接挂 `QTableView` 实现可视化增删改查

### 下一步学习
- `QSqlTableModel` + `QTableView` 做真正的表格 CRUD 界面；
- `QSqlRelationalTableModel` 处理外键关联显示；
- 结合 [19 线程](19-thread-movetothread.md) 把耗时查询丢到子线程，主线程只更新 UI。

> 💡 Tips：把本篇和知识地图里"下一步"那篇直接挂钩——[19 线程](19-thread-movetothread.md) 解决"大查询卡界面"，[20 绘图](20-painter-base.md) 解决"把查询结果画成图表"，[13 信号槽](13-signal-slot.md) 解决"数据变了自动通知视图刷新"。三者组合起来，就是一个能落地的本地数据应用。

## Q&A（常见问题）

**Q1：为什么我 `exec("INSERT ...")` 没报错，但数据库里却没有数据？**
A1：最常见两种原因——① 没检查 `exec()` 返回值，实际返回 `false`，错误被你忽略了（务必 `if(!query.exec()) qDebug()<<query.lastError().text();`）；② 你 `QSqlQuery` 没传 `db`，它跑在了默认的空连接上，根本没写进你打开的那个库。用外部 SQLite 工具打开 `.db` 文件做二次确认最稳妥。

**Q2：`QSqlDatabase::addDatabase` 的"连接名"到底有什么用？**
A2：Qt 维护一个"连接名 → 连接对象"的全局映射，默认名是 `defaultConnection`。单线程单库时不用管；但当你需要**同时开多个库**，或**多线程各开一条连接**时，必须给不同连接名（如 `"conn1"`、`"conn2"`），否则后 `addDatabase` 的会覆盖前面的，导致旧连接失效甚至断言崩溃。`QSqlQuery` 构造时传入对应 `QSqlDatabase` 对象，才能精确指定跑在哪条连接上。

**Q3：`?` 和 `:name` 两种占位符能混着用吗？**
A3：不能。一条 SQL 里要么全用 `?`（位置绑定，`bindValue(0, v)`、`bindValue(1, v)`……从 0 开始），要么全用 `:name`（命名绑定，`bindValue(":name", v)`）。混用会导致绑定顺序错位或根本不生效，`exec` 报错。推荐用 `:name`，可读性更好、参数顺序不易错。

**Q4：SQLite 在 Qt 里是线程安全的吗？可以直接在子线程读写吗？**
A4：SQLite 文件本身支持多线程访问，但 **Qt 的 `QSqlDatabase` 连接对象不是线程安全的**。正确做法：在子线程里**重新** `addDatabase("QSQLITE", 该线程专属名)` 并建立自己的连接，再通过信号槽把结果回传主线程更新 UI（见 [19 线程](19-thread-movetothread.md)）。绝对不要把一个在主线程建立的 `QSqlDatabase` 直接丢给子线程用。

**Q5：内存数据库 `:memory:` 和文件数据库有什么区别？**
A5：`:memory:` 整个库存在 RAM，进程退出即销毁，常用于单元测试、临时计算；文件库（如 `demo.db`）持久化到磁盘。注意：**每个 `:memory:` 连接是相互独立的私有库**，即使连接名相同，两个 `:memory:` 连接也看不到彼此的表——想要共享内存库，需要同一个连接对象或 `file::memory:?cache=shared`。

## Tips（全局速记）

### 最容易忽视的细节
- `open()` 必须判返回值；`exec()` 最好也判返回值并打印 `lastError()`。
- `QSqlQuery` 构造时把 `db` 传进去，避免默认连接串库。
- 建表一律 `IF NOT EXISTS`，让程序可重复运行。

### 踩坑记录
- 没加 `QT += sql` → 编译/链接失败（出现频率最高）。
- 相对路径的 `.db` 落在构建目录而非源码目录，看起来"每次都新建空库"。
- 多线程共用 `defaultConnection` → 偶发崩溃。

### 与后续章节的衔接
- 与 [13 信号槽](13-signal-slot.md)：模型 `dataChanged` 等信号用于自动刷新视图。
- 与 [19 线程](19-thread-movetothread.md)：耗时查询移至子线程，避免 GUI 卡顿。
- 与 [20 绘图](20-painter-base.md)：把查询出的数值绘制成曲线/仪表。

### 设计决策思考
- 小型单机应用优先 SQLite（零部署、单文件、事务可靠）；需要多机并发写入再考虑 MySQL/PostgreSQL（Qt 切换驱动即可，代码几乎不变，这是 Qt SQL 抽象的最大价值）。
- 只读展示用 `QSqlQueryModel`，需要编辑用 `QSqlTableModel`，复杂联表再用 `QSqlQuery` + 自定义 model，按需求复杂度选层，不要一开始就用最重的方案。
