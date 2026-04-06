# AI Todo Planning：让 Agent 有计划

> 没有计划的 Agent 走哪算哪。先列步骤再动手，完成率翻倍。Todo 列表是 LLM 的外挂工作记忆——不衰减、不被挤出注意力。

---

## 为什么需要计划

多步任务中，模型会丢失进度。这不是 bug，是注意力机制的结构性限制：

- **重复做过的事** —— 模型不记得第 3 步已经做过了，在第 7 步又做一遍
- **跳步** —— 直接跳到第 8 步，漏掉中间的关键步骤
- **跑偏** —— 做到一半开始即兴发挥，偏离原始目标
- **对话越长越严重** —— 工具结果不断填满上下文，系统提示的影响力逐渐被稀释

一个 10 步重构任务，模型做完 1-3 步就开始即兴发挥，因为 4-10 步的计划已经被挤出了注意力窗口。

**解决方案：把计划写在模型每次都能看到的地方。**

---

## 工作原理

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  ← doing   |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

### TodoManager：带状态的任务清单

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

**同一时间只允许一个 `in_progress`。** 这是核心约束。为什么？

因为上下文切换有成本。LLM 开始做任务 B 时，脑子里关于任务 A 的推理质量会下降。如果同时标记两个 in_progress，说明模型在多线程工作——但 LLM 是单线程的。强制一个 in_progress = 强制顺序聚焦。

### 注册为工具

```python
TOOL_HANDLERS = {
    # ...s02 的基础工具...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

todo 就是一个普通工具。模型调 `todo` 更新计划，和调 `read_file` 读文件没有本质区别。

### Nag reminder：问责压力

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

**模型连续 3 轮以上不调用 `todo` 时，注入提醒。**

这个设计的心理学原理：nag 制造问责压力。模型可以选择忽略，但通常不会——因为提醒被包装成了系统要求（`<reminder>` 标签），模型倾向于服从。

为什么是 3 轮？不是 1 轮（太频繁，打断正常工作流）也不是 10 轮（太晚了，模型已经跑偏了）。3 轮是一个经验值：模型做 3 步操作不更新计划还算正常（可能在一个复杂工具调用上连续工作），超过 3 步还不更新，大概率是忘了。

---

## 隐性知识

### 注意力衰减是 LLM 的根本问题

Transformer 的自注意力机制理论上能关注所有 token，但实际上：
- **位置偏见**：模型更关注开头和结尾，中间的内容被"稀释"
- **新信息覆盖旧信息**：最新的工具结果比 10 步前的计划获得更多注意力
- **指令稀释**：随着上下文变长，系统提示（包括"按计划执行"这类指令）的影响力下降

Todo 列表解决这些问题的方法：**每次模型调用 `todo`，整个计划作为工具结果重新注入上下文尾部。** 这相当于把计划"搬"到了模型注意力最高的位置。

### "只允许一个 in_progress" 是反多任务的

人类的多任务已经被证明是快速切换而非真正并行。LLM 也一样——它不能同时推理两个问题。强制一个 in_progress 的效果：
- 模型必须显式地"完成 A → 开始 B"，而不是模糊地"同时在处理 A 和 B"
- 状态转换（pending → in_progress → completed）成为可观察的行为
- 如果模型卡在一个任务上，nag 会推它去做下一步

### Nag 是行为设计，不是技术功能

这个提醒机制来自行为心理学：
1. **可见性**：让期望行为保持显眼（像把健身服放在床边）
2. **触发器**：不是惩罚不更新，而是温和地提醒"你是不是该更新了"
3. **低成本**：模型只需要调用一次 todo 工具就能消除提醒

把 nag 想象成一个项目管理员站在你身后，不说话，但每隔一会儿看你一眼。你不一定每次都响应，但你不会完全忘掉计划。

---

## 相对 s02 的变更

| 组件 | s02 | s03 |
|------|-----|-----|
| 工具数量 | 4 | 5（+todo） |
| 规划能力 | 无 | 带状态的 TodoManager |
| Nag 注入 | 无 | 3 轮后注入 `<reminder>` |
| Agent loop | 简单分发 | + `rounds_since_todo` 计数器 |

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
