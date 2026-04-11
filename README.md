# Engineering Basics

> 看完就会、拿走就用的工程基础指南。每个概念都有代码、有图、有注释。

---

## 文章列表

### 开发工具

| 文章 | 一句话 |
|------|--------|
| [Git Worktree 完整工作流程](git-worktree-guide-2026-03-28.md) | 同一个仓库同时开多个分支，互不干扰 |
| [tmux + AI Coding CLI 实用指南](tmux-ai-coding-guide-2026-03-29.md) | tmux 搭配 Claude Code / Codex 的最佳姿势 |

### 协议与标准

| 文章 | 一句话 |
|------|--------|
| [Agent Client Protocol (ACP)](agent-client-protocol-2026-04-06.md) | AI Agent 和编辑器之间的标准通信协议，编程界的 USB 接口 |
| [MCP Apps](mcp-apps-guide-2026-04-06.md) | 在 AI 对话里直接渲染交互界面（图表、表单、仪表盘） |

### AI Agent 内部机制

**第一部分：单 Agent 基础** —— 从 0 到 1 搭建一个能干活的 Agent

| # | 文章 | 一句话 |
|---|------|--------|
| 1 | [Agent Loop 基础](01-agent-loop-fundamentals-2026-04-06.md) | Agent 的核心就是一个循环：看 → 想 → 做 → 看结果 |
| 2 | [工具分发设计](02-tool-dispatch-design-2026-04-06.md) | Agent 怎么决定调哪个工具、传什么参数——dispatch map 是最纯粹的开放封闭原则 |
| 3 | [Todo 驱动规划](03-todo-driven-planning-2026-04-06.md) | 让 Agent 有计划：拆任务、排优先级、跟踪进度，nag 机制制造问责压力 |
| 4 | [Subagent 上下文隔离](04-subagent-context-isolation-2026-04-06.md) | 大任务拆小：子 Agent 用干净的上下文跑完，父 Agent 只收摘要 |
| 5 | [按需加载 Skill](05-on-demand-skill-loading-2026-04-06.md) | 懒加载知识——系统提示放目录，需要时才加载完整内容 |
| 6 | [三层上下文压缩](06-three-layer-context-compression-2026-04-06.md) | 对话太长怎么办——micro / auto / manual 三层压缩，像垃圾回收一样管理上下文 |

**第二部分：多 Agent 协作** —— 从 1 到 N，让 Agent 团队自组织

| # | 文章 | 一句话 |
|---|------|--------|
| 7 | [持久化任务图](07-persistent-task-dag-2026-04-06.md) | 扁平清单不够用——任务之间的依赖关系用 DAG 管理，持久化到磁盘 |
| 8 | [后台执行](08-background-task-execution-2026-04-06.md) | 长任务丢后台：Agent 继续想下一步，完成后通知注入 |
| 9 | [多 Agent 团队](09-multi-agent-team-coordination-2026-04-06.md) | 多 Agent 组队：JSONL 邮箱通信，append-only 最简协调 |
| 10 | [团队协议](10-request-response-protocols-2026-04-06.md) | 关机握手、计划审批——request-response + FSM 驱动所有协商 |
| 11 | [自组织团队](11-self-organizing-agent-teams-2026-04-06.md) | 队友自己看任务板认领工作，空闲超时自动下班 |
| 12 | [Worktree 任务隔离](12-worktree-task-isolation-2026-04-06.md) | 每个 task 绑定独立 git worktree，互不污染 |

**第三部分：多Agent系统工程** —— 从 N 到可靠，用分工和验证收敛复杂度

| # | 文章 | 一句话 |
|---|------|--------|
| 13 | [三角色Agent分工架构](13-three-role-agent-architecture-2026-04-11.md) | Orchestrator 规划、Worker 实现、Validator 验收——上下文即命运 |
| 14 | [Agent Ready代码库评估](14-agent-ready-codebase-2026-04-11.md) | 8支柱 × 5等级 × 60+标准：代码库离AI自主开发还差什么 |

### 写作风格

| 文章 | 一句话 |
|------|--------|
| [写作风格指南](WRITING-STYLE-GUIDE.md) | 本仓库的写作规范——把隐性知识显性化的 7 条规则 |

---

## 写作原则

每篇文章遵循同样的风格：

- **先为什么，再怎么做** —— 感受到痛点才会认真看方案
- **可复制粘贴** —— 代码块直接能跑，没有占位符
- **有图有表** —— 架构用 ASCII 图，对比用表格，命令有速查表
- **三档递进** —— 入门 5 分钟搞定 → 进阶日常够用 → 高级专家场景
- **注释在代码里** —— 解释「为什么」，不翻译「是什么」

想给这个仓库贡献文章？先看 [写作风格指南](WRITING-STYLE-GUIDE.md)。
