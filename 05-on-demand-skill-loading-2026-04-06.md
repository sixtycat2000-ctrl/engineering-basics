# AI Skill Loading：按需加载知识

> 用到什么知识，临时加载什么知识。10 个 Skill 每个 2000 token 全塞进系统提示就是 20000 token 的永久税。按需加载是懒求值（lazy evaluation）用在 prompt engineering 上的版本。

---

## 为什么不能全塞进系统提示

你希望 Agent 遵循各种领域工作流：git 约定、测试模式、代码审查清单、部署流程。每个领域知识写成 Skill 大约 2000 token。

全塞进系统提示的代价：

| Skill 数量 | 系统提示 token | 每次调用的额外成本 |
|-----------|---------------|-------------------|
| 1 | +2,000 | 可忽略 |
| 5 | +10,000 | 开始有感觉 |
| 10 | +20,000 | 明显变贵 |
| 20 | +40,000 | 不可接受 |

**系统提示里的每个 token 都在每次 LLM 调用时重复计费。** 一个和当前任务无关的 Skill（比如你在修 bug 时加载的部署流程 Skill），每次调用都在浪费 token。

而且还有更微妙的问题：系统提示越长，模型越容易"选择性忽略"。人类面对过长的指令也会抓不住重点，LLM 一样。

---

## 两层架构

```
System prompt (Layer 1 — always present, cheap):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 — on demand):   |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

**第一层**：系统提示中放 Skill 名称和简短描述（~100 token 每个）。模型知道"有哪些技能可用"，但不看详细内容。

**第二层**：模型需要时，调 `load_skill` 工具，完整内容通过 tool_result 注入。内容只在需要时出现，不需要时不占任何空间。

---

## 工作原理

### Skill 文件结构

每个 Skill 是一个目录，包含 `SKILL.md` 文件和 YAML frontmatter：

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

为什么用文件而不是代码里的字典？因为：
- Skill 内容可能很长（2000+ token 的 Markdown），不适合嵌在代码里
- 非程序员可以编辑 SKILL.md 来定制 Agent 行为
- 文件系统天然支持目录结构——按领域组织 Skill

### SkillLoader：扫描和注册

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

`get_descriptions()` 用于第一层（写入系统提示）。`get_content()` 用于第二层（作为工具返回值）。

### 整合到 Agent

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...基础工具...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

就这么简单。系统提示里列出 Skill 目录，工具表里加一个 `load_skill`。模型自己决定什么时候加载。

---

## 隐性知识

### 懒加载是 prompt engineering 的核心策略

这个模式和编程里的懒加载完全一样：
- **急切加载（Eager）**：启动时把所有 Skill 塞进系统提示 → 快但贵
- **懒加载（Lazy）**：只放目录，需要时再加载详情 → 慢一度但省

为什么"慢一度"可接受？因为 `load_skill` 只调一次。模型第一次看到 Skill 目录，决定加载需要的那个，之后完整内容就在上下文里了。一次调用的延迟换来整个会话的 token 节省。

### 描述文本是模型的"搜索引擎"

模型选择加载哪个 Skill，完全基于系统提示里的简短描述。如果描述写得不好：
- `"test": "Testing stuff"` → 模型不知道这个 Skill 包含什么，可能不加载
- `"test": "Pytest 最佳实践，包括 fixture 设计、mock 策略、覆盖率配置"` → 模型能精确判断是否需要

**Skill 描述 = 搜索索引。** 写得越精确，模型的选择越准确。

### tool_result 注入比 system prompt 注入更好

为什么 Skill 内容通过 tool_result 而不是直接修改系统提示？

1. **不可变性**：系统提示应该是静态的。运行时修改系统提示容易出错，模型可能对"突然变了"感到困惑
2. **位置优势**：tool_result 在上下文尾部，模型对尾部内容的注意力更高
3. **自然对话流**：模型"请求"Skill（工具调用）→ 系统"提供"Skill（工具结果），比悄悄塞进系统提示更自然
4. **可追踪**：模型调用了 `load_skill`，你能在日志里看到；内容进了 tool_result，位置明确

### `<skill>` XML 标签的意义

返回内容用 `<skill name="git">...</skill>` 包裹。这不是装饰：
- 模型能通过标签结构区分"Skill 内容"和"普通工具输出"
- 标签提供了明确的边界——Skill 内容从哪开始、到哪结束
- 多个 Skill 可以同时加载，标签让模型知道每个 Skill 的边界

---

## 相对 s04 的变更

| 组件 | s04 | s05 |
|------|-----|-----|
| 工具 | 5（基础 + task） | 5（基础 + load_skill） |
| 系统提示 | 静态字符串 | + Skill 描述列表 |
| 知识库 | 无 | skills/*/SKILL.md 文件 |
| 注入方式 | 无 | 两层（系统提示 + tool_result） |

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
12. [Worktree 任务隔离](12-worktree-task-isolation-2026-04-06.md)
