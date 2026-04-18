# ACP 交互循环：用户、Agent、工具的三方对话

> 一次 prompt turn 不只是「发消息、收回复」。用户发消息后，Agent 可能调用多个工具，每个工具可能要权限，权限拿到了才执行，执行完把结果喂回 LLM，LLM 再决定要不要继续调工具——这个循环可能转很多圈。

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)

---

## 为什么叫 "Turn"

在 ACP 里，一次完整的交互叫一个 **prompt turn**。它不是「一发一收」，而是：

```
用户说"重构这个函数"
  → Agent 思考，决定要先读文件
  → Agent 调用工具读文件
  → 读到了，喂回 LLM
  → LLM 决定还要跑测试
  → Agent 调用工具跑测试
  → 测试结果喂回 LLM
  → LLM 生成最终回复
  → Turn 结束
```

一个 turn 里，Agent 可能和 LLM 来回交互多次。**Turn 结束的唯一条件是 LLM 不再要求调用工具**（或者用户取消了）。

---

## Prompt Turn 生命周期

```
┌──────────────────────────────────────────────────┐
│                  Prompt Turn                      │
│                                                   │
│  1. 用户发消息 (session/prompt)                   │
│       │                                           │
│  2. Agent 处理（发给 LLM）                         │
│       │                                           │
│  3. Agent 汇报输出 (session/update)                │
│       │                                           │
│  4. 有工具调用？── 否 ──► 返回 stopReason，结束    │
│       │                                           │
│       是                                          │
│       │                                           │
│  5. 请求权限 → 执行工具 → 汇报结果                 │
│       │                                           │
│  6. 工具结果喂回 LLM ────► 回到步骤 2              │
│                                                   │
└──────────────────────────────────────────────────┘
```

每个步骤都有明确的协议方法。下面逐个拆解。

---

## 第 1 步：发送提示（session/prompt）

Client 把用户的消息发给 Agent：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_abc123",
    "prompt": [
      {
        "type": "text",
        "text": "帮我分析这段代码有没有问题"
      },
      {
        "type": "resource",
        "resource": {
          "uri": "file:///home/user/main.py",
          "mimeType": "text/x-python",
          "text": "def process(items):\n    for i in items:\n        print(i)"
        }
      }
    ]
  }
}
```

`prompt` 是一个数组，支持混合多种内容类型。用户的消息可能同时包含文字、代码、图片、文件引用。

⚠️ Client **必须**根据初始化时声明的 `promptCapabilities` 来限制内容类型。Agent 没声明 `image: true`，就别发图片。

---

## 第 2 步：Agent 处理

Agent 收到 prompt 后，把消息发给 LLM。这一步是 Agent 内部的，协议不关心 Agent 用什么模型、怎么调。

LLM 的输出可能是：
- **纯文本** —— 直接回复用户
- **工具调用** —— 需要执行某个工具才能继续
- **两者都有** —— 一边解释一边调工具

---

## 第 3 步：Agent 汇报输出

Agent 通过 `session/update` 通知实时推送 LLM 的输出。这是 ACP 最核心的流式机制——**不需要等 Agent 全部完成，用户就能看到中间过程**。

`session/update` 有四种类型：

| 类型 | `sessionUpdate` 值 | 说明 |
|------|-------------------|------|
| 执行计划 | `plan` | Agent 要做什么 |
| 文本流 | `agent_message_chunk` | Agent 的回复文字 |
| 工具调用 | `tool_call` | Agent 要调工具 |
| 工具更新 | `tool_call_update` | 工具执行进度 |

### 执行计划

Agent 告诉 Client 它打算怎么做：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "plan",
      "entries": [
        { "content": "分析代码结构", "priority": "high", "status": "pending" },
        { "content": "识别类型问题", "priority": "medium", "status": "pending" },
        { "content": "检查错误处理", "priority": "medium", "status": "pending" },
        { "content": "提出改进建议", "priority": "low", "status": "pending" }
      ]
    }
  }
}
```

计划可以动态更新。Agent 执行过程中发现新需求，可以发新的 plan 替换旧的。每次发送**完整的计划列表**，Client 直接替换。

### 文本流

Agent 逐块推送回复文字：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": {
        "type": "text",
        "text": "我来分析这段代码..."
      }
    }
  }
}
```

Agent 可以发多个 chunk，Client 拼接显示。这就是「打字机效果」的底层机制。

### 工具调用

LLM 决定调工具时，Agent 立刻通知 Client：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "tool_call",
      "toolCallId": "call_001",
      "title": "读取配置文件",
      "kind": "read",
      "status": "pending"
    }
  }
}
```

Client 收到后，可以立即在 UI 上显示「正在读取配置文件...」。

### 工具更新

工具执行过程中，Agent 推送进度：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "tool_call_update",
      "toolCallId": "call_001",
      "status": "in_progress"
    }
  }
}
```

工具完成后，推送最终结果：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "tool_call_update",
      "toolCallId": "call_001",
      "status": "completed",
      "content": [
        {
          "type": "content",
          "content": {
            "type": "text",
            "text": "找到 3 个配置文件..."
          }
        }
      ]
    }
  }
}
```

---

## 第 4 步：检查是否完成

每次 LLM 回复后，Agent 检查：还有没有未完成的工具调用？

```
没有工具调用 → Turn 结束，返回 stopReason
有工具调用 → 继续第 5 步
```

Turn 结束时，Agent 响应最初的 `session/prompt` 请求：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "stopReason": "end_turn"
  }
}
```

这个响应标志着 Turn 结束。注意：`session/prompt` 是一个请求，它的响应可能很久才来——中间 Agent 会发很多 `session/update` 通知。

---

## 第 5 步：工具执行（详见第 4 篇）

简要流程：

```
Agent 请求权限 (session/request_permission)
  → Client 展示给用户
  → 用户选择（允许/拒绝）
  → Agent 执行工具
  → Agent 汇报结果 (session/update)
```

权限和工具细节在[第 4 篇](acp-04-tools-permissions-and-resources-2026-04-18.md)展开。

---

## 第 6 步：继续对话

工具结果喂回 LLM，回到第 2 步。循环直到 LLM 不再要求调用工具。

```
第 1 轮 LLM 调用：LLM 说"我要读文件"
  → 工具执行 → 结果喂回 LLM
第 2 轮 LLM 调用：LLM 说"我要跑测试"
  → 工具执行 → 结果喂回 LLM
第 3 轮 LLM 调用：LLM 说"分析完成，结论是..."
  → 没有工具调用了 → Turn 结束
```

---

## 内容类型（Content Blocks）

`session/prompt` 的 `prompt` 数组和 `session/update` 的内容都使用 **ContentBlock** 结构。ACP 复用了 MCP 的 ContentBlock——Agent 可以直接转发 MCP 工具输出，不用转换。

### 五种内容类型

| 类型 | `type` 值 | 用途 | prompt 中需要能力？ |
|------|----------|------|-------------------|
| 纯文本 | `text` | 文字消息 | 基线必须支持 |
| 图片 | `image` | 截图、图表分析 | `image` capability |
| 音频 | `audio` | 语音转文字 | `audio` capability |
| 嵌入资源 | `resource` | 文件内容直接嵌入 | `embeddedContext` capability |
| 资源引用 | `resource_link` | 只给链接，Agent 自己读 | 基线必须支持 |

### text —— 纯文本

最基础的类型，所有 Agent 必须支持：

```json
{ "type": "text", "text": "帮我看看这段代码" }
```

### image —— 图片

Base64 编码的图片。需要 Agent 声明 `image` 能力：

```json
{
  "type": "image",
  "mimeType": "image/png",
  "data": "iVBORw0KGgoAAAANSUhEUg..."   // Base64 编码
}
```

### audio —— 音频

Base64 编码的音频。需要 `audio` 能力：

```json
{
  "type": "audio",
  "mimeType": "audio/wav",
  "data": "UklGRiQAAABXQVZFZm10..."     // Base64 编码
}
```

### resource —— 嵌入资源

把文件内容直接塞进消息里。最常见的场景是 `@` 引用：

```
用户在编辑器里输入：帮我重构 @main.py
```

Client 把 `main.py` 的内容嵌入 prompt：

```json
{
  "type": "resource",
  "resource": {
    "uri": "file:///home/user/main.py",
    "mimeType": "text/x-python",
    "text": "def process(items):\n    for i in items:\n        print(i)"
  }
}
```

为什么不用 `resource_link`？因为 Agent 可能**访问不到**那个文件。嵌入内容确保 Agent 一定能拿到。

### resource_link —— 资源引用

只给链接，Agent 自己决定要不要读：

```json
{
  "type": "resource_link",
  "uri": "file:///home/user/doc.pdf",
  "name": "doc.pdf",
  "mimeType": "application/pdf",
  "size": 1024000
}
```

适合大文件——不用把整个文件内容塞进消息。

### 选择哪种类型

```
文件小、Agent 可能访问不到 → resource（嵌入内容）
文件大、Agent 能访问 → resource_link（只给链接）
截图、图表 → image
```

---

## Agent Plan（执行计划）

Plan 是 Agent 向 Client 汇报的执行策略。不是每个 turn 都有 plan，但复杂任务通常会有。

### 计划条目

每个条目有三个字段：

| 字段 | 值 | 说明 |
|------|-----|------|
| `content` | 字符串 | 这个步骤要做什么 |
| `priority` | `high` / `medium` / `low` | 优先级 |
| `status` | `pending` / `in_progress` / `completed` | 当前状态 |

### 动态更新

Plan 不是一成不变的。Agent 可以随时发新的 plan 替换旧的：

```
初始计划：
  1. 分析代码结构     (high, pending)
  2. 识别问题         (high, pending)
  3. 提出建议         (low, pending)

执行中更新：
  1. 分析代码结构     (high, completed)    ← 完成了
  2. 识别问题         (high, in_progress)  ← 进行中
  3. 提出建议         (low, pending)
  4. 写单元测试       (medium, pending)    ← 新增的

再次更新：
  1. 分析代码结构     (high, completed)
  2. 识别问题         (high, completed)
  3. 提出建议         (low, completed)
  4. 写单元测试       (medium, in_progress)
```

⚠️ 每次发 plan 都要发**完整列表**。Client 直接替换，不合并。别只发变化的条目。

---

## StopReason

Turn 结束时，Agent 必须告诉 Client 为什么结束：

| StopReason | 含义 | 谁触发的 |
|------------|------|---------|
| `end_turn` | LLM 回复完毕，没有更多工具调用 | Agent（LLM 决定） |
| `max_tokens` | Token 上限到了 | Agent（配置限制） |
| `max_turn_requests` | 单轮 LLM 请求次数超限 | Agent（配置限制） |
| `refusal` | Agent 拒绝继续 | Agent（策略决定） |
| `cancelled` | 用户取消了 | Client（用户操作） |

Agent 可以在**任何时刻**停止 turn，不只是 `end_turn`。比如发现需要用户确认，可以先返回 `end_turn`，等用户确认后再开新的 turn。

---

## 取消（session/cancel）

用户可以随时取消正在进行的 turn：

```json
// Client → Agent
{
  "jsonrpc": "2.0",
  "method": "session/cancel",          // notification，没有 id
  "params": {
    "sessionId": "sess_abc123"
  }
}
```

取消时的清理规则：

| 谁做什么 | 规则 |
|---------|------|
| **Client** | 发出 cancel 后，立刻把所有未完成的工具调用标记为 `cancelled` |
| **Client** | 所有未回复的 `request_permission` 请求返回 `cancelled` |
| **Agent** | 收到 cancel 后，尽快停止所有 LLM 请求和工具调用 |
| **Agent** | 停完后，响应原始的 `session/prompt`，`stopReason` 为 `cancelled` |
| **Agent** | 可以在 cancel 后再发几个 `session/update`（但必须在响应 prompt 之前） |

```
时间线：

Client 发 session/prompt (id=2)
Agent 发 session/update × N
Client 发 session/cancel                  ← 用户点取消
Client 标记所有工具调用为 cancelled
Client 回复所有 permission 请求为 cancelled
Agent 停止处理
Agent 发最后的 session/update（可选）
Agent 回复 session/prompt (id=2, stopReason="cancelled")  ← turn 正式结束
```

⚠️ `session/cancel` 是 notification，没有 `id`，Agent 不会回复它。Agent 通过回复原始的 `session/prompt` 来确认取消完成。

---

## 实战：一个完整的 Turn 追踪

用户说"帮我检查 main.py 有没有 bug"：

```
1. Client → Agent: session/prompt
   "帮我检查 main.py 有没有 bug"

2. Agent → Client: session/update (plan)
   [分析代码结构(high), 检查错误处理(medium), 提出建议(low)]

3. Agent → Client: session/update (agent_message_chunk)
   "我来检查这个文件..."

4. Agent → Client: session/update (tool_call)
   "读取 main.py" (call_001, kind=read, status=pending)

5. Agent → Client: session/update (tool_call_update)
   call_001: status=in_progress

6. Agent → Client: session/update (tool_call_update)
   call_001: status=completed, content="文件内容..."

7. Agent → Client: session/update (plan)
   [分析代码结构(completed), 检查错误处理(in_progress), 提出建议(pending)]

8. Agent → Client: session/update (agent_message_chunk)
   "发现两个问题：1) 缺少空列表检查 2) 没有类型标注"

9. Agent → Client: session/prompt response
   { "stopReason": "end_turn" }
```

这个 turn 里只有一个工具调用。复杂的 turn 可能包含 5-10 个工具调用，LLM 来回交互多次。

---

## 速查表

### session/update 类型

| sessionUpdate | 方向 | 说明 |
|---------------|------|------|
| `plan` | Agent → Client | 执行计划 |
| `agent_message_chunk` | Agent → Client | 文本输出 |
| `tool_call` | Agent → Client | 新的工具调用 |
| `tool_call_update` | Agent → Client | 工具进度更新 |
| `user_message_chunk` | Agent → Client | 重放历史（session/load） |
| `available_commands_update` | Agent → Client | 斜杠命令更新 |
| `current_mode_update` | Agent → Client | 模式切换通知 |
| `config_option_update` | Agent → Client | 配置变更通知 |

### ContentBlock 类型

| type | prompt 中需要能力 | 包含字段 |
|------|-----------------|---------|
| `text` | 无 | `text` |
| `image` | `image` | `data`, `mimeType` |
| `audio` | `audio` | `data`, `mimeType` |
| `resource` | `embeddedContext` | `resource.uri`, `resource.text` |
| `resource_link` | 无 | `uri`, `name`, `mimeType`, `size` |

### StopReason

| 值 | 含义 |
|----|------|
| `end_turn` | 正常结束 |
| `max_tokens` | Token 超限 |
| `max_turn_requests` | LLM 调用次数超限 |
| `refusal` | Agent 拒绝 |
| `cancelled` | 用户取消 |

---

## 参考资料

- Prompt Turn 规范：https://agentclientprotocol.com/protocol/prompt-turn
- Content Blocks 规范：https://agentclientprotocol.com/protocol/content
- Agent Plan 规范：https://agentclientprotocol.com/protocol/agent-plan

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)
