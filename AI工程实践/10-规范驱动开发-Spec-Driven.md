---
title: "规范驱动开发（Spec-Driven）实践"
published: 2026-09-04
description: "不讲概念只讲操作：每一步怎么做、每个模块为什么存在、生成的项目长什么样、如何与规范体系交互编写。"
category: AI工程
tags: ["Spec-Driven", "规范驱动", "Next.js"]
draft: false
---
> 本文不讲概念科普，只回答四个问题：**每一步怎么做**、**每个模块为什么存在**、**项目生成出来长什么样**、**如何与 Speckit 的规范体系交互编写**。
>
> 环境：Node v22.13.1 / npm 11.6.2 / Next.js 16.3.4 / AI 工具 CodeBuddy
> 校验来源：本地 `specify init --help` 输出 + spec-kit 官方仓库 templates 目录（v1.0.0）

---

## 一、先建立心智模型：Speckit 在改什么

AI 编程的问题不是「写不出代码」，而是「写得不稳定」：

1. **需求与实现脱节**——需求说一套，代码做一套。
2. **多人协作风格混乱**——提示词各异，产出像拼接工程。
3. **结果不可复现**——同样需求，不同时间结果差异很大。

Speckit 的做法是把流程从「直接让 AI 写代码」改成「先写规约，再让 AI 按规约执行」。一句话：**它是 AI 时代的工程化护栏系统**。

由此产生三条贯穿全篇的原则，后面所有模块设计都源自它们：

| 原则 | 含义 | 直接后果 |
|---|---|---|
| **规约驱动** | 先定义「做什么、不能做什么、验收标准」，再实现 | 维护软件＝维护规约；调试＝修复规约；重构＝重构规约 |
| **宪法至上** | 用 `constitution.md` 锁死技术与质量底线 | 防止 AI 在关键决策上自由发挥 |
| **先规划后施工** | 需求 → 方案 → 任务 → 实现，逐层收敛 | 产出不满意时改上游文档，而不是直接改代码 |

---

## 二、安装与初始化：每条命令、每个参数的意义

### 2.1 安装 Specify CLI

```bash
# 方式 1：持久化安装（推荐）
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git

# 方式 2：一次性使用
uvx --from git+https://github.com/github/spec-kit.git specify init my-project
```

**为什么用 `uv tool install`**：Speckit 的核心交付物是**模板文件和脚本**，不是运行库。CLI 只在初始化那一刻被需要，之后真正驱动流程的是 AI agent 读取 `.specify/` 下的模板。所以持久化安装只为了方便，一次性执行也完全可行。

生产环境建议**锁定版本**，避免模板结构随版本漂移导致团队产出不一致：

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
```

### 2.2 初始化项目

```bash
specify init my-project
```

**这一步实际做了三件事**，理解它才能理解后续目录：

1. **下载模板包**——从 spec-kit release 拉取与你选择的 AI 工具匹配的模板 + 命令文件
2. **解压落地**——写入 `.specify/`（模板与脚本）和 agent 命令目录（如 `.claude/commands/`、CodeBuddy 的命令目录）
3. **初始化 Git 仓库**——让规约从第一天起就进版本控制

### 2.3 参数逐个解释

以下参数来自本地 `specify init --help` 实测输出：

| 参数 | 作用 | 什么时候用 |
|---|---|---|
| `--ai <name>` | 指定 AI 工具（codebuddy / claude / copilot / cursor-agent / gemini / codex …） | **必用**，跳过交互式选择器 |
| `--script <sh\|ps>` | 生成脚本的类型 | Unix 用 `sh`，Windows PowerShell 用 `ps` |
| `--here` | 在当前目录初始化，不新建子目录 | **存量项目接入时必用** |
| `--force` | 目录非空时跳过确认直接合并 | 配合 `--here` 用于已有项目 |
| `--no-git` | 跳过 Git 初始化 | 项目已有 Git 仓库时（大多数存量场景） |
| `--ignore-agent-tools` | 跳过 AI 工具是否安装的检查 | CI 环境或工具不在 PATH 中 |
| `--ai-skills` | 以 agent skills 形式安装而非 slash 命令文件 | 需要技能化形态时 |
| `--github-token <token>` | 指定 GitHub token 读取 release | 遇到 API 限流时（见第十章） |
| `--ai-commands-dir <dir>` | 自定义命令输出目录 | 配合 `--ai generic` 使用 |
| `--skip-tls` | 跳过 TLS 校验 | 仅内网自签证书环境，不推荐 |
| `--debug` | 输出网络与解压的详细诊断 | 初始化失败排查 |

**推荐写法**（把参数写全，避免卡在交互选择器上）：

```bash
# 新项目
specify init my-project --ai codebuddy --script sh

# 存量项目（已有代码 + 已有 Git）
specify init . --ai codebuddy --script sh --no-git --force
```

**版本差异提醒**：spec-kit 1.0.0 的官方 README 使用 `--integration <name>`（如 `--integration copilot`），并提供 `--non-interactive` 用于无键盘的 CI 环境。本地安装的旧版 CLI 使用 `--ai`。**以你机器上 `specify init --help` 的实际输出为准**——这是最可靠的判断方式。

### 2.4 初始化之后你要做的第一件事

进入项目目录，启动你的 AI 工具：

```bash
cd my-project
```

此时 agent 已经能识别 `/speckit.*` 系列斜杠命令。**命令不是 CLI 提供的，而是初始化时写进 agent 命令目录的文件**——这点很关键，它决定了：

- 换个 AI 工具，就要重新 `specify init` 或用对应参数；
- 你可以直接编辑这些命令文件来改造行为（见第九章）。

---

## 三、项目结构详解：每个模块为什么存在

这是本文的核心章节。下面结构基于 spec-kit v1.0.0 的 templates 目录与初始化行为整理，**具体文件以你实际生成的版本为准**。

```
my-project/
├── .specify/                      # Speckit 的"规则与模板中心"
│   ├── memory/
│   │   └── constitution.md        # 项目宪法：全局约束，所有阶段都读它
│   ├── templates/                 # Spec Kit 核心模板（优先级最低）
│   │   ├── commands/              # 各命令对应的提示词模板
│   │   ├── constitution-template.md
│   │   ├── spec-template.md
│   │   ├── plan-template.md
│   │   ├── tasks-template.md
│   │   └── checklist-template.md
│   ├── extensions/                # 扩展：新增能力（可选）
│   │   └── templates/
│   ├── presets/                   # 预设：改写现有模板（可选）
│   │   └── templates/
│   └── scripts/                   # 辅助脚本（新建功能分支、初始化计划等）
├── specs/                         # 规约产物：真正随业务演进的资产
│   └── 001-feature-name/
│       ├── spec.md                # 需求（WHAT）
│       ├── plan.md                # 技术方案（HOW）
│       ├── tasks.md               # 任务清单
│       └── （plan 阶段可能附带 data-model.md / research.md / contracts/）
├── .codebuddy/ 或 .claude/        # agent 命令目录（由 --ai 决定）
│   └── commands/                  # /speckit.* 命令文件本体
└── （你的业务代码）
```

### 3.1 `.specify/memory/` —— 项目的长期记忆

存放 `constitution.md`。叫 memory 是有道理的：**这是唯一一个不随功能切换、贯穿项目全生命周期的文件**。

它的内容质量直接决定 AI 产出质量。对比一下：

```diff
- 代码要规范，注意用户体验
+ 必须通过 ESLint 且无 error；界面文案全部使用简体中文
+ 不使用蓝紫渐变风格；所有数据仅本地存储，禁止上传云端
+ 核心逻辑必须有单元测试，覆盖率不低于 60%
```

**判定标准**：至少 5 条**可检查**的规则，而不是口号。

### 3.2 `.specify/templates/` —— 产物的格式契约

`spec-template.md`、`plan-template.md`、`tasks-template.md` 等，决定了 AI 生成的 `spec.md` / `plan.md` / `tasks.md` **长什么样**。

这解释了一个常见疑惑：为什么不同项目生成的 `spec.md` 章节结构一模一样？因为 AI 是**填空**，模板定义了填空的格子。`checklist-template.md` 则对应「像单元测试之于代码一样，检查英文规约本身的完整性」。

**这意味着**：你想让全团队的 spec 都带「异常场景」章节，改模板比在每次对话里重复要求有效得多。

### 3.3 四级模板优先级（理解定制化的钥匙）

Speckit 运行时按**自上而下**查找模板，命中第一个就停止：

| 优先级 | 类型 | 位置 | 适用场景 |
|---|---|---|---|
| 1（最高） | 项目本地覆盖 | `.specify/templates/overrides/` | 单个项目的一次性调整 |
| 2 | 预设 Presets | `.specify/presets/templates/` | 组织级标准、术语本地化 |
| 3 | 扩展 Extensions | `.specify/extensions/templates/` | 新增能力带来的新模板 |
| 4（最低） | Spec Kit 核心 | `.specify/templates/` | 默认行为 |

两个容易踩的细节：

- **模板在运行时解析**（每次执行命令时现找），所以改完模板**下次执行即生效**，无需重新 init；
- **扩展/预设的命令在安装时写入 agent 目录**，所以 `specify extension add` 之后命令才出现。

### 3.4 `specs/` —— 随业务演进的功能资产

每个功能一个目录，通常带序号前缀（`001-`、`002-`）以体现创建顺序。

| 文件 | 回答的问题 | 谁写的 | 能否手改 |
|---|---|---|---|
| `spec.md` | 要做什么（WHAT / WHY） | `/speckit.specify` + `/speckit.clarify` | 可以，且鼓励 |
| `plan.md` | 怎么做（HOW） | `/speckit.plan` | 可以 |
| `tasks.md` | 按什么顺序做 | `/speckit.tasks` | **可以，这是核心协作界面** |
| `checklists/` | 需求是否写清楚了 | `/speckit.checklist` | 可以 |

`tasks.md` 被称为「与 AI 协作的核心界面」：你随时可以手动改顺序、调优先级、标记完成后再让 AI 继续。

### 3.5 哪些该进 Git

- **必须提交**：`.specify/memory/`、`specs/`、模板覆盖与预设。它们是团队的共同契约。
- **按需提交**：agent 命令目录（通常建议提交，保证团队一致）。
- **不必提交**：`.specify/scripts/` 中由 CLI 管理的部分、临时研究产物。

spec-kit 官方自己就把 `.specify/` 与 `specs/` 下的 dogfooding 产物做了 gitignore 区分，普通项目不必照搬，但**「工具更新」与「规约演进」应当分开提交**这条原则值得借鉴：升级 CLI 改的是工具，改 `specs/` 改的是预期行为，混在一个 commit 里会让规约历史难以追溯。

---

## 四、七条命令与产物的映射

| 命令                      | 阶段   | 产出                                | 是否必做    |
| ----------------------- | ---- | --------------------------------- | ------- |
| `/speckit.constitution` | 立规矩  | `.specify/memory/constitution.md` | 每项目一次   |
| `/speckit.specify`      | 写需求  | `specs/xxx/spec.md`               | 必做      |
| `/speckit.clarify`      | 补边界  | 更新 `spec.md`                      | 强烈建议    |
| `/speckit.plan`         | 定方案  | `plan.md`（+ 数据模型等）                | 必做      |
| `/speckit.tasks`        | 拆任务  | `tasks.md`                        | 必做      |
| `/speckit.analyze`      | 查一致性 | 分析报告                              | 中大型项目建议 |
| `/speckit.implement`    | 写代码  | 可运行代码                             | 必做      |

1.0.0 新增的三个命令，教程里还没覆盖：

| 命令 | 作用 |
|---|---|
| `/speckit.converge` | 拿代码对照 spec/plan/tasks 做收敛评估，把剩余工作**追加为新任务**；官方建议反复执行 implement → converge 直到报告 `Converged` |
| `/speckit.checklist` | 生成质量检查清单，校验需求本身的完整性、清晰度、一致性 |
| `/speckit.taskstoissues` | 把 `tasks.md` 转成 GitHub Issues 做跟踪 |

> 命名提示：官方 README 的 SDD Quickstart 里写作 `/speckit-constitution`（连字符），命令表中写作 `/speckit.constitution`（点号），两者指向同一能力；CodeBuddy 环境使用点号形式。

---

## 五、逐步操作：每一步的意义、写法与验收

每步统一回答五件事：**这一步在解决什么问题 → 怎么输入 → 产出什么 → 验收标准 → 不合格怎么办**。

### Step 1：`/speckit.constitution` 制定宪法

**意义**：把「你不希望 AI 自由发挥的地方」一次性写死。后面每一次生成，AI 都会先读它。

```
/speckit.constitution 这是一个 Nextjs 框架的问卷调研系统
```

AI 依据 `constitution-template.md` 生成后务必自己审一遍，典型产出：

```
- 所有数据只存本地，采用 Mock 的方式，不上传云端，计算过程透明
- 面向普通用户，交互简单直观
- 每个功能都能独立测试、渐进上线
```

- **验收**：≥5 条可检查规则，覆盖技术栈、测试、UI/语言、安全边界
- **不合格**：把「代码要规范」改成「必须通过 ESLint 且无 error」
- **存量项目**：直接执行 `/speckit.constitution`，让 AI 读现有代码反向生成

### Step 2：`/speckit.specify` 描述需求

**意义**：产出 PRD。核心纪律是**只写 WHAT，不写 HOW**——一旦在这里写技术选型，Step 4 的 plan 就被架空了。

```markdown
/speckit.specify 我要做一个问卷系统。
参考腾讯问卷，至少支持问卷新建、删除。
UI 设计要有活力感一点。
当完成问卷生成后，给一个大大的鼓励动画。
```

- **验收**：包含主流程、边界条件、异常场景
- **不合格**：删掉技术实现细节，改写成「用户行为 + 系统反馈」。例如把「用 LocalStorage 存储」改成「刷新页面后数据不丢失」

### Step 3：`/speckit.clarify` 澄清需求

**意义**：让 AI 反过来审你的 spec，把模糊点问出来。这是**成本最低的返工预防**——在这里花 5 分钟，能省掉后面几小时的返工。

```
/speckit.clarify 进行需求澄清，有无不足或未考虑到的地方
```

无需额外输入，AI 会提出若干问题（通常是五个），你逐一回答。

- **验收**：关键歧义已关闭（空值怎么处理、排序规则、删除是否软删、冲突时优先级）
- **不合格**：对每个问题给出「明确规则 + 示例」，不要回答「都行」

> 官方建议在 `/speckit.plan` **之前**执行 clarify。每个 `spec.md` 里还有一个 `Review Checklist` 章节，可用来检查 AI 交付物是否达标。

### Step 4：`/speckit.plan` 生成技术方案

**意义**：把 WHAT 翻译成 HOW。这一步 AI 会同时读宪法和需求，所以技术选型不会跑偏。

```
/speckit.plan 执行
```

- **验收**：方案完整覆盖 `spec.md` 的核心需求
- **不合格**：先补齐缺失模块再重跑 `plan`
- **重要权利**：AI 选的库你不认可，直接要求改——**你拥有最终决策权**，改的是 `plan.md` 而不是代码

### Step 5：`/speckit.tasks` 拆解任务

**意义**：把方案切成可独立验收、有依赖顺序的施工步骤。粒度决定了 Step 7 的可控性。

```
/speckit.tasks 生成详细任务列表
```

典型产出：

- **Phase 1**：项目搭建（初始化结构、配置 ESLint/Prettier、基础组件）
- **Phase 2**：核心功能（数据模型、新增、删除、标记完成、状态筛选）
- **Phase 3**：数据持久化
- **Phase 4**：测试与优化

- **验收**：每个任务都有目标与完成判定，且可按顺序执行
- **不合格**：拆分过大的任务，补上验收动作

### Step 6：`/speckit.analyze` 一致性检查

**意义**：前五步已经产出大量文档，靠人眼很难保证 `spec.md`、`plan.md`、`tasks.md` 彼此一致。这一步就是**返工预防器**。

```
/speckit.analyze 进行全面分析
```

- **验收**：无重大冲突
- **不合格按类型回改**：需求冲突 → 改 `spec.md`；技术不可行 → 改 `plan.md`；任务覆盖不足 → 重生 `tasks.md`

> 执行时机：官方建议放在 `/speckit.tasks` 之后、`/speckit.implement` 之前。

### Step 7：`/speckit.implement` 落地实现

```
/speckit.implement 开始写代码
```

AI 会按顺序执行 `tasks.md`，每完成一个任务就在文档里打勾，遇到测试失败自动修复，全部完成后提示。

- **验收**：主流程可演示，关键边界可验证
- **不合格**：**回到上游文档修正后重新实现**，不要直接改代码

### Step 8（1.0.0）：`/speckit.converge` 收敛

**意义**：实现完不等于对齐。`converge` 拿真实代码对照 spec/plan/tasks 评估，把没做完的追加成新任务。

```
/speckit.converge
```

官方推荐循环：**implement → converge → implement → converge …… 直到报告 `Converged`**。

---

## 六、实战 A：从 0 到 1 做一个 Todo 应用

**需求目标**：新增任务（标题+描述）、删除任务、标记完成、状态筛选、LocalStorage 持久化。

**完整操作序列**：

```bash
# Step 0 初始化
specify init todo-app --ai codebuddy --script sh
cd todo-app
```

```
# Step 1 宪法
/speckit.constitution 这是一个任务清单管理系统
```

AI 生成示例：

```
- 中文界面，所有文案使用简体中文
- 不使用"AI 味"泛滥的蓝紫渐变，采用清新简洁的设计风格
- 代码风格统一，配置 Lint/Format 工具
- 基础测试必须存在，确保核心功能稳定
- 纯前端实现，使用 LocalStorage 持久化数据
```

```
# Step 2 需求
/speckit.specify 我要做一个任务清单应用，包含以下功能：
1. 新增任务：支持输入任务标题和详细描述
2. 删除任务：可以删除不需要的任务
3. 标记完成：点击可切换任务的完成状态
4. 状态筛选：可按"全部/未完成/已完成"筛选任务显示
5. 数据持久化：使用 LocalStorage 保存数据，刷新页面后数据不丢失
UI 要求：界面简洁清新，避免使用蓝紫渐变；任务列表清晰易读；操作按钮直观易用
```

```
# Step 3～7
/speckit.clarify 进行需求澄清
/speckit.plan 执行
/speckit.tasks 生成详细任务列表
/speckit.analyze 进行全面分析
/speckit.implement 开始写代码
```

**验收清单**：

- [ ] 功能完整可用（新增、删除、标记、筛选）
- [ ] 刷新页面后数据不丢失
- [ ] 界面文案为中文，设计风格清新
- [ ] 控制台无关键报错信息
- [ ] 基础测试通过

**常见问题**：

- **改任务优先级** → 直接编辑 `specs/xxx/tasks.md` 调整顺序，再执行 `/speckit.implement`
- **中途暂停** → 打断对话，下次用 `/speckit.implement --continue` 继续
- **清空 LocalStorage** → 浏览器 DevTools 的 Application 面板，或在代码里加清空按钮

---

## 七、实战 B：存量项目（Brownfield）的缺陷修复

**场景**：页面某区域图片显示异常、产品项缺失。

### 关键差异

存量项目用 `--here` 在当前目录初始化，**并且通常要加 `--no-git`**（项目已有仓库）和 `--force`（目录非空）：

```bash
specify init . --ai codebuddy --script sh --no-git --force
```

Speckit 会在既有代码上叠加 `.specify/` 与 `specs/`，**不破坏原有代码**。

### 简化流程

```
1. specify init .                     # 接入
2. /speckit.specify  描述缺陷与期望   # 现状 → 期望 → 验收标准
3. /speckit.plan     输出修复策略
4. /speckit.tasks    生成修复任务
5. /speckit.implement 落地修复
6. 回归验证 + 对照需求确认
```

**描述缺陷的写法模板**（把现状和期望分开，AI 定位根因的准确率会明显提高）：

```markdown
/speckit.specify 发现两个问题需要修复：

1. 产品列表页面，图片显示异常
   - 现状：图片区域显示为空白或破碎图标
   - 期望：图片正常显示，支持懒加载

2. 产品数据缺失
   - 现状：某些产品项在列表中不显示
   - 期望：所有产品项都能正确展示

验收标准：
- 图片加载正常，无破碎图标
- 产品列表数据完整，无遗漏
- 控制台无图片加载错误
```

任务通常被拆成：定位（分析加载逻辑、检查查询代码）→ 修复（修正路径、修复过滤逻辑）→ 验证（手动测试、完整性、回归）。

### Brownfield 的两条纪律

1. **工具更新与规约演进分开**：升级 CLI 是一个 commit，改 `specs/` 是另一个 commit。混在一起会让「预期行为变更」无法追溯。
2. **修完对照原需求确认**：价值在于把「拍脑袋修 bug」变成有依据的修复闭环——缺陷描述、根因、策略、回归清单都留在仓库里，下次同类问题可直接复用。

---

## 八、如何与 Speckit 的规范体系交互编写

这一章回答「怎么改造 Speckit 让它产出符合我团队习惯的东西」。

### 8.1 三个层级的介入方式

| 你想改变什么 | 手段 | 位置 |
|---|---|---|
| 某个项目的产物格式 | 项目本地覆盖 | `.specify/templates/overrides/` |
| 全组织的标准/术语 | 预设 Preset | `.specify/presets/templates/` |
| 需要核心没有的能力 | 扩展 Extension | `.specify/extensions/templates/` |

**改模板怎么操作**（以让所有 spec 强制带上「异常场景」章节为例）：

```bash
# 1. 建覆盖目录
mkdir -p .specify/templates/overrides

# 2. 复制核心模板作为基底
cp .specify/templates/spec-template.md .specify/templates/overrides/

# 3. 编辑覆盖版，加入你要求的章节
# 4. 下次执行 /speckit.specify 即生效（模板运行时解析，无需重新 init）
```

### 8.2 用扩展增加新能力

```bash
specify extension search      # 搜索可用扩展
specify extension add         # 安装扩展
```

官方内置两个高价值扩展：

**bug 扩展**——`assess → fix → test` 三步，避免 AI 从报告直接跳到补丁：

```bash
specify extension add bug
/speckit.bug-assess "<描述>" slug=login-crash
/speckit.bug-fix slug=login-crash
/speckit.bug-test slug=login-crash
```

**assess 扩展**——把想法变成 go / needs-clarification / kill 的文档化决策：

```bash
specify extension add assess
/speckit.assess-intake "<想法>" slug=offline-mode
/speckit.assess-research slug=offline-mode
/speckit.assess-define slug=offline-mode
/speckit.assess-shape slug=offline-mode
/speckit.assess-decide slug=offline-mode
```

决策为 go 之后，可直接交给 `/speckit.specify` 进入正式开发链路。

### 8.3 用 Bundle 一次性配置角色

Bundle 把扩展、预设、步骤打包成可版本化的角色配置（产品经理、业务分析师、安全研究员、开发）：

```bash
specify bundle search
specify bundle info <name>     # 先看清会装什么再装
specify bundle install <name>
specify bundle list
specify bundle update
specify bundle remove <name>
```

`info` 展示的内容与 `install` 实际安装的内容完全一致，安装是幂等的且限于项目根目录内——可以放心试。

### 8.4 版本与自升级

```bash
specify self check              # 只检查，不改动
specify self upgrade --dry-run  # 预览将执行什么
specify self upgrade            # 就地升级到最新稳定版
specify self upgrade --tag vX.Y.Z   # 锁定特定版本
```

**建议团队锁定 tag**：Speckit 的产出格式由模板决定，模板会随版本演进。锁定版本 = 锁定产出格式 = 团队可复现。

---

## 九、踩坑实录：真实环境遇到的两个问题

### 坑 1：`specify init` 报 GitHub API 403

**现象**：

```
Error fetching release information
GitHub API returned status 403 for
https://api.github.com/repos/github/spec-kit/releases/latest

Rate Limit Information:
  • Rate Limit: 60 requests/hour
  • Remaining: 0
  • Resets at: 2026-09-02 12:18:21 UTC
```

**原因**：init 需要调 GitHub API 查最新 release 来下载模板，而**未认证请求限额 60 次/小时**，共享出口 IP 的环境（CI、公司内网、云开发机）极易被耗尽。

**三种解法，按推荐度排序**：

**① 用 Token（推荐）**——认证后限额提升到 5000 次/小时：

```bash
export GH_TOKEN="github_pat_xxxxxxxxxxxx"    # 或 GITHUB_TOKEN
specify init my-project --ai codebuddy --script sh --github-token "$GH_TOKEN"
```

Token 只需最小权限：Fine-grained 勾选 `Public Repositories → Read-only`，或 Classic token 不勾选任何 scope（默认即可读公开仓库）。**用环境变量，别把明文写进命令行历史**。

**② 等限流重置**——错误信息里的 `Resets at` 就是恢复时间：

```bash
date -u    # 对比当前 UTC 时间
```

**③ 手动下载模板**——绕过 API，缺点是跳过工具的完整性校验。

**排查建议**：先跑 `specify init --help` 确认它到底接受哪些参数，比猜快得多。

### 坑 2：Next.js 16 拦截预览域名的 dev 资源

**背景**：一条命令创建 Next.js + TailwindCSS + TypeScript 项目（实际版本 Next.js 16.3.4 / React 19.2.8 / TailwindCSS v4）：

```bash
npx --yes create-next-app@latest vibe-lab \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --use-npm --yes
```

构建验证：

```
✓ Compiled successfully in 4.9s
✓ Finished TypeScript in 2.1s
✓ Generating static pages using 5 workers (4/4) in 582ms
```

**问题**：通过云开发机预览域名打开时，dev 日志出现：

```
⚠ Blocked cross-origin request to Next.js dev resource /_next/hmr
  from "xxxx--3000.ap-shanghai2.cloudstudio.club".
```

**原因**：Next.js 16 默认阻止跨源访问 dev-only 资源。`localhost` 访问正常，外部预览域名会被拦，表现为**热更新失效**。

**解法**：在 `next.config.ts` 配置 `allowedDevOrigins`，支持通配符子域名：

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // 允许预览域名在 dev 模式下访问 HMR 等资源
  allowedDevOrigins: ["*.cloudstudio.club", "*.cloudstudio.work"],
};

export default nextConfig;
```

两个注意点：该配置**仅开发模式生效**；改完**必须重启 dev server**。

**附带的小坑**：这类环境里 `curl localhost:3000` 可能因代理返回空响应（`HTTP:000`）。别急着判定服务挂了——看 dev 日志里有没有 `GET / 200` 更可靠，或者绕过代理：

```bash
curl -s --noproxy '*' -o /dev/null -w "%{http_code}\n" http://localhost:3000
```

---

## 十、小结

- Speckit 的核心不是多一套命令，而是**开发范式升级**：先规约，后实现；先一致，后加速。
- 想让产出稳定，靠的是**三样东西进版本库**：`constitution.md`（底线）、`templates`（格式契约）、`specs/`（功能资产）。
- 产出不满意时，**改上游文档而不是改代码**，这样下次重新生成才不会把问题带回来。

### 三条上手建议

1. **先跑小功能快路径**：完整链路较长，可先用 `plan → tasks → implement` 练手，再补 `constitution`、`clarify`、`analyze`。
2. **`clarify` 和 `analyze` 别省**：看似拖慢节奏，实为返工预防器。
3. **团队协同时锁定 CLI 版本**，并用 presets 固化组织级模板标准。

---

## 附：速查命令卡

```bash
# ---------- 安装与初始化 ----------
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
specify init my-project --ai codebuddy --script sh            # 新项目
specify init . --ai codebuddy --script sh --no-git --force    # 存量项目
specify init --help                                           # 参数以实际版本为准

# 遇到 GitHub API 403
export GH_TOKEN="github_pat_xxxxxxxxxxxx"

# ---------- 七步流程 ----------
/speckit.constitution   立规矩        → .specify/memory/constitution.md
/speckit.specify        写需求(WHAT)  → specs/xxx/spec.md
/speckit.clarify        补边界(建议)  → 更新 spec.md
/speckit.plan           定方案(HOW)   → plan.md
/speckit.tasks          拆任务        → tasks.md
/speckit.analyze        查一致(建议)  → 分析报告
/speckit.implement      写代码        → 可运行代码
/speckit.converge       收敛检查      → 追加剩余任务（1.0.0）

# ---------- 定制与扩展 ----------
mkdir -p .specify/templates/overrides   # 项目级模板覆盖（运行时生效）
specify extension add bug               # 缺陷修复：assess → fix → test
specify extension add assess            # 想法评估：intake → research → define → shape → decide
specify bundle info <name>              # 看清再装
specify self check                      # 检查版本

# ---------- 常用补救 ----------
/speckit.implement --continue   # 中断后继续
编辑 specs/xxx/tasks.md         # 调整优先级与顺序

# ---------- Next.js 16 + 预览域名 ----------
# next.config.ts: allowedDevOrigins: ["*.your-preview-domain.com"]
```

---

