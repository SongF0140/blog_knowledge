---
title: "MCP：让 AI 连接外部世界"
published: 2026-09-04
description: "理解 MCP 的基本概念与接入方式，把数据库结构等外部数据源接进 AI 上下文，完成元数据驱动的代码生成闭环。"
category: AI工程
tags: ["MCP", "工具集成", "教程"]
draft: false
---
> 本文整理自第 7 讲《MCP：让 AI 连接外部世界》的学习与实践。
> 这篇笔记记录 MCP 的基本概念和接入过程，以数据库为例，说明如何让助手查询表结构，再据此编写代码。
> 完成标准：成功连接 1 个 MCP Server，并完成一次"元数据驱动代码生成"的最小闭环。

## 一、起点：AI 的"信息孤岛"问题

在接触 MCP 之前，先得承认一个事实——没有 MCP 的 AI，能力边界其实很窄。

**AI 能做的：**

- 读取项目文件
- 修改代码
- 运行终端命令
- 搜索项目内的代码

**AI 做不到的（需要外部连接能力）：**

- 查询数据库
- 调用第三方 API
- 读取实时数据
- 操作 GitHub Issues
- 访问 Figma 设计稿
- 查询生产环境日志

问题就在这里：AI 活在一个"封闭空间"里，只能看到你手动喂给它的东西。**MCP 的目标，就是打破这些限制，让 AI 能访问任何外部服务。**

## 二、什么是 MCP？

**MCP（Model Context Protocol，模型上下文协议）** 是由 Anthropic 提出的开放标准，用于连接 AI 模型与外部数据源和工具。

它的一套核心概念，用一个"浏览器上网"的类比就全懂了：

| 概念 | 类比 | 说明 |
|------|------|------|
| **MCP Host** | 浏览器 | AI 应用（如 CodeBuddy） |
| **MCP Client** | 网页的 JS | 在 Host 中负责连接 Server |
| **MCP Server** | Web 服务器 | 提供工具和数据的服务程序 |
| **Tool** | API 接口 | Server 暴露给 AI 的具体能力 |
| **Resource** | 网页内容 | Server 提供的数据（如数据库表结构） |
| **Prompt** | 页面模板 | Server 提供的预置 Prompt 模板 |

Host 是容器，Client 是连接器，Server 是能力提供方——和浏览器访问网站的结构几乎一模一样。

## 三、MCP 的工作流程

以"帮我写查询用户订单的 SQL"为例，完整流程是六步：

```text
1. AI 收到用户请求："帮我写查询用户订单的 SQL"
       ↓
2. AI 判断需要外部信息（数据库表结构）
       ↓
3. AI 调用 MCP Client → 请求 MCP Server
       ↓
4. MCP Server 连接数据库 → 读取表结构
       ↓
5. MCP Server 返回表结构信息给 AI
       ↓
6. AI 基于真实表结构编写准确的 SQL/代码
```

注意第 2 步和第 6 步：**AI 是自主判断"我缺什么信息"，并在拿到真实数据后才动笔写代码。** 这就是"元数据驱动代码生成"的含义——代码的依据不再是猜测，而是真实的表结构。

## 四、MCP 生态：常用 Server 与 API 的区别

### 常用 MCP Server 一览

| MCP Server | 连接目标 | 典型用途 |
|-----------|---------|---------|
| **Filesystem** | 本地文件系统 | 安全地访问指定目录 |
| **PostgreSQL / MySQL** | 数据库 | 读取表结构、执行查询 |
| **SQLite** | SQLite 数据库 | 本地数据库操作 |
| **GitHub** | GitHub API | 管理 Issues、PR、仓库 |
| **Brave Search** | 搜索引擎 | 实时网络搜索 |
| **Puppeteer** | 浏览器 | 网页截图、自动化测试 |
| **Memory** | 持久化记忆 | 跨对话记住信息 |
| **Fetch** | 任意 URL | 获取网页内容 |

### MCP 和直接调用 API 有什么区别

MCP 和直接调用 API 容易混淆，可以从四个方面对照：

| 维度 | 传统 API 调用 | MCP 协议 |
|------|-------------|---------|
| 调用者 | 人类写代码调用 | AI 自主判断何时调用 |
| 认知层 | 开发者需要知道 API 细节 | AI 通过 Tool 描述自动理解 |
| 交互方式 | 程序化、固定流程 | 对话式、灵活路由 |
| 集成成本 | 每个服务单独对接 | 统一协议，即插即用 |

一句话总结：**API 是"人写给机器调的"，MCP 是"机器自己看懂就会调的"。**

## 五、动手实操：配置 SQLite MCP Server

SQLite 是最简单的入门选择——让 AI 直接读取和操作本地数据库。整个过程四步。

### Step 1：写配置文件

在 Cloud Studio 中，配置文件位于项目根目录的 `/root/.codebuddy/mcp.json`：

```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-server-sqlite",
        "data/app.db"
      ]
    }
  }
}
```

### Step 2：重启 CodeBuddy

配置完成后必须重新加载，MCP 连接才会生效。这一步不能省——不重启，AI 那边永远看不到新工具。

### Step 3：创建示例数据库

让 Agent 建一个包含 4 张表的电商测试库 `data/app.db`：

```text
users 表：id, name, email, role, created_at
products 表：id, name, price, category, stock, created_at
orders 表：id, user_id(外键), total_price, status, created_at
order_items 表：id, order_id(外键), product_id(外键), quantity, price
```

我在这一步额外做了两件事，让测试数据更可信：

1. **给外键列手动建了索引**——SQLite 不会自动给外键列建索引，而后面生成 Prisma 恰好需要 `@@index` 注解，正好对上；
2. **订单总额由明细回算**——`total_price` 不是随手填的，而是用 `SUM(quantity * price)` 聚合后写回，最后跑 `PRAGMA foreign_key_check` 验证，全部通过。

### Step 4：验证连接

对 AI 说一句：

```text
请列出当前数据库中的所有表及其字段结构
```

如果 AI 通过 MCP Server 查询并返回了**真实的**表结构信息，链路就算彻底走通了。

## 六、核心实战：让 AI 直接读取数据库结构编写代码

### 传统流程 vs MCP 流程

```text
传统流程（繁琐）：
1. 你打开数据库客户端
2. 查看表结构
3. 手动复制到文档
4. 把文档喂给 AI
5. AI 基于文档写代码
6. 发现有遗漏，再回去查...

MCP 流程（丝滑）：
1. 你说"帮我写用户订单查询接口"
2. AI 自动查询数据库表结构
3. AI 基于真实表结构写代码
4. 一次走通（减少来回补信息）
```

### 场景 1：自动生成 Prisma Schema

提示词的关键是把要求说清楚：

```text
请通过 MCP 读取数据库的完整表结构，
然后生成对应的 Prisma Schema 文件（prisma/schema.prisma）。

要求：
1. 正确识别所有外键关系
2. 添加中文注释说明每个字段
3. 添加 @@index 和 @@map 注解
4. 输出"字段来源证据"（说明来自哪张表/字段）
```

其中"**字段来源证据**"是点睛之笔：要求 AI 为生成的每个字段标注它来自哪张表、哪个字段。这直接压缩了幻觉空间——编造的字段对不上真实表结构，一眼就能拆穿。

### 场景 2：自动生成 CRUD 接口

```text
请读取数据库中 users 表的结构，
然后在 src/app/api/users/ 目录下生成完整的 CRUD 接口：

1. GET /api/users     - 列表（分页 + 搜索）
2. GET /api/users/:id - 详情
3. POST /api/users    - 创建
4. PUT /api/users/:id - 更新
5. DELETE /api/users/:id - 删除

使用 Prisma 作为 ORM，每个接口有完整的类型安全和错误处理。
并附上每个接口的入参/出参示例。
```

AI 先查真实表结构，再写代码——接口里的字段名、类型、可空性全部来自数据库，而不是 AI 的"合理想象"。

### 场景 3：智能查询生成

```text
我需要一个报表查询：
"统计每个分类下的商品数量、平均价格、总销售额，按总销售额降序排列"。

请先查看相关表结构，确认字段名和关联关系，
然后生成 Prisma 查询代码，并解释每个聚合字段来自哪里。
```

### 进阶场景：数据库变更自动同步代码

这是我认为最能体现 MCP 威力的玩法：

```text
我刚在数据库中给 users 表添加了一个 phone 字段（varchar(20)，可空）。

请：
1. 通过 MCP 确认数据库当前 users 表结构
2. 更新 prisma/schema.prisma 中的 User 模型
3. 更新 src/types/user.ts 中的 TypeScript 类型
4. 更新相关 API（创建和更新接口支持 phone）
5. 更新前端表单组件，添加手机号输入框
6. 输出"变更影响清单 + 回归检查项"
```

改一处数据库，AI 感知变化后把 Schema、类型、接口、前端表单整条链路同步落地，还附上回归检查项。**AI 能直接感知数据库变化，并在代码侧同步落地**——这就是元数据驱动的完整形态。

## 七、接入时的检查项

接入一个 MCP Server 后，需要确认下面几件事：

- 说清楚 MCP 的作用、边界与基本工作方式
- 配置并连通一个可用的 MCP Server（例如 SQLite）
- 走通"读取元数据 → 生成代码 → 验证"的最小闭环
- 了解自定义 MCP Server 的基本思路
- 明白在密钥、权限与稳定性方面需要注意什么

这个例子里，MCP 的作用是把数据库查询能力提供给助手。它能读哪些表、能否写入，仍取决于 Server 暴露的工具、数据库权限和调用授权。

## 八、可继续尝试的例子

1. **SQLite MCP 配置**：建 4 张表的库 → 配 Server → AI 读表结构 → 生成 Prisma Schema
2. **MCP 驱动的 CRUD 生成**：基于练习 1 的库生成完整 CRUD 接口，并验证能正确运行
3. **编写自定义 MCP Server（进阶）**：如读取 Git 最近 20 条 commit、分析依赖树、读取 ESLint 结果