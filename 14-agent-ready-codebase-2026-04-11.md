# Agent准备好了吗：代码库8维评估框架

> Agent跑不好，别急着换模型——先查你的代码库。8个技术支柱、5个成熟度等级、60+条二元标准，告诉你代码库离"AI能自主开发"还差什么。像餐厅卫生检查——厨房干净，任何厨师都出活快。

---

## 为什么需要Agent Readiness

### 问题：Agent不坏，是环境坏了

团队部署AI编码Agent，效果忽好忽坏。换模型、换Agent，还是一样。**真正的问题通常是代码库本身。**

三种典型失败：

| 失败 | 原因 | Agent遭遇 |
|------|------|----------|
| 没有pre-commit hooks | 格式问题要等CI跑10分钟才发现 | Agent等10分钟才看到格式错误 |
| 环境变量没文档 | 只有Slack聊天记录里有 | Agent猜一个、错一个、再猜一个 |
| 构建命令靠口口相传 | "大家都知道"怎么跑 | Agent完全不知道怎么验证自己的代码 |

```
反馈循环对比：

好的代码库：
  Agent改代码 → pre-commit 5秒报错 → 立即修 → 再提交 → 通过 ✅

差的代码库：
  Agent改代码 → 提交 → 等CI 10分钟 → 失败 → 盲猜修复 → 再等10分钟 ❌
  → 3轮循环后token预算耗尽，任务失败
```

**核心洞察：** 一个反馈循环差的代码库会打败任何Agent。一个反馈快、指令清晰的代码库会让任何Agent都高效。

这不是某个Agent的问题——**是环境问题，而环境问题会复合。**

---

## 整体架构

```
┌─────────────────────────────────────────────┐
│              8 个技术支柱                      │
├────────┬────────┬────────┬────────┬─────────┤
│ 风格   │ 构建   │ 测试   │ 文档   │ 开发环境 │
│ 验证   │ 系统   │        │        │         │
├────────┼────────┼────────┴────────┴─────────┤
│ 代码   │ 可观测 │ 安全                        │
│ 质量   │ 性     │ 治理                        │
└────────┴────────┴────────────────────────────┘
         │
         │ 每个支柱一组二元标准（通过/失败）
         │ 60+ 条标准
         ▼
┌─────────────────────────────────────────────┐
│              5 个成熟度等级                    │
├─────────────────────────────────────────────┤
│ L1 功能性    │ 能跑，手动配置                  │
│ L2 有文档    │ 工作流写下来了                  │
│ L3 标准化 ← │ 生产级Agent自主操作（目标）       │
│ L4 优化     │ 亚分钟级反馈                    │
│ L5 自治     │ 自愈、自改进                    │
└─────────────────────────────────────────────┘
```

**评估规则：**
- 每条标准**二元判定**：通过或失败（文件存在？配置解析通过？）
- 通过当前等级**80%**的标准 + 所有前置等级标准 → 解锁下一等级
- **门控递进**：不能跳级，必须从地基开始

---

## 8个支柱详解

### 支柱 1：风格与验证 —— Linter、格式化、类型检查

自动捕获bug的工具。没有它们，Agent在语法错误和风格漂移上浪费时间。

```
❌ 没有Linter：
  Agent提交代码 → CI报格式错误 → 盲修 → 再报错 → 循环

✅ 有Linter + pre-commit：
  Agent提交代码 → pre-commit 2秒报错 → 自动修 → 通过
```

| 标准 | 说明 | 为什么要Agent在意 |
|------|------|------------------|
| formatter配置 | Prettier/Black/Ruff format | 不用为缩进和空格浪费token |
| lint_config | ESLint/Ruff/Pylint有规则 | 即时捕获常见错误 |
| type_check | TypeScript strict / mypy | 类型错误不用跑测试就知道 |
| pre_commit_hooks | Husky / .pre-commit-config.yaml | 提交前拦截问题，不等CI |

真实数据：Express（Level 2）的Style & Validation只有**10%**通过率——没有Prettier、没有pre-commit hooks、没有类型检查。

### 支柱 2：构建系统 —— 清晰、可文档化的构建命令

确定性的构建命令让Agent能验证改动。不用猜。

| 标准 | 说明 | 为什么要Agent在意 |
|------|------|------------------|
| build_cmd_doc | README里有构建命令 | Agent知道跑什么命令 |
| single_command_setup | 一条命令搞定环境 | `npm install` 或 `uv sync` |
| release_automation | semantic-release / changesets | 不用手动发版 |
| fast_ci_feedback | CI在10分钟内完成 | 快反馈 = 快迭代 |

真实数据：CockroachDB（Level 4）的Build System **92%**通过率——`CLAUDE.md`里写清楚了所有构建命令，`./dev doctor`一条命令验证环境。

### 支柱 3：测试 —— 快速单元测试和集成测试

快速测试创建紧凑的反馈循环。Agent通过跑测试、看失败、迭代来学习。

| 标准 | 说明 | 关键阈值 |
|------|------|---------|
| unit_tests_exist | 有单元测试 | — |
| unit_tests_runnable | 测试能跑 | `pytest --collect-only`能收集 |
| test_coverage_thresholds | 覆盖率门槛 | FastAPI要求**100%** |
| integration_tests_exist | 有集成测试 | — |

⚠️ 测试速度是关键：单元测试**< 30秒**，集成测试**< 5分钟**。慢测试会打断Agent的迭代循环。

### 支柱 4：文档 —— AGENTS.md、README、贡献指南

把"大家都知道"的隐性知识写出来。Agent无法从走廊聊天和Slack历史中学习。

| 标准 | 说明 | 最重要的一条 |
|------|------|------------|
| AGENTS.md / CLAUDE.md | 给Agent的专门指令 | ✅ **这是最关键的文件** |
| readme | README.md存在且完整 | 入口文档 |
| documentation_freshness | 文档在180天内更新过 | 过期文档比没文档更糟 |
| skills | `.claude/skills/` 或 `.factory/skills/` | 可复用的Agent能力 |

**AGENTS.md / CLAUDE.md 是Agent最重要的文件。** 它告诉Agent：怎么构建、怎么测试、项目的约定是什么。

CockroachDB的`CLAUDE.md`有**886行**——详细记录了所有命令、架构、工作流。这是Level 4项目的重要标志。

### 支柱 5：开发环境 —— 可复现的环境

当开发者和Agent在相同环境里工作，"在我机器上能跑"的问题就消失了。

| 标准 | 说明 |
|------|------|
| devcontainer | `.devcontainer/devcontainer.json` |
| env_template | `.env.example` 存在 |

### 支柱 6：代码质量 —— 模块化、合理文件大小

模块化代码帮Agent理解系统。几千行的文件超出上下文窗口和认知负荷。

| 标准 | 说明 | 经验阈值 |
|------|------|---------|
| 文件大小 | 每个文件 | **< 500行** |
| 函数大小 | 每个函数 | **< 50行** |
| 模块边界 | 清晰的模块划分 | — |

### 支柱 7：可观测性 —— 结构化日志、追踪、指标

好的可观测性把"出错了"变成"出错是因为调Y时X是null"。Agent需要运行时可见性来诊断问题。

```
❌ 没有可观测性：
  Agent看到 "Error: something went wrong"
  → 不知道哪里出了问题，无法修复

✅ 有结构化日志 + 追踪：
  Agent看到 "Error: null value in user.email
    at UserService.create (user.go:142)
    trace_id: abc123 request_id: def456"
  → 精确定位，直接修复
```

### 支柱 8：安全与治理 —— 分支保护、密钥扫描、CODEOWNERS

安全控制防止Agent引入漏洞、泄露密钥或绕过必要的审查。没有这些，自主Agent就是**风险源头**。

| 标准 | 说明 |
|------|------|
| branch_protection | 分支保护规则 |
| secret_scanning | 密钥扫描 |
| CODEOWNERS | 代码所有权文件 |

---

## 5个成熟度等级

```
L1 ──── L2 ──── L3 ──── L4 ──── L5
功能性   有文档   标准化   优化    自治
                 ↑
              目标等级
```

### Level 1：功能性 —— 代码能跑，需要手动配

**关键信号：** README存在、Linter配置了、类型检查开着、有单元测试

**Agent能力：** 勉强完成简单任务。失败率高，需要频繁人工介入。

**例子：** 个人项目、早期原型

### Level 2：有文档 —— 工作流写下来了

**关键信号：** AGENTS.md存在、可复现开发环境、pre-commit hooks、分支保护

**Agent能力：** 在监督下做简单任务。修bug、小功能。

**例子：** Flask、Express、Zod

### Level 3：标准化 —— Agent可以自主操作 ✅ 目标

**关键信号：** E2E测试、文档持续维护、安全扫描、可观测性

**Agent能力：** 常规维护——修bug、写测试、写文档、升级依赖。

**例子：** FastAPI、GitHub CLI、pytest、Zed

> **Level 3是目标。大多数团队应该先到这里。**

### Level 4：优化 —— 亚分钟级反馈

**关键信号：** 亚分钟验证、完整可观测性、金丝雀部署、构建优化

**Agent能力：** 复杂多步任务。新功能、重构、迁移。

**例子：** CockroachDB、Temporal

### Level 5：自治 —— 自改进系统

**关键信号：** 任务分解、多服务编排、自愈、自动修复

**Agent能力：** 组合管理。人类定方向，Agent执行。

**例子：** 目前对大多数组织来说是理想状态

---

## 真实项目对比

三个知名开源项目，三种等级：

```
           Express    FastAPI    CockroachDB
           (L2)       (L3)       (L4)
风格验证    10%  ■     64%  ████  50%  ███
构建系统    33%  ██    64%  ████  92%  ██████
测试        63%  ████  71%  █████ 100% ███████
文档        29%  ███   57%  ████  86%  ██████
开发环境     0%        0%         50%  ███
可观测性    25%  ██    25%  ██    82%  ██████
安全        25%  ██    43%  ███   67%  ████
任务发现    25%  ██    75%  █████ 75%  █████
───────────────────────────────────────────
总通过率    28%        53%        74%
```

**关键差距在哪：**

| 维度 | Express缺什么 | FastAPI缺什么 | CockroachDB为什么强 |
|------|-------------|-------------|-------------------|
| 文档 | 没AGENTS.md | 没AGENTS.md | CLAUDE.md有886行 |
| 构建 | 没锁文件、没自动发布 | 没feature flag | Bazel + 完整CI |
| 可观测性 | 只有基础日志 | 没Sentry、没追踪 | Sentry + OpenTelemetry + Prometheus |
| 安全 | 没CODEOWNERS | 没CODEOWNERS | CODEOWNERS + 分支保护 + 日志脱敏 |

**一个有趣的发现：** Express和CockroachDB都是成功的、广泛使用的项目。但Agent在CockroachDB上的效率会高得多。**项目质量 ≠ Agent就绪度。**

---

## 三档递进

### 入门：从Level 0到Level 2（5个文件，30分钟）

最低成本、最高回报的改进：

```bash
# 1. 创建 AGENTS.md（最关键）
cat > AGENTS.md << 'EOF'
# Agent 指南

## 构建
npm run build

## 测试
npm test

## 代码风格
npm run lint

## 常见陷阱
- 环境变量见 .env.example
- 测试必须跑通才能提交
EOF

# 2. 添加 .env.example
cp .env .env.example    # 脱敏后提交
echo ".env" >> .gitignore

# 3. 配置 pre-commit hooks
npm install husky --save-dev
npx husky init
echo "npm run lint" > .husky/pre-commit

# 4. 添加 PR 模板
mkdir -p .github
cat > .github/pull_request_template.md << 'EOF'
## 改了什么
## 怎么测试
## 截图（如适用）
EOF

# 5. 添加 Issue 模板
mkdir -p .github/ISSUE_TEMPLATE
cat > .github/ISSUE_TEMPLATE/bug_report.md << 'EOF'
## Bug描述
## 复现步骤
## 期望行为
EOF
```

这5步就能从Level 0到Level 2。

### 进阶：从Level 2到Level 3（日常80%够用）

Level 2 → Level 3 的关键差距：

| 差距 | 怎么补 | 工作量 |
|------|--------|-------|
| E2E测试 | 加Playwright/Cypress配置 | 1-2天 |
| 安全扫描 | 加Dependabot + CodeQL | 30分钟 |
| 可观测性 | 加Sentry + 结构化日志 | 半天 |
| CODEOWNERS | 按模块分配代码所有者 | 1小时 |
| Issue标签 | 建立标签体系 | 1小时 |

```yaml
# .github/dependabot.yml — 自动依赖更新
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### 高级：从Level 3到Level 4+

- **亚分钟验证**：把CI拆成fast path和slow path，pre-commit跑fast path
- **构建缓存**：Bazel / Turborepo / Nx 分布式缓存
- **金丝雀部署**：Argo Rollouts / Flagger
- **Agent协作痕迹**：Co-authored-by in commits，让Git历史记录Agent参与
- **Skills目录**：`.claude/skills/` 里放可复用的Agent技能

---

## 常见错误

```
❌ 错误 1：换Agent，不改善环境

"Cursor不行，换Copilot"
"Claude不行，换GPT"
→ 根本问题在代码库，换什么Agent都一样

✅ 正确：先跑readiness report，修最严重的缺口

三个项目数据证明：
同一套Agent，在不同代码库上的表现天差地别
```

```
❌ 错误 2：追求高等级，跳过基础

直接上分布式追踪和金丝雀部署
→ 但连AGENTS.md都没有

✅ 正确：门控递进，从L1开始

先确保README、Linter、测试跑通
再加AGENTS.md、pre-commit hooks
再考虑可观测性和安全扫描
```

```
❌ 错误 3：平均分焦虑

"我们的平均分是73.2%"
→ 这个数字不能指导行动

✅ 正确：看"活跃仓库达到Level 3+的比例"

"80%的活跃仓库达到Agent就绪"
→ 这是一个可追踪、可行动的目标
```

---

## 速查表

### 8支柱速查

| 支柱 | 核心标准 | 没有会怎样 |
|------|---------|-----------|
| 风格与验证 | Linter + formatter + pre-commit | Agent在格式错误上浪费循环 |
| 构建系统 | 构建命令文档化 + 一键设置 | Agent猜命令、猜失败 |
| 测试 | 单元测试<30s + 集成测试<5min | Agent无法验证改动 |
| 文档 | **AGENTS.md** + README | Agent靠猜，猜错就错 |
| 开发环境 | devcontainer + .env.example | 环境不一致导致诡异失败 |
| 代码质量 | 文件<500行 + 函数<50行 | 超出上下文窗口 |
| 可观测性 | 结构化日志 + 错误追踪 | Agent看到"error"就傻了 |
| 安全治理 | 分支保护 + CODEOWNERS | Agent变成风险源 |

### 5等级速查

| 等级 | 关键信号 | Agent能做什么 |
|------|---------|-------------|
| L1 功能性 | README + Linter + 测试 | 几乎不能自主工作 |
| L2 有文档 | AGENTS.md + pre-commit + 分支保护 | 简单任务（需监督） |
| **L3 标准化** | E2E测试 + 安全扫描 + 可观测性 | **常规维护（自主）** |
| L4 优化 | 亚分钟验证 + 完整可观测性 | 复杂任务、重构 |
| L5 自治 | 自愈 + 自动修复 | 人类定方向，Agent执行 |

### 关键数字

| 指标 | 值 | 说明 |
|------|-----|------|
| 评估标准数 | 60+条 | 全部二元判定 |
| 解锁门槛 | 80% | 当前级 + 所有前置级 |
| 评估方差 | 0.6% | 基于"前次报告锚定"技术 |
| 修复前方差 | 7% | 锚定后降至0.6% |
| 活跃仓库定义 | 90天内有提交 | 只评估活跃仓库 |
| L3目标 | Level 3 | 大多数团队的目标 |

---

## 参考资料

- [Introducing Agent Readiness — Factory.ai](https://factory.ai/news/agent-readiness) — 本文框架来源
- [Agent Readiness 开源项目报告](https://factory.ai/agent-readiness) — 30+个真实项目的评估结果
- [Factory.ai 文档](https://docs.factory.ai/web/agent-readiness/overview) — 详细评分标准
- [Subagent 上下文隔离](04-subagent-context-isolation-2026-04-06.md) — 为什么干净上下文重要
- [三层上下文压缩](06-three-layer-context-compression-2026-04-06.md) — 上下文管理的基础

---

## 系列导航

1. [Agent Loop 基础](01-agent-loop-fundamentals-2026-04-06.md) →
2. [工具分发设计](02-tool-dispatch-design-2026-04-06.md) →
3. [Todo 驱动规划](03-todo-driven-planning-2026-04-06.md) →
4. [Subagent 上下文隔离](04-subagent-context-isolation-2026-04-06.md) →
5. [按需加载 Skill](05-on-demand-skill-loading-2026-04-06.md) →
6. [三层上下文压缩](06-three-layer-context-compression-2026-04-06.md) |
7. [持久化任务图](07-persistent-task-dag-2026-04-06.md) →
8. [后台执行](08-background-task-execution-2026-04-06.md) →
9. [多 Agent 团队](09-multi-agent-team-coordination-2026-04-06.md) →
10. [团队协议](10-request-response-protocols-2026-04-06.md) →
11. [自组织团队](11-self-organizing-agent-teams-2026-04-06.md) →
12. [Worktree 任务隔离](12-worktree-task-isolation-2026-04-06.md) →
13. [三角色Agent分工架构](13-three-role-agent-architecture-2026-04-11.md) →
14. **Agent Ready代码库评估**（本文）
