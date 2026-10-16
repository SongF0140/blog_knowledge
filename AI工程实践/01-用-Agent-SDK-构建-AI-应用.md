---
title: "用 Agent SDK 构建 AI 应用：从七个示例看懂一个生态"
published: 2026-09-04
description: "通过七个示例串起 Agent SDK 的完整能力面：多轮对话、多智能体、Tool Use、MCP 与 Hooks。"
category: AI工程
tags: ["Agent SDK", "Multi-Agent", "Tool Use"]
draft: false
---
# 用 CodeBuddy Agent SDK 构建 AI 应用：从七个示例看懂一个生态

> **关键词**：Agent SDK · 多轮对话 · 多智能体 · Tool Use · MCP · Hooks
> **配套代码**：本仓库 `quick-start` / `chat-demo` / `multi-turn-session` / `profile-builder` / `research-assistant` / `spreadsheet-assistant` / `mail-assistant`
> **日期**：2026-09-04

---

## 太长不看（TL;DR）

- Agent SDK 把「模型 + 工具 + 记忆 + 子智能体」打包成一个可直接调用的运行时，**一行 `query()` 就能跑起一个会自己上网、写文件、执行命令的智能体**。
- 同一套 SDK 提供两种语言入口：TypeScript（`@tencent-ai/agent-sdk`）与 Python（`codebuddy-agent-sdk`），心智模型一致：**查询（query）→ 会话（session）→ 编排（agents/hooks）**。
- 本仓库的 7 个示例正好串起一条能力进化线：**单次问答 → 多轮对话 → 会话恢复 → 工具与钩子 → 多智能体编排 → 全栈聊天应用 → AI 自举的邮件助理**。
- 如果你想快速上手：**做实验用 `quick-start`，做产品用 `chat-demo`，玩多智能体用 `research-assistant`，看天花板用 `mail-assistant`**。

---

## 一、SDK 是什么：不是「调用 API」，而是「托管一个智能体」

传统做法里，接入大模型要走一条很长的路：

```text
Prompt 工程 → 函数调用 Schema → 工具注册 → 循环调度
→ 上下文管理 → 子任务拆分 → 流式输出 → 错误恢复 → 安全护栏
```

而 Agent SDK 把这套「智能体运行时」直接下沉成一行代码：

```ts
// quick-start/quick-start.ts（摘录）
import { query } from '@tencent-ai/agent-sdk';

const q = query({
  prompt: '你好！请用简洁的方式告诉我你擅长做什么事情。',
  options: {
    maxTurns: 100,
    cwd: path.join(__dirname, 'agent'),
    allowedTools: ['Task', 'Bash', 'Glob', 'Grep', 'Read', 'Write', 'Edit',
                   'WebFetch', 'WebSearch', 'TodoWrite', 'KillBash'],
    debug: true,
  },
});

for await (const message of q) {
  if (message.type === 'assistant' && message.message) {
    const textContent = message.message.content.find((c: any) => c.type === 'text');
    if (textContent && 'text' in textContent) {
      console.log('智能体回复：', textContent.text);
    }
  }
}
```

`query()` 返回一个**异步流**：`assistant`（模型讲话）、`tool_use`（模型要调工具）、`result`（本轮结束、带计费与耗时）会依次涌出，你只需要决定「展示什么」。

Python 侧完全镜像：

```py
# research-assistant/research_agent/agent.py（摘录）
from codebuddy_agent_sdk import CodeBuddySDKClient, CodeBuddyAgentOptions

async with CodeBuddySDKClient(options=options) as client:
    await client.query(prompt=query_text)
    async for msg in client.receive_response():
        if type(msg).__name__ == 'AssistantMessage':
            process_assistant_message(msg, tracker, transcript)
```

> 值得注意：所有示例的模型配置注释都写着「不指定模型，使用默认配置」——SDK 会继承 CodeBuddy 的默认模型配置，让示例开箱即用，不必先为选模型纠结。

---

## 二、七个示例，一条能力进化线

| # | 示例 | 语言 | 一句话定位 | 关键技术点 |
|---|------|------|-----------|-----------|
| 1 | `quick-start` | TS | 最简问答 + 首个 Hook | `query()`、`allowedTools`、`PreToolUse` 钩子做写入白名单 |
| 2 | `multi-turn-session` | TS | 会话 API 四种玩法 | `unstable_v2_createSession` / `resumeSession` / `prompt` |
| 3 | `profile-builder` | TS | 让智能体「搜索→写文件→产出 .docx」 | `systemPrompt` + `Skill` 工具、自定义脚本目录约定 |
| 4 | `research-assistant` | Python | 一个只负责「分活」的研究主管 | `AgentDefinition` 子智能体、`hooks`、文件系统共享黑板 |
| 5 | `chat-demo` | TS | 可对话的全栈聊天应用 | Express + WebSocket + `query()` 流广播 + React |
| 6 | `spreadsheet-assistant` | Python | 会操作 Excel 的智能体 | `xlsx` 技能（openpyxl/pandas + LibreOffice 重算公式） |
| 7 | `mail-assistant` | TS（Bun） | AI 能自己造插件的邮件助理 | Actions / Listeners 热重载、MCP 邮箱工具、IMAP 实时监听 |

把它们排成一条线，就是 Agent 应用从玩具到生产力的**四个里程碑**：

```text
里程碑 ① 会聊天          → quick-start / multi-turn-session
里程碑 ② 会干活（工具）  → profile-builder / spreadsheet-assistant
里程碑 ③ 会分工（多体）  → research-assistant
里程碑 ④ 会自举（扩展）  → mail-assistant / chat-demo（全栈形态）
```

---

## 三、核心概念地图

示例再多，底层概念就五个：

### 1. Query（查询）：一次「带工具循环」的对话

`query()` 是原子单元：给定 prompt 与 options，SDK 内部自动完成 *思考 → 调用工具 → 观察结果 → 继续* 的循环，直到得出 `result`。`maxTurns` 是兜底上限，防止失控。

### 2. Session（会话）：跨轮次的记忆载体

`query()` 每次是独立的；要“记住上下文”，用 Session：

```ts
// multi-turn-session/examples.ts（摘录）
await using session = unstable_v2_createSession({}); // 不指定模型

await session.send('今年是哪一年？只回答年份数字。');
for await (const msg of session.stream()) { /* …第一轮… */ }

await session.send('再往后推10年是哪一年？只回答数字。'); // 它还记得上一轮
for await (const msg of session.stream()) { /* …第二轮… */ }
```

会话可以「挂起再恢复」：`createSession` 返回的流里有一条 `system/init` 消息携带 `session_id`，之后用 `unstable_v2_resumeSession(sessionId)` 就能带着全部记忆复活——这是做「上次聊到一半」类产品的基础（详见本系列第 4 篇）。

### 3. Tools（工具）：智能体的双手

`allowedTools` 白名单化地开放能力：

- **通用工具**：`Bash`、`Read`、`Write`、`Edit`、`Glob`、`Grep`、`WebSearch`、`WebFetch`…
- **编排工具**：`Task`（唤起子智能体）、`Skill`（加载领域技能）
- **自定义工具**：mail-assistant 用 `createSdkMcpServer` 注册了 `mcp__email__search_inbox`、`mcp__email__read_emails`，把 IMAP 邮箱封装成模型可调的工具——**MCP 协议让工具生态开箱即插**。

### 4. Hooks（钩子）：在工具调用前后插一道安全门

```ts
// quick-start/quick-start.ts（摘录）
hooks: {
  PreToolUse: [{
    matcher: 'Write|Edit|MultiEdit',
    hooks: [
      async (input: any) => {
        const filePath = input.tool_input.file_path ?? '';
        if (['.js', '.ts'].includes(path.extname(filePath).toLowerCase())
            && !filePath.startsWith(customScriptsPath)) {
          return {
            decision: 'block',               // ← 拦截
            stopReason: '脚本必须写入 custom_scripts 目录',
            continue: false,
          };
        }
        return { continue: true };
      },
    ],
  }],
},
```

钩子返回值用 `{ continue: true }` 放行、`{ decision: 'block', ... }` 拦截。上例把「模型写代码」约束进沙箱目录，**这是 Agent 上生产的第一个护栏**。

### 5. Agents + Skills（子智能体与技能）：把复杂度拆出去

- **Skill（技能）**：一段领域知识与脚本的打包（如 spreadsheet-assistant 的 `xlsx` 技能，内含 openpyxl/pandas 用法与 `recalc.py` 公式重算工具）；
- **Agent（子智能体）**：一个带独立 prompt、独立工具集、独立模型的“分身”，通过 `Task` 工具按需唤醒，配合 hooks 全程可观测（见第 2 篇）。

---

## 四、两种调用形态：CLI 与 Server

7 个示例也展示了智能体应用的两种主流形态：

### CLI 形态（进程内跑完即走）

```bash
# quick-start：跑一次对话
npx tsx quick-start.ts

# profile-builder：研究一个人并生成 docx 简历
npx tsx profile-builder.ts "张三"

# research-assistant：交互式研究，产出带图表的 PDF
uv run python research_agent/agent.py '量子计算最新进展'
```

这种形态把 `query()` 的返回值直接 `console.log`，几行代码就是一个小工具。

### Server 形态（长驻进程，面向多用户）

`chat-demo` 与 `mail-assistant` 把智能体包进 WebSocket 服务：

```text
浏览器 (React)
   │  ① subscribe { chatId }
   │  ② chat { content }
   ▼
WebSocket Server ──► ChatSession（一个聊天 = 一个会话）
   │                    │  query() 流式输出
   │                    ▼
   │              SDK Agent 运行时
   │  ③ 广播 assistant_message / tool_use / result
   ▼
浏览器实时渲染「思考过程 + 最终回答」
```

这里的关键设计是**订阅-广播**：一个 `ChatSession` 持有多个订阅者，把 SDK 流拆成 `user_message / assistant_message / tool_use / result / error` 五类事件推给前端——工具调用过程也能实时看到，而不是干等最后结果（详见第 3 篇）。

---

## 五、天花板：mail-assistant 的「AI 自举」

最后一个示例值得单独讲，因为它把 Agent 的边界推到了「**让 AI 自己扩展自己**」：

1. 智能体（模型）运行在 `agent/` 工作区，能读懂 `CODEBUDDY.md` 里声明的**子智能体**（如 `inbox-searcher`，自带“假设驱动迭代搜索”方法论）与**技能**（`skills/action-creator`、`listener-creator`，为它提供「造插件」的模板）；
2. 当模型被允许写代码时，`PreToolUse` 钩子强制它把生成的 `.ts` 脚本写进 `agent/custom_scripts/`——这个目录同时被服务端的 **Actions/Listeners Manager** 用文件 watcher 热加载；
3. 于是：用户说「把超过 30 天的促销邮件归档」→ 模型调用 `action-creator` 技能现场写一个 action 脚本 → 文件落盘 → 下一秒服务端已能执行它，并且**下次邮件进来，写好的 listener 会自动触发**。

```text
用户一句话 ──► 模型写代码 ──► custom_scripts/ 落盘
                                   │  watcher 热重载
                                   ▼
                Actions（点一下执行）/ Listeners（事件自动触发）
                                   │
              IMAP IDLE 收新邮件 ──► 自动处理 ──► SQLite / FTS5 检索
```

配套的还有 SQLite 邮件库（含 FTS5 全文索引）、WebSocket 双通道（REST 收邮件、WS 推聊天与 UI 状态）、React + Tailwind 前端。**它不是「邮件 + 一个聊天框」，而是「一个会持续值守、并且能自己学会新任务的员工」**。

---

## 六、怎么选：给四种角色的建议

| 你想做什么 | 从哪个示例起步 | 原因 |
|-----------|--------------|------|
| 只是想试试 SDK | `quick-start` | 代码最少，几分钟跑通第一个智能体 |
| 理解多轮/记忆机制 | `multi-turn-session` | 四个独立小例子逐层演示 send/stream/resume |
| 做一个真实产品 | `chat-demo` | 服务端-前端-会话管理一应俱全，可直接改造成产品骨架 |
| 研究/报告类自动化 | `research-assistant` | 多智能体编排范式最干净，Python 友好 |
| 深入工具/技能体系 | `profile-builder` + `spreadsheet-assistant` | 一个演示生成代码产出文件，一个演示领域技能封装 |
| 探索 Agent 上限 | `mail-assistant` | Actions/Listeners/MCP/自举，概念最全，适合做架构参考 |

---

## 七、结语

把七个示例连起来看，Agent SDK 想传达的产品哲学很清楚：

> **把「智能体的运行与治理」做成基础设施，让开发者专注于「业务、体验与边界」。**

- 心智负担被压缩：prompt 之外，你只需要声明工具白名单、钩子护栏、子智能体分工；
- 可观测性被内置：每一次 `tool_use`、每一个 `result` 都出现在流里，天然可追踪、可审计、可计费；
- 扩展性被打开：MCP 工具、Skill、自定义 Actions/Listeners，AI 的能力池可以持续生长。
