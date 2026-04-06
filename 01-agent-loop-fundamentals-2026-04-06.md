# AI Agent Loop：循环即 Agent

> LLM 是纯函数——同样的输入永远给同样的输出，没有记忆。循环是唯一让模型"记住"之前发生了什么的东西。一个循环 + 一个工具调用 = 一个完整的 Agent。

---

## 为什么是循环

LLM 的每次调用都是独立的。它不知道上一秒自己做了什么，不知道文件有没有被修改，不知道命令跑没跑完。**消息列表（messages）就是它全部的工作记忆。**

不循环，你就是一个复制粘贴的人肉中继：
1. 用户说"创建 hello.py"
2. 你把 prompt 发给 LLM，LLM 回复"我要调用 bash 创建文件"
3. 你手动执行 bash，把输出粘回 prompt
4. 再发给 LLM……

循环把这个人工环节自动化了。循环的每一次迭代都在做同一件事：**把 LLM 的决策变成动作，把动作的结果喂回 LLM。**

---

## 核心流程

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

四个步骤，永远不变：

**1. 用户 prompt 作为第一条消息**

```python
messages.append({"role": "user", "content": query})
```

`messages` 是唯一的状态。LLM 没有隐藏的内部状态——你塞进 messages 里的内容就是它能看到的全部。

**2. 把消息和工具定义一起发给 LLM**

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

`tools` 参数告诉模型"你可以用这些工具"。没有这个参数，模型只能输出文本，不能调用工具。

**3. 检查 stop_reason——这是退出条件**

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

**退出条件由模型决定，不是由代码决定。** 这是 Agent 和脚本的分界线——脚本什么时候停由人决定，Agent 什么时候停由模型自己决定。模型认为自己做完了（没有工具要调用了），循环就结束。

**4. 执行工具，把结果追加为 user 消息，回到第 2 步**

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

注意：工具结果以 `user` 角色追加。这是因为 API 要求 tool_result 必须跟在 assistant 的 tool_use 后面，而消息列表的角色必须交替。对 LLM 来说，这些不是"用户说的话"，而是"工具执行的结果"——它通过 `tool_result` 类型来区分。

---

## 完整实现

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

不到 30 行，这就是整个 Agent。后面 11 个章节都在这个循环上叠加机制——**循环本身始终不变**。

---

## 隐性知识

### 消息列表是不可变日志

`messages` 只能追加，不能修改（除了后面章节的压缩机制）。每一次交互都永久记录在里面。这意味着：
- 模型能"记住"之前做了什么，因为消息列表告诉了它
- 上下文窗口总有一天会满，这就是后面 s06 要解决的问题
- 消息列表越长，模型的注意力越分散——它不能平等地关注每一条消息

### stop_reason 是模型对自己的判断

`stop_reason` 有两种主要值：
- `tool_use`：模型决定调用工具，循环继续
- `end_turn`：模型认为任务完成，循环结束

模型不是"被迫停止"的，而是"自己决定停下来"。这个设计意味着 Agent 有自主权——它可以在认为需要更多工具调用时继续，也可以在认为已经回答了用户问题时停止。**这个退出条件的自主性，是 Agent 和传统自动化脚本的本质区别。**

### 为什么只有 bash 一个工具

第一个版本只需要一个工具：bash。因为 bash 是图灵完备的——理论上任何操作都能通过 bash 完成。读文件用 `cat`，写文件用 `echo >`，搜索用 `grep`。

但后面 s02 会引入专用工具（read_file、write_file），原因是：
- bash 里 `cat` 截断不可预测
- `sed` 遇到特殊字符就崩
- 每次 bash 调用都是不受约束的安全面

专用工具不是"更好用"，而是"更安全、更可控"。

### max_tokens 是安全阀

`max_tokens=8000` 限制模型单次回复的最大长度。这不是为了省钱（虽然也有这个效果），而是为了防止单次调用占用太多上下文空间。一个没完没了的回复比一个简短的回复更难被后续迭代消化。

---

## 这一层的组件

| 组件 | 说明 |
|------|------|
| Agent loop | `while True` + `stop_reason` 退出条件 |
| Tools | `bash`（唯一工具） |
| Messages | 累积式消息列表，只追加不改 |
| Control flow | `stop_reason != "tool_use"` |

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
