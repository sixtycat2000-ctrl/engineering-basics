# AI Subagent：上下文隔离

> 大任务拆小，每个小任务用干净的上下文。Subagent 用独立的 messages[]，不污染主对话。跑完 30 次工具调用，父 Agent 只收到一段摘要——token 预算省了，思维清晰度保住了。

---

## 为什么需要隔离

Agent 工作越久，messages 数组越胖。每次读文件、跑命令的输出都**永久**留在上下文里。

一个典型场景：用户问"这个项目用什么测试框架？" 模型需要读 5 个文件（package.json、README、配置文件等）才能回答。但父 Agent 只需要一个词：**"pytest"。**

那 5 个文件的完整内容留在上下文里就是噪音——占空间、分散注意力、挤占后续推理的 token 预算。

**Subagent 的本质：用计算换清晰。** 多花一次 LLM 调用的钱，换来主对话的干净。

---

## 工作原理

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | ← 全新上下文
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean.
Subagent context is discarded after use.
```

### 工具定义

父 Agent 有 `task` 工具，可以生成 Subagent。Subagent 拥有除 `task` 外的所有基础工具——**禁止递归生成**（一个 Subagent 不能再生 Subagent）。

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

禁止递归不是技术限制——技术上完全可以让 Subagent 也生成 Subagent。但递归会带来：
- 调用深度不可控，一个任务可能裂变成 10 个子任务
- Token 成本指数增长
- 调试难度翻倍

**限制递归深度 = 限制复杂度爆炸。**

### Subagent 执行

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # 安全上限
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Subagent 跑自己的循环（和 s01 的循环完全一样），最多 30 轮。跑完后，整个 `sub_messages` 直接丢弃。父 Agent 只收到最后一轮的文本内容，作为 `tool_result` 返回。

---

## 隐性知识

### Subagent 是顾问，不是员工

把 Subagent 想象成外部顾问：你有一个具体问题（"这个项目的测试框架是什么？"），请顾问来调查。顾问翻遍了 5 个抽屉（读了 5 个文件），给了你答案（"pytest"）。顾问走了，你不需要记住他翻了哪些抽屉。

对比不用 Subagent：你自己翻了 5 个抽屉，每个抽屉的东西都记在你脑子里。后来你想集中精力写代码时，脑子里还塞着 5 个文件的内容。

### Token 是最稀缺的资源

一次 LLM 调用的成本 = 输入 token 数 × 单价。上下文越长，每次调用的输入 token 就越多，成本越高。

Subagent 的经济学：
- 不用 Subagent：5 个文件内容留在主对话，后续每次 LLM 调用都为它们付 token 费
- 用 Subagent：多花一次 Subagent 的调用费用，但主对话里只有一段摘要

**在长会话中，Subagent 每次主循环迭代都为你省钱。** 因为那些被省掉的 token 不再参与每一轮的输入计费。

### 安全上限 30 轮的意义

30 轮不是随便选的。大多数调查类任务 5-10 轮就够。30 轮是一个安全网——防止 Subagent 陷入死循环（比如反复读同一个文件、反复执行失败的命令）。

这个上限和 Agent loop 的 `max_tokens` 一样，是安全阀而不是功能限制。

### 摘要质量取决于最后一轮

`return "".join(b.text for b in response.content if hasattr(b, "text"))` 只返回最后一轮的文本。这意味着：

- Subagent 的工作质量取决于它在最后一轮是否给出了**完整的总结**
- 如果模型在最后一轮只说"Done."而没有总结做了什么，父 Agent 就白等了
- 系统提示（`SUBAGENT_SYSTEM`）应该明确要求 Subagent 在完成时给出总结

这是 Subagent 设计中唯一的脆弱点——摘要质量完全依赖模型的自觉性。

---

## 相对 s03 的变更

| 组件 | s03 | s04 |
|------|-----|-----|
| 工具 | 5（基础 + todo） | 5（基础）+ task（仅父端） |
| 上下文 | 单一共享 | 父 + 子隔离 |
| Subagent | 无 | `run_subagent()` 函数 |
| 返回值 | 不适用 | 仅摘要文本 |

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
