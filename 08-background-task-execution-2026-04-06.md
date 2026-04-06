# AI Background Tasks：后台执行

> `npm install` 跑 3 分钟，Agent 只能干等？不。后台线程跑命令，完成后通知注入。Agent 继续想下一步，不浪费一滴推理能力。

---

## 为什么需要后台执行

有些命令要跑好几分钟：`npm install`、`pytest --full-trace`、`docker build`、大数据集处理。阻塞式循环下，模型只能等——等命令跑完才能继续思考。

用户说"装依赖，顺便建个配置文件"。阻塞模式下 Agent 只能一个一个来：
1. 跑 `npm install`（等 3 分钟）
2. 建配置文件（30 秒）

浪费了 3 分钟的推理能力。Agent 本可以在等依赖安装的同时就把配置文件写好。

---

## 工作原理

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+-------- | enqueue(result) |
|  ↑drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      ← 并行执行
             |          |
             +-- results injected before next LLM call --+
```

### BackgroundManager：线程 + 通知队列

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

`_lock` 保护 `_notification_queue` 的线程安全。后台线程写，主线程读，不能同时操作。

### 启动后台任务

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

`daemon=True` 意味着主线程退出时后台线程自动终止。不会出现"主程序关了，后台进程还在跑"的情况。

### 后台执行 + 结果入队

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

300 秒超时是安全阀。后台命令不应该无限运行。

**注意截断 `[:500]`：通知队列里的结果只保留前 500 字符。** 为什么？因为通知要注入到上下文里，完整输出可能几千行。500 字符足够知道"成功了"还是"失败了"——如果需要详情，模型可以主动读取输出文件。

### 主循环排空通知

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
        response = client.messages.create(...)
```

**每次 LLM 调用前排空通知队列。** 后台任务的结果变成一条特殊的 user 消息，包裹在 `<background-results>` 标签里。

---

## 隐性知识

### 为什么是"队列 + 排空"而不是"回调"

LLM 是单线程的。它不能在思考到一半时被中断处理回调。通知必须等模型下一轮调用前注入。

这个设计是事件循环的经典模式：
- Node.js 的 event loop：异步 I/O 完成后，回调在下一次 tick 执行
- 浏览器的事件循环：网络请求完成后，回调在下一个 microtask 执行
- Agent loop：后台任务完成后，结果在下一次 LLM 调用前注入

**不能打断正在进行的推理，只能排队等下一轮。**

### 通知注入的时机很关键

注入发生在 `drain_notifications()` 和 `client.messages.create()` 之间——在模型思考之前、工具执行之后。这个顺序确保：
1. 模型先看到后台任务的结果
2. 然后决定下一步做什么（可能是继续等、可能是开始新的工作）

如果把注入放在模型调用之后，模型就得多等一轮才能看到结果。

### 为什么只并行化 I/O，不并行化推理

多个后台线程可以同时跑命令（I/O 并行），但 LLM 推理始终是单线程的——一次只有一个 LLM 调用在跑。

这不是技术限制，而是设计选择。如果允许两个 LLM 推理同时跑：
- 两个推理可能产生冲突的工具调用（同时改同一个文件）
- 两个推理的上下文不同步（A 看不到 B 刚刚的决策）
- 消息列表的并发修改会导致状态不一致

**单线程推理 + 多线程 I/O = 简单且安全。**

### daemon 线程的取舍

`daemon=True` 意味着主线程退出时后台任务会被杀死。在 Agent 场景下这是对的——用户按 Ctrl+C 退出时，不应该让 `npm install` 继续在后台跑。

但如果后台任务有副作用（比如写了一半的文件），强制终止可能留下不一致状态。这就是为什么后台任务通常应该是"读多写少"的操作（测试、构建、查询），而不是"改写"操作。

---

## 相对 s07 的变更

| 组件 | s07 | s08 |
|------|-----|-----|
| 工具 | 8 | 6（基础 + background_run + check） |
| 执行方式 | 仅阻塞 | 阻塞 + 后台线程 |
| 通知机制 | 无 | 每轮排空的队列 |
| 并发 | 无 | 守护线程 |

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
