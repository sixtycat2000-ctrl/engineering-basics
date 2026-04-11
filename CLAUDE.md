# CLAUDE.md — Engineering Basics 项目指南

## 项目定位

中文技术写作仓库。"看完就会、拿走就用"的工程基础指南。每篇文章讲透一个概念，有代码、有图、有注释。

## 仓库结构

```
├── README.md                     # 文章索引，新增文章必须更新
├── WRITING-STYLE-GUIDE.md        # 写作风格指南（7条规则，写新文章前必读）
├── CLAUDE.md                     # 本文件
│
├── {NN}-{topic}-{date}.md        # AI Agent 内部机制系列（编号01-14+）
├── {topic}-{date}.md             # 独立指南（Git Worktree、tmux、ACP、MCP Apps）
│
└── info-cards/                   # 所有文章的配图/信息卡片
    └── {NN}-{topic}.png          # 用 qiaomu-info-card-designer skill 生成
```

## 新增文章流程

1. **读 `WRITING-STYLE-GUIDE.md`** — 7条规则是硬约束，不是建议
2. **写文章** — 跟随文章结构模板（为什么 → 架构图 → 核心概念 → 步骤 → 实战 → 速查表）
3. **更新 `README.md`** — 在对应分类里加一行
4. **更新系列导航** — 在前后文章的"系列导航"部分更新链接（仅编号系列01-14+）
5. **生成 info card** — 用 `/qiaomu-info-card-designer` skill 生成卡片，保存到 `info-cards/`

## 文件命名

- 编号系列：`{NN}-{english-topic}-{YYYY-MM-DD}.md`，如 `13-three-role-agent-architecture-2026-04-11.md`
- 独立指南：`{english-topic}-{YYYY-MM-DD}.md`，如 `agent-client-protocol-2026-04-06.md`
- 编号从 `01` 开始，连续递增

## 写作硬约束（从 WRITING-STYLE-GUIDE.md 提炼）

### 结构
- 先"为什么需要"，再"怎么做"
- 每篇文章有至少 1 张 ASCII 图 + 1 个表格
- 结尾有参考资料链接

### 代码
- 可复制粘贴直接运行，不用占位符 `<your-path>`
- 注释解释"为什么"，不翻译"是什么"
- 关键步骤有验证命令

### 语言
- **全中文撰写**
- 短句优先，一句不超过 30 字
- 用词口语化：用"跑"不用"运行"，用"就行"不用"即可"
- 技术术语不翻译：`JSON-RPC`、`iframe`、`postMessage`

### 三档递进
- 入门：5 分钟能跑
- 进阶：日常 80% 够用
- 高级：专家场景

### 格式
- 标题格式：`# 主题：一句类比或定位`
- 开头有 blockquote 摘要
- 用 `---` 分隔大章节
- 错/对用 ❌ / ✅ 标注
- ASCII 图不超过 60 字符宽

### Info Card 标题
- 主标题要结论性的，让读者看到就想知道"为什么"
- 用数字/动词驱动，不用描述性标题
- 可用 AskUserQuestion 让用户选标题方向

## Git 约定

- commit message 中文，简洁描述改动
- 不需要 push，用户会说什么时候 push
- 新文章用独立 commit，不和其他改动混在一起
