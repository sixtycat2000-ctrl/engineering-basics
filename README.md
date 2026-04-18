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
| [Agent Client Protocol (ACP) 快速概览](agent-client-protocol-2026-04-06.md) | 5 分钟了解 ACP —— AI Agent 和编辑器之间的标准通信协议 |
| [MCP Apps](mcp-apps-guide-2026-04-06.md) | 在 AI 对话里直接渲染交互界面（图表、表单、仪表盘） |

**ACP 深度拆解系列**

| # | 文章 | 一句话 |
|---|------|--------|
| 01 | [ACP 协议全景](acp-01-protocol-overview-2026-04-18.md) | 三个角色、JSON-RPC 2.0、四阶段生命周期 —— AI 编程的 USB 接口 |
| 02 | [ACP 初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) | 版本协商、能力交换、会话创建 —— Agent 和编辑器的第一次对话 |
| 03 | [ACP 交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) | Prompt Turn 6 步生命周期、5 种内容类型、流式推送 —— 用户、Agent、工具的三方对话 |
| 04 | [ACP 工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) | 工具调用、权限请求、文件系统、终端操作 —— Agent 怎么读写文件、跑命令 |
| 05 | [ACP 会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md) | 模式切换、配置选项、斜杠命令、自定义扩展 —— 让协议适应你的场景 |

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

**第四部分：Agent可靠性工程** —— 诊断失败模式，调校运行时底盘

| # | 文章 | 一句话 |
|---|------|--------|
| 21 | [Agent翻车诊断](21-agent-failure-diagnosis-2026-04-17.md) | 同一个模型跑分差3倍——7个失败模式揭示：可靠性是工程问题不是模型问题 |
| 22 | [Agent运行时工程](22-agent-runtime-engineering-2026-04-17.md) | Schema字段顺序、渐进式思考、模型差异适配——运行时才是胜负手 |

### LLM 推理基础

| # | 文章 | 一句话 |
|---|------|--------|
| 15 | [LLM的内存账本](15-llm-memory-budget-2026-04-12.md) | 模型权重大小、KV Cache 内存、运行时开销——三笔账算清 LLM 要多少内存 |
| 16 | [浮点数精度](16-floating-point-precision-2026-04-12.md) | Float32 不是均匀精度——靠近0最密，模型参数也扎堆在0附近，这就是量化的基础 |
| 17 | [推理的两个阶段](17-inference-two-phases-2026-04-12.md) | Prefill 并行算一遍，Decode 逐字吐 N 遍——瓶颈从算力变成内存带宽 |
| 18 | [量化三步曲](18-quantization-algorithms-2026-04-12.md) | 对称量化、非对称量化、分块量化——从 16 位压到 4 位的数学，误差从 117% 降到 8% |
| 19 | [量化效果实测](19-quantization-benchmark-2026-04-12.md) | 8-bit 几乎无损，4-bit 是甜点（4倍压缩2倍加速），2-bit 是悬崖 |
| 20 | [长上下文的救兵](20-long-context-2026-04-12.md) | Flash Attention 不存中间矩阵，TurboQuant 把 KV 压6倍，PagedAttention 按页管理显存 |

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
