# AI Team Protocols：团队协议

> 队友能通信了，但怎么协调？"我要关机了，你准备好了吗？""这个重构方案你同意吗？"request-response 模式，一个 FSM 驱动所有协商。

---

## 为什么需要协议

s09 中队友能通信，但缺少结构化协调。两个核心问题：

**关机问题**：领导说"alice，你下班了"，直接杀线程？alice 可能正在写文件，写了一半。config.json 还显示"working"，但线程已经没了。需要**握手**——领导请求，alice 收尾后确认退出。

**计划审批问题**：领导说"重构认证模块"，alice 立刻开干。如果重构方案有风险（改了 50 个文件），应该先过审再执行。

两者结构一样：**一方发请求（带 ID），另一方引用同一 ID 响应。**

---

## 协议模式

```
Shutdown Protocol              Plan Approval Protocol
==================             ======================

Lead             Teammate      Teammate           Lead
  |                 |             |                 |
  |--shutdown_req-->|             |--plan_req------>|
  | {req_id:"abc"}  |             | {req_id:"xyz"}  |
  |                 |             |                 |
  |<--shutdown_resp-|             |<--plan_resp-----|
  | {req_id:"abc",  |             | {req_id:"xyz",  |
  |  approve:true}  |             |  approve:true}  |

Shared FSM:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]
```

一个有限状态机（FSM），两种用途。`pending → approved | rejected` 的状态转换可以套用到任何请求-响应场景。

---

## 工作原理

### 关机协议

领导生成 `request_id`，通过收件箱发起请求：

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

队友收到后，用 approve/reject 响应（引用同一 `request_id`）：

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = \
        "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

如果 approve，领导更新 config.json 把队友标记为 shutdown。如果 reject，队友继续工作。

### 计划审批协议

完全相同的模式，方向可能相反（队友提交计划，领导审批）：

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

### request_id 是关联键

为什么需要 `request_id`？因为消息是异步的。考虑这个场景：

```
1. 领导发 shutdown_req (req_id: "abc") 给 alice
2. 领导发 shutdown_req (req_id: "def") 给 bob
3. alice 的响应到达——是响应 abc 还是 def？
```

没有 `request_id`，无法区分。有了它，`{req_id: "abc", approve: true}` 明确表示"alice 同意关机请求 abc"。

**这是分布式系统中 correlation ID 模式的直接应用。** 每个请求一个唯一 ID，响应用同一个 ID 关联。

---

## 隐性知识

### 为什么不直接杀线程

技术上完全可以 `thread.terminate()`。但：
- 线程可能在写文件——杀了之后文件可能损坏
- config.json 可能没更新——其他 Agent 以为该队友还在工作
- 队友可能有未完成的消息——发送方不知道消息没被处理

**优雅关机（graceful shutdown）的核心：给被关机方一个收尾的机会。** "你该下班了"和"直接把你的电脑拔了"是完全不同的。

### FSM 让协议状态可追踪

没有 FSM 时，"关机请求发出去了"和"关机已完成"之间没有显式状态。领导只能靠"读收件箱有没有回复"来判断——脆弱且不可靠。

有了 FSM：
- `shutdown_requests["abc"]["status"] == "pending"` → 等待中
- `shutdown_requests["abc"]["status"] == "approved"` → 已批准
- `shutdown_requests["abc"]["status"] == "rejected"` → 已拒绝

状态是可查询、可审计的。任何 Agent 都可以检查任何请求的当前状态。

### 同一个 FSM 可以复用

`pending → approved | rejected` 不只适用于关机和计划审批。任何需要双方同意的操作都能用：
- 资源请求："我需要访问数据库" → pending → approved/rejected
- 代码合并请求："我可以合并这个分支吗？" → pending → approved/rejected
- 权限升级："我需要 sudo 权限执行这条命令" → pending → approved/rejected

**一个模式，无限场景。** 这就是协议抽象的力量。

### 为什么不让模型"自然结束"

s09 的队友在 LLM 停止调用工具时自然进入 idle。为什么不直接用这个机制关机？

因为"自然结束"不等于"收到关机指令后结束"。区别在于：
- 自然结束：队友做完了当前任务，没有新任务，自己停了。但领导不知道它停了（直到读 config.json）
- 协议关机：领导**主动发起**，队友**确认响应**，双方都知道关机发生了

**协议关机是双向确认，自然结束是单方面行为。** 在协调场景中，双向确认比单方面行为可靠得多。

---

## 相对 s09 的变更

| 组件 | s09 | s10 |
|------|-----|-----|
| 工具 | 9 | 12（+shutdown_req/resp + plan） |
| 关机 | 仅自然退出 | 请求-响应握手 |
| 计划门控 | 无 | 提交/审查与审批 |
| 关联 | 无 | 每个请求一个 request_id |
| FSM | 无 | pending → approved/rejected |

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
