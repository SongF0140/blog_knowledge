---
title: "AI 进入 CI/CD 流水线"
published: 2026-09-04
description: "CLI 形态的 AI 如何真正嵌入流水线：AI 进入自动化流程的入口只有一个参数。"
category: AI工程
tags: ["CI/CD", "AI", "工程效率"]
draft: false
---
## 一、先说结论：AI 进入流水线的入口，只有一个参数

大多数人对 CLI 形态的 AI 的印象是"在终端里跟它聊天"——启动、输入、等回答、再追问。这个模式很好用，但它有个致命问题：**它需要人一直坐在那儿**。

而流水线里没有人。

于是就有了非交互模式。给一个指令，AI 执行完直接输出结果，然后退出。不等待、不对话，**像一条普通命令一样被调用**。

```bash
# 交互模式：进入对话界面（适合人机协作）
codebuddy

# 非交互模式：执行完直接退出（适合脚本/CI）
codebuddy -p "用一句话解释什么是闭包"
```

一旦 AI 变成了"一条普通命令"，它就能被放进任何自动化流程：

```
Shell 脚本 → codebuddy -p → 输出结果 → jq 解析 → 后续处理
```

这一步转变的意义非常大。你可以把它理解成：**AI 从"一个聊天窗口"变成了流水线上的一道工序**，和 `npm run lint`、`npx jest`、`docker build` 是同一层级的东西。

本文要做的事情，就是把这个想法落地成两条能跑的流水线：

1. **PR 自动代码审查**：`git diff → AI 审查 → JSON 报告 → PR 评论`
2. **自动测试生成**：`找出未覆盖函数 → AI 生成测试 → 定时补全存量`

---

## 二、核心参数速查表

先备好工具箱，后面所有示例都是这些参数的组合。

| 参数 | 缩写 | 功能 | 典型场景 |
|------|------|------|---------|
| `--print` | `-p` | 打印响应后退出，不进入交互模式 | 所有自动化场景的基础 |
| `--output-format` | — | 指定输出格式：`text`（默认）/ `json` / `stream-json` | 脚本解析结果 |
| `--dangerously-skip-permissions` | `-y` | 跳过所有权限确认提示 | AI 需要写文件/执行命令时 |
| `--max-turns` | — | 限制 Agent 最大操作轮数 | 防止 CI 中无限循环 |
| `--continue` | `-c` | 继续上一次对话 | 多步骤关联任务 |
| `--resume` | `-r` | 恢复指定会话（传 session-id） | 跨脚本步骤保持上下文 |
| `--system-prompt-file` | — | 从文件加载系统提示词 | 统一团队 AI 行为规范 |
| `--append-system-prompt` | — | 在默认 System Prompt 后追加内容 | 临时追加额外约束 |
| `--json-schema` | — | 验证输出是否符合指定 JSON Schema | 结构化输出校验 |
| `--verbose` | — | 输出详细日志（调试用） | 排查 CI 中的异常 |
| `--input-format` | — | 指定输入格式：`text`（默认）/ `stream-json` | 串联多个 AI 步骤 |

真正高频的其实只有三个：**`-p`、`--output-format json`、`-y`**。

### 四种最常用的组合模式

**模式一：管道分析（最基础）**

```bash
# 分析日志
cat error.log | codebuddy -p "统计 ERROR 级别的错误类型，按出现次数降序排列"

# 审查代码变更
git diff HEAD~1 | codebuddy -p "这次改动有没有安全漏洞或明显的逻辑错误？"

# 交叉分析多个文件
cat src/auth.ts src/middleware.ts | codebuddy -p "这两个文件的权限验证逻辑有没有冲突？"
```

**模式二：JSON 输出（脚本集成的前提）**

```bash
git diff HEAD~1 | codebuddy -p \
  "审查这次代码改动，JSON 格式输出，字段：severity（high/medium/low）、issues（数组）、summary（字符串）" \
  --output-format json | jq '.issues[] | select(.severity == "high")'
```

**模式三：让 AI 写文件（必须加 `-y`）**

```bash
cat src/api.ts | codebuddy -p \
  "给这个文件写完整的 Jest 单测，写入 src/__tests__/api.test.ts" \
  -y --max-turns 10
```

> ⚠️ **`-y` 使用原则**：只在受控的 CI 环境或信任的本地脚本中使用。`-y` 会跳过所有权限确认，AI 可能在没有提示的情况下修改文件、执行命令。

**模式四：用规则文件统一 AI 行为**

```bash
git diff | codebuddy -p "审查代码" \
  --system-prompt-file .codebuddy/review-rules.md \
  --append-system-prompt "输出语言必须是中文，每条建议不超过 50 字"
```

---

## 三、本地先跑通：三步验证

在把 AI 塞进 CI 之前，**一定要先在本地跑通**。CI 里的任何问题排查成本都是本地的十倍。

### 第 1 步：拿到 diff

```bash
git log --oneline -5          # 找一个 commit hash
git diff 2943ff39 > /tmp/changes.diff   # 或 git diff > /tmp/changes.diff 审查未提交改动
wc -l /tmp/changes.diff       # 确认非空
```

> 如果输出被 `less` 分页器占用，按 `Q` 退出。

### 第 2 步：跑一次纯文本审查

```bash
cat /tmp/changes.diff | codebuddy -p \
  "审查这段代码变更，找出潜在问题，每条问题说明严重程度（高/中/低）和改进建议"
```

先要文字版，肉眼确认 AI 的理解是靠谱的——**这一步是在校准你的 prompt**。

### 第 3 步：切 JSON，用 jq 提取

```bash
cat /tmp/changes.diff | codebuddy -p \
  "审查代码变更，输出纯 JSON 格式，包含 issues 数组，每个 issue 有 severity/file/line/description/suggestion 字段" \
  --output-format json > review.json

# 只看高严重度问题
cat review.json | jq -r '.[-1].result' | tail -n +2 | head -n -1 > /tmp/issues.json
grep -A 4 '"severity": "high"' /tmp/issues.json
```

> 💡 **坑点**：`--output-format json` 的输出外层有包装，真正的 AI 输出在 `result` 字段里。在 CI 里更稳的做法是用 Python 提取：
> ```bash
> cat raw_result.json | python3 -c "import json,sys; print(json.load(sys.stdin).get('result','{}'))" > review_result.json
> ```

本地跑通之后，后面的事情就只是"把这三步搬进 Actions"。

---

## 四、规则文件：让 AI 审查稳定可控

光靠一句"帮我审查代码"不够。**没有规则，每次结果都不一样；有了规则，标准才稳定、可控、不随 Reviewer 变化**。

```
无规则：prompt "审查代码" → 每次结果不一样
有规则：prompt + 规则文件 → 稳定、可控
```

在项目根目录建 `.codebuddy/review-rules.md`：

```markdown
# 代码审查规则

## 必须检查的安全项
- 所有 SQL 查询必须使用参数化查询，禁止字符串拼接
- 用户输入在使用前必须经过校验（不能直接传入系统命令）
- 不允许在代码或日志中出现明文密钥、Token、密码
- HTTP 接口必须有认证校验（除白名单接口外）
- 禁止使用不安全的加密算法

## 代码规范
- TypeScript 文件不允许使用 `any` 类型（特殊情况需加注释说明）
- 所有 export 函数必须有 JSDoc 注释
- 错误处理不允许使用空 catch（catch(e) {}）
- 禁止硬编码 Magic Number
- 大文件（>300行）需要拆分

## 性能检查
- 数据库查询必须建索引
- 禁止在循环内执行数据库查询
- 大数据量操作需要分页

## 输出格式
- 输出语言：中文
- 每条问题包含：严重程度（高/中/低）、文件位置、描述、改进建议
- 高严重度问题必须修复后才能合并
```

三种使用方式：

```bash
# 方式一：直接把规则塞进 prompt（最简单，兼容性最好）
cat /tmp/changes.diff | codebuddy -p "$(cat .codebuddy/review-rules.md) 严格按照上述规则审查" --output-format text

# 方式二：作为系统提示词加载
cat /tmp/changes.diff | codebuddy -p "审查代码" --system-prompt-file .codebuddy/review-rules.md

# 方式三：规则 + 临时追加约束
cat /tmp/changes.diff | codebuddy -p "审查代码" \
  --system-prompt-file .codebuddy/review-rules.md \
  --append-system-prompt "每条建议不超过 50 字，输出 JSON 格式"
```

**建议**：规则文件纳入版本控制，和代码一起走 Review。改规则的流程和改代码一样——`提出规则 → Review → 合并 → 自动生效`。多项目可以各自维护 `.codebuddy/review-rules.md`，团队通用规则单独抽一个仓库。

---

## 五、接入 GitHub Actions：PR 自动审查

### 5.1 配 Secret（必须先做）

1. 拿 API Key
   - 海外版：https://www.codebuddy.ai/profile/keys
   - 中国版：https://copilot.tencent.com/profile
2. 仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
3. Name 填 `CODEBUDDY_API_KEY`，Value 粘贴 Key

> ⚠️ Name 必须是 `CODEBUDDY_API_KEY`，这是 CLI 认死的名字，不能改。

**用服务账号 Token，不要用个人 Token。** 个人 Token 一旦持有人离职 CI 就挂了，服务账号走团队管理、权限最小化。

### 5.2 工作流文件

创建 `.github/workflows/ai-review.yml`：

```yaml
name: AI Code Review

on:
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  ai-review:
    runs-on: ubuntu-latest

    permissions:
      pull-requests: write
      contents: read

    steps:
      # 1. 检出代码（fetch-depth: 0 才能拿到完整历史做 diff）
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # 2. 获取 PR 的变更内容
      - name: Get PR diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > pr_diff.txt

      # 3. 设置 Node.js 环境
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      # 4. 安装 CodeBuddy CLI（锁定版本，别用 latest）
      - name: Install CodeBuddy CLI
        run: npm install -g @tencent-ai/codebuddy-code@1.7.0

      # 5. 执行 AI 代码审查
      - name: AI Code Review
        env:
          CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}
        run: |
          if [ -z "$CODEBUDDY_API_KEY" ]; then
            echo "Error: CODEBUDDY_API_KEY is not set"
            exit 1
          fi

          cat pr_diff.txt | codebuddy -p \
            "审查代码变更，输出 JSON 格式：{issues: [{severity, file, line, description, suggestion}]}" \
            --output-format json > raw_result.json || true

          # 提取干净的 result 字段
          if [ -s raw_result.json ]; then
            cat raw_result.json | python3 -c "import json,sys; print(json.load(sys.stdin).get('result','{}'))" > review_result.json
          else
            echo '{}' > review_result.json
          fi

      # 6. 发布 PR 评论
      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const result = JSON.parse(fs.readFileSync('review_result.json', 'utf8') || '{}');

            let comment = '## 🤖 AI 代码审查结果\n\n';

            if (result.issues && result.issues.length > 0) {
              const high = result.issues.filter(i => i.severity === 'high' || i.severity === '高');
              const medium = result.issues.filter(i => i.severity === 'medium' || i.severity === '中');

              comment += `发现 **${result.issues.length}** 个问题：\n`;
              comment += `- 🔴 高严重度: ${high.length} 个\n`;
              comment += `- 🟡 中严重度: ${medium.length} 个\n\n`;

              result.issues.forEach((issue, index) => {
                const label = issue.severity === 'high' || issue.severity === '高' ? '🔴' : '🟡';
                comment += `### ${label} 问题 ${index + 1}\n`;
                comment += `- **文件**: \`${issue.file || '未知'}\`\n`;
                comment += `- **位置**: 第 ${issue.line || '?'} 行\n`;
                comment += `- **描述**: ${issue.description || ''}\n`;
                comment += `- **建议**: ${issue.suggestion || ''}\n\n`;
              });
            } else {
              comment += '✅ 代码审查通过，未发现问题。';
            }

            github.rest.issues.createComment({
              issue_number: context.payload.pull_request.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

几个容易被忽略的细节：

- `fetch-depth: 0`——默认 checkout 只拉一个 commit，`git diff origin/main...HEAD` 会失败。
- CLI **锁定版本号**——别用 `latest`，否则上游发版可能一夜之间搞挂你的流水线。
- `|| true`——AI 步骤失败不应该直接把 PR 判死（后面会讲为什么）。

### 5.3 触发验证

```bash
git checkout -b feat/test-ai-review
git add .github/workflows/ai-review.yml
git commit -m "feat: add AI code review workflow"
git push -u origin feat/test-ai-review
```

然后在 GitHub 上开 PR（`base: main` ← `compare: feat/test-ai-review`），等 2~3 分钟，PR 页面下方就会出现 AI 审查评论。

---

## 六、AI 测试生成：别指望它替代人，指望它补缺口

先摆正预期：

```
AI 生成测试 = 快速生成测试骨架 + 覆盖基础场景

最终还是要人工确认：
- 边界条件合不合理
- 业务逻辑对不对
- 是否需要补充特殊用例
```

它真正解决的是两个高频痛点：**新代码赶进度没写测试**、**存量代码压根没测试且补测代价太高**。

### 6.1 本地生成

```bash
cat src/validators.ts | codebuddy -p \
  "给这个文件里所有 export 函数写完整的 Jest 单测。

  要求：
  1. 覆盖正常输入、边界条件、异常场景三类用例
  2. 使用 describe/it 结构，测试名称用中文
  3. 只输出代码，不要任何解释文字" \
  --output-format text > src/__tests__/validators.test.ts
```

### 6.2 让 AI 反过来诊断失败日志

这个用法比"生成测试"更实用：

```bash
npx jest --no-coverage 2>&1 | tail -100 | codebuddy -p \
  "分析这些测试失败日志，指出根本原因，给出修复建议"

# 更狠一点：约束它不许偷改测试
npx jest --no-coverage 2>&1 | codebuddy -p \
  "分析失败原因，优先修业务代码，不要改测试，给出修复步骤"
```

### 6.3 CI 场景一：PR 覆盖率守门

```yaml
name: Test Coverage Guard

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  coverage-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci

      - name: Run Tests
        run: npx jest --coverage --coverage-reporters=json-summary

      - name: Check Missing Tests
        env:
          CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}
        run: |
          npm install -g @tencent-ai/codebuddy-code

          git diff origin/${{ github.base_ref }}...HEAD -- '*.ts' '*.js' \
            '!**/*.test.*' '!**/*.spec.*' > new_code.txt

          if [ ! -s new_code.txt ]; then
            echo "No new source files"
            exit 0
          fi

          cat new_code.txt | codebuddy -p \
            "分析新增代码，判断哪些 export 函数没有测试。
            JSON 格式输出：
            {
              \"missing_tests\": [{\"file\": \"\", \"function\": \"\", \"reason\": \"\"}],
              \"coverage_ok\": true|false
            }" \
            --output-format json > missing_tests.json
```

### 6.4 CI 场景二：每周定时补全存量测试

```yaml
name: Backfill Tests (Weekly)

on:
  schedule:
    - cron: '0 2 * * 1'   # 每周一凌晨 2 点

jobs:
  backfill:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci && npm install -g @tencent-ai/codebuddy-code

      - name: Find Low Coverage Files
        run: |
          npx jest --coverage --coverage-reporters=json 2>/dev/null || true
          node -e "
          const cov = require('./coverage/coverage-summary.json');
          Object.entries(cov)
            .filter(([f, d]) => f !== 'total' && d.lines.pct < 50)
            .slice(0, 5)
            .forEach(([f]) => console.log(f));
          " > low_coverage_files.txt

      - name: Generate Tests
        env:
          CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}
        run: |
          while IFS= read -r file; do
            echo "正在为 $file 生成测试..."
            cat "$file" | codebuddy -p \
              "为这个文件生成完整的 Jest 单测，覆盖所有 export 函数" \
              --output-format text \
              --max-turns 10 > "${file%.ts}.test.ts" || true
          done < low_coverage_files.txt

      - name: Create PR
        uses: peter-evans/create-pull-request@v6
        with:
          commit-message: "test: AI 自动补全测试"
          title: "chore: AI 自动生成测试覆盖"
          body: "本 PR 由定时任务自动生成"
          branch: "auto/test-backfill"
          labels: "automated, tests"
```

**注意最后的动作是"创建 PR"，不是"直接合入"。** AI 生成的测试必须由人过一遍再合并。

---

## 七、安全合规：这是底线，不是建议

把 AI 接入 CI，意味着**代码内容会发送到 AI 服务处理**：

| 场景 | 发送内容 | 是否发送到云端 |
|------|---------|------------|
| `git diff \| codebuddy -p "..."` | diff 文本内容 | 是 |
| `cat src/api.ts \| codebuddy -p "..."` | 文件完整内容 | 是 |

所以以下几类内容**绝对不能发送**：

- 密钥、Token、密码（**即使是测试环境的**）
- 用户个人信息（姓名、手机号、身份证号、邮箱）
- 内部 IP 地址、数据库连接字符串
- 合同、财务数据等涉密商业信息

### 7.1 diff 必须过滤敏感文件

```bash
git diff origin/main...HEAD \
  -- '*.ts' '*.js' '*.py' '*.go' \
  ':!**/.env*' \
  ':!**/secrets*' \
  ':!**/config/production*' \
  ':!**/*_key*' \
  ':!**/*_secret*' \
  | codebuddy -p "审查代码"
```

用**白名单 + 黑名单双保险**：先限定只看源码后缀，再排除敏感路径。

### 7.2 API Key 管理

```yaml
# ✅ 正确：从 Secret 读取
env:
  CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}

# ❌ 错误：明文硬编码
env:
  CODEBUDDY_API_KEY: "sk-xxxxx"
```

再加一道 CI 自检，防 Key 混进代码：

```bash
if git diff origin/main...HEAD | grep -E "CODEBUDDY_API_KEY\s*=\s*['\"]?[a-zA-Z0-9_-]{20,}"; then
  echo "错误：检测到明文 API Key"
  exit 1
fi
```

其他平台同样处理：GitLab CI 用 Variables，Jenkins 用 `withCredentials`。

### 7.3 `-y` 的使用红线

| 场景 | 是否使用 `-y` | 理由 |
|------|------------|------|
| 只读分析（审查、诊断） | **不要用** | 只读不需要 |
| 生成文件到指定目录 | 可以用，**限定目录** | 明确写入范围 |
| 执行任意 Bash 命令 | **谨慎，必须审查 prompt** | 高风险 |
| 修改生产配置 | **禁止** | 不可逆 |

```bash
# ✅ 在指定目录生成
codebuddy -p "在 src/__tests__ 生成单测" -y

# ✅ 显式限定范围
codebuddy -p "只修改 src/__tests__ 目录，不要动其他文件" -y

# ❌ 没有任何约束
codebuddy -p "帮我写代码" -y
```

### 7.4 合规检查清单

上线前逐项确认：

- [ ] API Key 通过 Secret 管理，未硬编码
- [ ] diff 过滤规则已排除 `.env*`、密钥文件
- [ ] `-y` 参数只用于写入指定受控目录
- [ ] CI 日志不会打印 API Key 或敏感代码
- [ ] 已确认行业合规要求
- [ ] 团队已知晓 AI 审查的能力边界

> 金融、医疗、政务这类有数据不出境/网络隔离要求的行业，可以考虑私有化部署：把 `CODEBUDDY_API_BASE_URL` 指向企业内网 AI 网关，CLI 用法完全不变。

---

## 八、进阶实践

### 8.1 AI 结果只做建议，不要直接 Fail PR

**正确做法**：AI 审查结果作为 Review Comment。低/中严重度只评论不阻塞，高严重度可标 `request changes`。

**错误做法**：AI 发现问题就 Fail PR——AI 会误报，误报直接卡住合并，团队三天就会想办法绕过这道门禁。

质量门禁能提高稳定性，但**不能取消人工判断和回滚预案**。

### 8.2 独立步骤并行跑

```yaml
jobs:
  review:      # 可以并行
  test-gen:    # 可以并行
  changelog:   # 可以并行
```

判断标准很简单：**有依赖关系的串行，互相独立的并行**。

### 8.3 缓存 AI 结果，削峰

```yaml
- name: Check cache
  id: cache-review
  uses: actions/cache@v4
  with:
    path: .review-cache
    key: review-${{ github.sha }}-${{ hashFiles('**/*.ts') }}

- name: AI Review
  if: steps.cache-review.outputs.cache-hit != 'true'
  run: |
    # 执行审查...
```

也可以按分支做优先级：`hotfix/*` 走高优队列，普通 PR 走常规队列。

### 8.4 完整的 AI 增强型流水线

```yaml
name: AI-Assisted CI/CD

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  # 1. 基础检查（并行）
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  # 2. AI 步骤（并行）
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm install -g @tencent-ai/codebuddy-code
      - name: AI Review
        env:
          CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > diff.txt
          cat diff.txt | codebuddy -p "审查代码" --output-format json > review.json

  ai-test-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm install -g @tencent-ai/codebuddy-code
      - name: Check Test Coverage
        env:
          CODEBUDDY_API_KEY: ${{ secrets.CODEBUDDY_API_KEY }}
        run: |
          # 检查新增代码是否有测试...

  # 3. 汇总
  summary:
    needs: [lint, test, ai-review, ai-test-check]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - run: echo "CI 检查完成"
```

除此之外，AI 在流水线里还能干这些：

- **自动生成 Changelog**：`git log --oneline main..HEAD | codebuddy -p "生成结构化 Changelog"`（发布前人工确认）
- **构建失败根因分析**：`cat build.log | codebuddy -p "分析构建失败日志，给出修复步骤"`

### 8.5 生产环境检查表

| 项目 | 建议 |
|------|------|
| CLI 版本 | 锁定具体版本号，不要用 latest |
| API Key | 通过 Secret 管理，定期轮换 |
| CI 日志 | 不打印 Key，不打印敏感代码 |
| AI 结果 | 作为建议参考，不直接 Fail PR |
| 回滚预案 | 始终保留人工回滚能力 |

---

## 九、常见错误排查

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `codebuddy: command not found` | npm 全局安装路径未加入 PATH | `npm config get prefix` 查路径，把 `/bin` 加入 `$PATH` |
| 操作被权限检查拦截，什么都没执行 | AI 需要写文件但缺少 `-y` | 加上 `-y` 或 `--dangerously-skip-permissions` |
| JSON 输出解析失败 | AI 输出里混入了非 JSON 文字 | 在 prompt 里强调"只输出 JSON，不要任何解释文字" |
| CI 中登录态失效 | `CODEBUDDY_API_KEY` 未设置或过期 | 在 CI 的 Secret 里设置该环境变量 |
| 响应被截断 | `--max-turns` 太小 | 根据任务复杂度调大，或拆分为多个步骤 |
| `git diff origin/main...HEAD` 为空 | checkout 没拉全历史 | `actions/checkout@v4` 加 `fetch-depth: 0` |

---

## 十、总结

回到最开始那句话：

> **CLI 的管道模式，让 AI 成为流水线上的一道工序。**

```bash
git diff | codebuddy -p "审查" | jq '.issues'
```

三个命令，AI 就进了 CI。

**三个关键参数**

| 参数 | 作用 |
|------|------|
| `-p` | 非交互模式，执行完就退出 |
| `-y` | 跳过权限确认（写文件时必须加） |
| `--output-format json` | 结构化输出，让脚本能解析 |

**两个核心场景**

- PR 自动审查：`git diff → AI 审查 → JSON 报告 → PR 评论`
- 自动测试生成：`找出未覆盖函数 → AI 生成测试 → 定时补全存量`

**三条安全底线**

| 规则 | 说明 |
|------|------|
| Secret 管 Key | API Key 必须通过 Secret 管理，用服务账号 |
| diff 过滤 | 审查范围排除 `.env*`、密钥文件 |
| `-y` 受控 | 只在写文件时使用，禁止修改生产配置 |

最后提醒一句 **AI 的能力边界**：

```
AI 能做的：
  ✅ 语法错误、潜在 bug
  ✅ 安全漏洞（SQL 注入、XSS 等）
  ✅ 代码规范问题
  ✅ 测试覆盖率分析

AI 做不了的：
  ❌ 业务逻辑正确性
  ❌ 架构设计合理性
  ❌ 性能优化方向
  ❓ 最终合不合并 —— 人说了算
```

学会了，不等于会用；会用了，才是真正掌握了。挑一个你自己的项目，从 `git diff HEAD~1 | codebuddy -p "审查这段代码"` 开始，跑一遍。

---

### 动手练习

1. **本地代码审查**：用 `git diff HEAD~1 | codebuddy -p "..."` 审一次真实提交，记录"原始改动 → AI 审查结果 → 你的判断"，看看有哪些你认同、哪些是误报。
2. **GitHub Actions 工作流**：在自己的仓库里配好 `CODEBUDDY_API_KEY`，提一个小 PR 触发审查。
3. **测试生成**：找一个没有测试的工具函数文件，让 AI 生成单测并跑通；失败了就把错误日志回传给它要修复建议。
4. **安全合规自查**：确认 Key 走 Secret、diff 过滤了 `.env*`、CI 日志不打印敏感信息。
