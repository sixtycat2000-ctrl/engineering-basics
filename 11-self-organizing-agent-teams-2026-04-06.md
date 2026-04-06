# AI Autonomous Agents：自组织团队

> 领导不再逐个派活。队友自己看任务板，有活就认领，做完找下一个。空闲 60 秒没人理，自动下班。上下文压缩后忘了自己是谁？身份重注入保住记忆。

---

## 为什么需要自治

s09-s10 中，队友只在被明确指派时才动。领导得给每个队友写 prompt，任务看板上 10 个未认领的任务得手动分配。

想象一个 10 人团队，领导给每个人发"去做第 3 号任务"的消息。10 条消息，10 次等待回复，10 次确认。**这扩展不了。**

真正的自治：队友自己扫描任务看板，认领没人做的任务，做完再找下一个。领导只管"往看板上放任务"和"从看板上收结果"。

还有一个微妙的问题：s06 的上下文压缩后，Agent 可能忘了自己的身份。"我是谁？我在做什么任务？" 如果不解决这个问题，压缩后的 Agent 就像失忆的员工。

---

## 架构

```
Teammate lifecycle with idle cycle:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (or idle tool called)
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox → message? ----------→ WORK
    |
    +---> scan .tasks/ → unclaimed? ------→ claim → WORK
    |
    +---> 60s timeout ---------------------→ SHUTDOWN

Identity re-injection after compression:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

---

## 工作原理

### WORK 和 IDLE 两个阶段

```python
def _loop(self, name, role, prompt):
    while True:
        # -- WORK PHASE --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # 执行工具...
            if idle_requested:
                break

        # -- IDLE PHASE --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

WORK 阶段就是标准的 Agent loop。当 LLM 停止调用工具（或调用了 `idle` 工具），进入 IDLE 阶段。

### IDLE 阶段：轮询收件箱 + 扫描任务板

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12 次
        time.sleep(POLL_INTERVAL)

        # 优先检查收件箱
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True

        # 然后扫描任务板
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True

    return False  # 60 秒无事可做 → shutdown
```

每 5 秒轮询一次：
1. 先检查收件箱——消息优先级高于任务（可能是领导的紧急指令）
2. 再扫描任务板——找没人认领的任务
3. 都没有 → 继续等
4. 等 60 秒还没事 → 自动关机

### 任务板扫描

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

三个条件：
- `status == "pending"` —— 还没开始
- `owner == ""` —— 没人认领
- `blockedBy == []` —— 没有未完成的前置依赖

**按文件名排序（ID 顺序）**：多个可认领任务时，优先认领最早创建的。

### 身份重注入

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

**检测压缩的信号：messages 长度 ≤ 3。** 正常对话不可能这么短，除非刚发生了压缩。

插入两轮模拟对话：用户告诉 Agent 身份，Agent 确认。这让模型"回忆起"自己是谁。

---

## 隐性知识

### 轮询 vs 推送

为什么用轮询（每 5 秒检查一次）而不是推送（有消息就通知）？

技术上推送更高效（不用空转）。但推送需要：
- 事件总线或观察者模式
- 注册/注销机制
- 线程间信号

轮询只需要：`time.sleep(5)` + 读文件。**简单到不可能出错。**

在 Agent 场景下，5 秒延迟完全可接受。这不是高频交易系统——一个 Agent 晚 5 秒认领任务，不会造成任何问题。

### 自动关机是资源回收

空闲 60 秒后自动关机 = 自动回收不再需要的 LLM 调用配额。每个空闲 Agent 都在消耗线程资源，虽然不调 LLM（idle 阶段只有 sleep + 读文件），但仍然占用内存和线程。

60 秒是经验值：
- 太短（10 秒）：Agent 刚完成一个任务，还没来得及从看板认领下一个就被关了
- 太长（10 分钟）：浪费资源
- 60 秒：给 Agent 足够时间认领新任务，但不无限等待

### 认领竞争怎么解决

两个 Agent 同时扫描到同一个任务，同时认领？在实际实现中，`claim_task()` 应该是原子操作——先检查 owner 是否为空，为空才写入。如果两个 Agent 同时写同一个 JSON 文件，文件系统层面的并发保护可以避免数据损坏（但可能出现两个 Agent 都以为认领成功的情况）。

更健壮的实现可以用文件锁（`fcntl.flock`）。但在大多数场景下，竞争窗口极小（两个 Agent 恰好同一毫秒扫描到同一任务），可以忽略。

### 身份重注入是"失忆后的唤醒"

把压缩想象成"Agent 昏迷了"。醒来后它不记得自己是谁、在做什么。身份重注入就是"叫它的名字，告诉它角色"。

为什么用两轮对话而不是直接修改系统提示？因为模型对"对话历史"的注意力高于"系统提示"。一个 `<identity>` 消息在对话开头比系统提示里的角色描述更容易被注意到。

---

## 相对 s10 的变更

| 组件 | s10 | s11 |
|------|-----|-----|
| 工具 | 12 | 14（+idle, +claim_task） |
| 自治性 | 领导指派 | 自组织 |
| 空闲阶段 | 无 | 轮询收件箱 + 任务看板 |
| 任务认领 | 仅手动 | 自动认领未分配任务 |
| 身份 | 系统提示 | + 压缩后重注入 |
| 超时 | 无 | 60 秒空闲 → 自动关机 |

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
