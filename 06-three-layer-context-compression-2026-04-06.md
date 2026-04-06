# AI Context Compact：上下文压缩

> 上下文总会满。读一个 1000 行文件就吃掉 ~4000 token；读 30 个文件、跑 20 条命令，轻松突破 100k。三层压缩策略，从静默到激进，换来无限会话。

---

## 为什么上下文会满

上下文窗口是固定大小的。模型能"看到"的内容量有硬性上限。

典型消耗：

| 操作 | 估算 token |
|------|-----------|
| 读一个 1000 行文件 | ~4,000 |
| 跑一条 bash 命令（含输出） | ~500-2,000 |
| 模型的一次回复 | ~500-2,000 |
| 10 轮工具调用后 | ~30,000-50,000 |
| 30 轮工具调用后 | 突破上限 |

**不压缩，Agent 在大项目里根本没法干活。** 它读不了几个文件就满了。

---

## 三层压缩策略

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        ← 静默，每轮自动执行
  3 轮以前的 tool_result
  替换为 "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              保存完整对话到 .transcripts/
              LLM 生成摘要
              所有消息替换为 [summary]
                    |
                    v
            [Layer 3: compact 工具]
              模型主动调用 compact
              和 auto_compact 一样的摘要机制
```

三层的设计哲学：**不同的场景需要不同的激进程度。**

---

## 第一层：micro_compact（静默）

每次 LLM 调用前自动执行，模型完全无感知。

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

**只保留最近几轮的完整 tool_result，之前的替换为占位符。**

为什么不是"删掉"而是"替换"？因为消息列表的结构不能随意改变——tool_result 必须和之前的 tool_use 配对。用占位符替换内容，保持结构完整，同时释放空间。

为什么保留"最近几轮"？因为模型可能还在基于最近几轮的输出推理。把刚读的文件内容也压缩了，模型会"失忆"。

### 隐性洞察

micro_compact 是有损压缩。它假设"旧的上下文不如新的重要"。这个假设在大多数场景下成立——模型正在处理的问题通常依赖最近的工具输出，而不是 20 步前的。但在需要"回头看"的场景（比如比较文件 A 和文件 B，中间隔了 15 步操作），micro_compact 会让模型失去这个能力。

---

## 第二层：auto_compact（紧急）

token 数超过阈值时触发。这是"紧急手术"——当上下文已经臃肿到影响工作，必须暴力压缩。

```python
def auto_compact(messages: list) -> list:
    # 1. 保存完整对话到磁盘（不丢信息）
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")

    # 2. 让 LLM 做摘要
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
    ]
```

两个关键步骤：
1. **先存再压缩**：完整对话保存到 `.transcripts/` 目录。信息没有丢失，只是移出了活跃上下文
2. **LLM 做摘要**：用 LLM 自己来总结之前的对话。这比算法压缩好——LLM 知道哪些信息对后续工作重要

### 隐性洞察

auto_compact 用 LLM 来压缩 LLM 的上下文——这听起来浪费（花钱压缩来省钱），但实际上是最优解。因为只有 LLM 自己知道"在当前任务中，哪些信息是关键线索"。一个简单的文本摘要算法不知道用户在做什么任务，可能会丢掉关键细节。

保存到磁盘是恢复能力的保障。如果压缩后模型丢失了关键信息，理论上可以从 transcript 恢复（虽然当前实现没有自动恢复机制）。

---

## 第三层：compact 工具（主动）

模型自己调用的 `compact` 工具，和 auto_compact 用相同的摘要机制。

```python
TOOL_HANDLERS = {
    # ...基础工具...
    "compact": lambda **kw: compact_messages(messages),
}
```

为什么需要手动触发？因为模型可能意识到"我在一个很长的对话里，我需要清理一下"。比 auto_compact 更主动——模型自己判断什么时候压缩，而不是等 token 阈值触发。

---

## 循环中的整合

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Layer 1: 每轮静默
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Layer 2: 紧急时自动
        response = client.messages.create(...)
        # ... tool execution ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Layer 3: 按需手动
```

---

## 隐性知识

### 上下文管理 = 垃圾回收

这个三层策略和编程语言的垃圾回收惊人地相似：
- **引用计数**（micro_compact）：即时回收确定无用的对象
- **标记-清除**（auto_compact）：内存紧张时全面回收
- **手动 free**（compact 工具）：程序员主动释放

同样的分代策略：越老的对象（越旧的 tool_result）越容易被回收。

### 压缩后的"失忆"是可接受的

auto_compact 后，模型对之前对话的记忆完全依赖摘要质量。摘要可能丢掉：
- 具体的文件内容（"文件里有一行 `debug=True`"）
- 中间推理过程（"我尝试了方法 A 失败了，所以用了方法 B"）
- 工具调用的精确输出（"测试结果显示第 3 个用例失败"）

但保留了：
- 当前任务目标
- 已完成的步骤
- 关键决策和原因
- 未解决的问题

**压缩保住了骨架，丢了血肉。** 对于继续工作来说，骨架通常够用。

### 50000 token 的阈值怎么选

阈值不能太高（压缩太晚，浪费 token）也不能太低（压缩太频繁，模型不断失忆）。50000 是一个经验值，基于：
- 典型模型的上下文窗口 128k-200k token
- 保留足够空间给后续工具调用
- 摘要本身也要占空间（~2000 token）

如果你的模型上下文窗口更大（比如 200k），可以调高阈值。如果用更小的模型（比如 32k 窗口），必须调低。

---

## 相对 s05 的变更

| 组件 | s05 | s06 |
|------|-----|-----|
| 工具 | 5 | 5 + compact |
| 上下文管理 | 无 | 三层压缩 |
| Micro-compact | 无 | 旧结果 → 占位符 |
| Auto-compact | 无 | token 阈值触发 |
| Transcripts | 无 | 保存到 .transcripts/ |

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
