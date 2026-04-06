# AI Agent Teams：多 Agent 团队

> 一个 Agent 干不完的活，交给团队。每个队友是独立的 Agent loop，有自己的上下文、自己的推理。通过 JSONL 邮箱通信——append-only，drain-on-read，最简单的协调机制。

---

## 为什么 Subagent 和 Background Tasks 不够

**Subagent（s04）** 是一次性的：生成、干活、返回摘要、消亡。没有身份，没有跨调用的记忆。你不能对 Subagent 说"继续昨天的工作"，因为它已经不存在了。

**Background Tasks（s08）** 能跑 shell 命令，但做不了 LLM 引导的决策。`npm install` 可以丢后台，但"分析这个 bug 的根因并写修复代码"不行——这需要 LLM 推理。

真正的团队协作需要三样东西：
1. **持久 Agent** —— 能跨多轮对话存活
2. **身份和生命周期** —— 知道谁是谁、谁在忙、谁空闲
3. **通信通道** —— Agent 之间能互相发消息

---

## 架构

```
Teammate lifecycle:
  spawn → WORKING → IDLE → WORKING → ... → SHUTDOWN

Communication:
  .team/
    config.json           ← team roster + statuses
    inbox/
      alice.jsonl         ← append-only, drain-on-read
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | ----------------------------→ |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ↑                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl → read + drain ---------+
```

---

## 工作原理

### TeammateManager：团队名册

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

`config.json` 是团队的名册——谁在团队里、角色是什么、当前状态。这是团队状态的唯一真相源。

### 生成队友

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

生成一个队友 = 在 config.json 注册 + 启动一个独立线程的 Agent loop。每个队友是一个完整的 Agent——有自己的 messages、自己的工具调用、自己的推理。

### MessageBus：JSONL 收件箱

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines()
                if l]
        path.write_text("")  # drain
        return json.dumps(msgs, indent=2)
```

两个操作：
- `send()`：往收件人的文件追加一行 JSON（append-only）
- `read_inbox()`：读取所有行，然后清空文件（drain-on-read）

**为什么是 JSONL？** 因为：
- Append-only 没有并发写入冲突（两个 Agent 同时给同一个人发消息，各追加各的）
- Drain-on-read 实现了"读过的消息不重复"
- 比数据库简单，比共享内存安全，比网络 socket 可靠

### 队友的主循环

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        # 每次 LLM 调用前检查收件箱
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # 执行工具，追加结果...
    self._find_member(name)["status"] = "idle"
```

每次循环迭代前先检查收件箱。有消息就注入上下文，让模型决定如何响应。

---

## 隐性知识

### JSONL 收件箱 = 最简单的协调机制

为什么不用 Redis、不用 RabbitMQ、不用 SQLite？因为这些在 Agent 场景下都是过度设计。Agent 的消息频率很低（每几秒一条），数据量很小（几 KB）。文件系统完全够用。

**Append-only + drain-on-read 的组合消除了所有并发问题：**
- 多个发送者同时写 → 各追加各的，文件系统保证原子性
- 读写竞争 → drain-on-read 是原子操作（读全部 + 清空）
- 消息丢失 → 不可能，只要写入成功就在文件里

这是分布式系统里"最弱的一致性模型"但也是"最简单的实现"。在 Agent 场景下，简单 = 可靠。

### 团队通信是异步的

发送者调用 `send()` 后立即返回，不等待接收者处理。接收者在下一次循环迭代时才看到消息。

这意味着：
- 发送者不能假设消息已被处理
- 消息可能有延迟（接收者正在忙于工具调用）
- 需要协议来确认收到（s10 会解决）

**异步通信是 Agent 团队和人类团队的区别之一。** 人类可以同步对话（"你做完了吗？""做完了"），Agent 的通信更像发邮件——发出去了，对方有空再看。

### 每个队友是完整的 Agent

每个队友都有自己的 messages 列表、自己的工具集、自己的推理过程。这意味着：
- Token 成本是线性的：N 个队友 = N 倍的 LLM 调用
- 每个队友的上下文是隔离的——alice 不知道 bob 的对话内容
- 队友之间只能通过消息通信，没有共享内存

这是分布式系统的 CAP 定理在 Agent 领域的体现：你选择了分区容错（每个 Agent 独立运行），就必须放弃强一致性（队友之间的状态可能不同步）。

### config.json 是活的状态

不是静态配置。每次队友状态变化（working → idle → shutdown）都更新 config.json。任何 Agent 都可以读它来了解团队状态。这就是 s11 自治 Agent 的基础——队友需要知道"还有谁在"。

---

## 相对 s08 的变更

| 组件 | s08 | s09 |
|------|-----|-----|
| 工具 | 6 | 9（+spawn/send/read_inbox） |
| Agent 数量 | 单一 | 领导 + N 个队友 |
| 持久化 | 无 | config.json + JSONL 收件箱 |
| 线程 | 后台命令 | 每线程完整 Agent loop |
| 生命周期 | 一次性 | idle → working → idle |
| 通信 | 无 | message + broadcast |

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
