# Agent Client Protocol (ACP)：AI 编程的 "USB 接口"

> ACP 定义了 AI Agent 和编辑器/IDE 之间的标准通信协议。类似 LSP 统一了语言服务器集成，ACP 统一了 AI 编程助手的接入方式。

---

## 为什么需要 ACP

现在的 AI 编程工具生态有个大问题：

- **集成成本高** —— 每个编辑器要为每个 AI Agent 做定制集成，M 个编辑器 × N 个 Agent = M×N 套代码
- **兼容性差** —— Claude Code 只能跑在终端里，Cursor 用自己的 Agent，Zed 又是另一套
- **锁定用户** —— 选了一个 Agent 就只能用它支持的编辑器，反过来也一样

ACP 的解法和当年 LSP 一样：定义一套标准协议，Agent 实现一次，所有编辑器都能用；编辑器支持一次，所有 Agent 都能接入。M×N 变成 M+N。

---

## 整体架构

```
┌─────────────┐    stdio / HTTP     ┌─────────────┐    stdio / HTTP    ┌────────────┐
│   编辑器     │ ◄── JSON-RPC 2.0 ──► │  AI Agent   │ ◄── MCP 协议 ────► │ MCP Server │
│  (Client)   │                     │  (子进程)    │                    │ (工具/数据) │
└─────────────┘                     └─────────────┘                    └────────────┘
```

三个核心角色：

| 角色 | 说明 | 例子 |
|------|------|------|
| **Client** | 用户界面，通常是编辑器/IDE | Zed、JetBrains、Neovim、Emacs |
| **Agent** | AI 编程助手，通常作为 Client 的子进程运行 | Claude Code、Gemini CLI、Goose、Codex CLI |
| **MCP Server** | 外部工具和数据源 | 文件系统、数据库、搜索 API |

协议基于 **JSON-RPC 2.0**，两种消息类型：
- **Methods**：请求-响应模式，需要返回结果
- **Notifications**：单向通知，不期待响应

---

## 通信生命周期

一个完整的 ACP 交互分四步：

```
1. 初始化 (initialize)
   ├── 版本协商
   ├── 能力交换
   └── 认证（可选）

2. 创建会话 (session/new)
   ├── 设置工作目录
   └── 配置 MCP Server

3. 提示轮次 (session/prompt)
   ├── 用户发消息
   ├── Agent 处理 → 可能调用工具
   ├── 流式返回结果 (session/update)
   └── 循环直到完成

4. 取消/结束 (session/cancel)
```

---

## 第一步：初始化

连接建立后，Client **必须先**调用 `initialize`，协商版本和能力：

```json
// Client → Agent
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": 1,
    "clientCapabilities": {
      "fs": { "readTextFile": true, "writeTextFile": true },
      "terminal": true
    },
    "clientInfo": { "name": "zed", "version": "1.0.0" }
  }
}

// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "loadSession": true,
      "promptCapabilities": { "image": true, "audio": false, "embeddedContext": true },
      "mcpCapabilities": { "http": true, "sse": false }
    },
    "agentInfo": { "name": "claude-code", "version": "1.0.0" },
    "authMethods": []
  }
}
```

关键点：

| 能力 | Client 端 | Agent 端 |
|------|-----------|----------|
| **文件系统** | `fs.readTextFile` / `fs.writeTextFile` | — |
| **终端** | `terminal` | — |
| **会话加载** | — | `loadSession` |
| **富媒体输入** | — | `promptCapabilities.image/audio/embeddedContext` |
| **MCP 传输** | — | `mcpCapabilities.http/sse` |

版本是一个**整数**（只有 MAJOR 版本），只有引入 breaking change 才会递增。新功能通过 capabilities 通知，不算 breaking change。

---

## 第二步：会话管理

### 创建会话

```json
// Client → Agent
{
  "method": "session/new",
  "params": {
    "cwd": "/home/user/project",
    "mcpServers": [
      {
        "name": "filesystem",
        "command": "/path/to/mcp-server",
        "args": ["--stdio"],
        "env": []
      }
    ]
  }
}

// Agent → Client（返回唯一 Session ID）
{
  "result": { "sessionId": "sess_abc123" }
}
```

### 加载已有会话

如果 Agent 声明 `loadSession: true`，Client 可以恢复之前的对话：

```json
{
  "method": "session/load",
  "params": {
    "sessionId": "sess_old_one",
    "cwd": "/home/user/project",
    "mcpServers": [...]
  }
}
```

Agent 会通过 `session/update` 通知**重放整个对话历史**，重放完毕后响应 `session/load` 请求。

### MCP Server 连接

三种传输方式：

| 传输 | 必须支持？ | 说明 |
|------|-----------|------|
| **stdio** | 是 | 默认方式，Agent 启动子进程 |
| **HTTP** | 否 | Agent 能力声明 `mcpCapabilities.http` |
| **SSE** | 否 | 已被 MCP 规范弃用 |

---

## 第三步：提示轮次（Prompt Turn）

这是核心交互循环，一次 prompt turn 的完整流程：

```
用户发消息 → Agent 发给 LLM → LLM 返回文本/工具调用
                                      │
                    ┌─────────────────┘
                    ▼
            有工具调用？
              ├── 是 → 请求权限 → 执行工具 → 把结果喂回 LLM → 回到上一步
              └── 否 → 返回 stopReason，轮次结束
```

### 发送提示

```json
{
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_abc123",
    "prompt": [
      { "type": "text", "text": "帮我重构这个函数" },
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

### Agent 流式汇报

Agent 通过 `session/update` 通知实时推送：

**1. 计划（Plan）**
```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "plan",
      "entries": [
        { "content": "分析现有代码结构", "priority": "high", "status": "pending" },
        { "content": "识别需要重构的组件", "priority": "high", "status": "pending" },
        { "content": "编写单元测试", "priority": "medium", "status": "pending" }
      ]
    }
  }
}
```

**2. 文本输出**
```json
{
  "method": "session/update",
  "params": {
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": { "type": "text", "text": "我来帮你重构这个函数..." }
    }
  }
}
```

**3. 工具调用**
```json
{
  "method": "session/update",
  "params": {
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

### 权限请求

Agent 可以在执行工具前请求用户确认：

```json
// Agent → Client
{
  "method": "session/request_permission",
  "params": {
    "toolCall": { "toolCallId": "call_001" },
    "options": [
      { "optionId": "allow", "name": "允许一次", "kind": "allow_once" },
      { "optionId": "reject", "name": "拒绝", "kind": "reject_once" }
    ]
  }
}

// Client → Agent（用户选择）
{
  "result": { "outcome": { "outcome": "selected", "optionId": "allow" } }
}
```

### 结束原因

| StopReason | 含义 |
|------------|------|
| `end_turn` | LLM 回复完毕，没有更多工具调用 |
| `max_tokens` | Token 上限到了 |
| `max_turn_requests` | 单轮 LLM 请求次数超限 |
| `refusal` | Agent 拒绝继续 |
| `cancelled` | 用户取消 |

---

## 内容类型（Content Blocks）

ACP 复用 MCP 的 `ContentBlock` 结构，Agent 无需转换就能直接转发 MCP 工具的输出：

| 类型 | 说明 | 提示中需要能力？ |
|------|------|-----------------|
| `text` | 纯文本 | 基线必须支持 |
| `image` | Base64 图片 | `image` capability |
| `audio` | Base64 音频 | `audio` capability |
| `resource` | 嵌入资源（文件内容等） | `embeddedContext` capability |
| `resource_link` | 资源引用（Agent 自行读取） | 基线必须支持 |

---

## 工具调用详解

### 工具类型（ToolKind）

| Kind | 说明 |
|------|------|
| `read` | 读取文件或数据 |
| `edit` | 修改文件 |
| `delete` | 删除文件 |
| `move` | 移动/重命名 |
| `search` | 搜索 |
| `execute` | 运行命令 |
| `think` | 内部推理 |
| `fetch` | 获取外部数据 |
| `other` | 其他 |

### 工具状态流转

```
pending → in_progress → completed
                      → failed
```

### 工具内容类型

除了普通文本，工具调用还能产出：

**Diff（文件变更）**
```json
{
  "type": "diff",
  "path": "/home/user/config.json",
  "oldText": "{\n  \"debug\": false\n}",
  "newText": "{\n  \"debug\": true\n}"
}
```

**Terminal（终端输出）**
```json
{ "type": "terminal", "terminalId": "term_xyz789" }
```

### 跟随功能（Follow Along）

工具调用可以报告文件位置，Client 实现 "跟随" 效果：

```json
{ "path": "/home/user/src/main.py", "line": 42 }
```

用户可以实时看到 Agent 在读/改哪个文件的哪一行。

---

## 文件系统操作

Agent 通过 Client 的能力来读写文件，这比直接访问磁盘更有优势——可以读到**未保存的编辑器状态**：

```json
// 读取文件（支持行号范围）
{
  "method": "fs/read_text_file",
  "params": {
    "sessionId": "sess_abc123",
    "path": "/home/user/src/main.py",
    "line": 10,
    "limit": 50
  }
}

// 写入文件（不存在则创建）
{
  "method": "fs/write_text_file",
  "params": {
    "sessionId": "sess_abc123",
    "path": "/home/user/config.json",
    "content": "{\n  \"debug\": true\n}"
  }
}
```

---

## 终端操作

Agent 通过 Client 的终端能力执行命令，支持实时输出流和进程控制：

```json
// 创建终端
{
  "method": "terminal/create",
  "params": {
    "sessionId": "sess_abc123",
    "command": "npm",
    "args": ["test", "--coverage"],
    "env": [{ "name": "NODE_ENV", "value": "test" }],
    "cwd": "/home/user/project",
    "outputByteLimit": 1048576
  }
}
// 返回 terminalId，命令后台运行

// 等待退出
{ "method": "terminal/wait_for_exit", "params": { "terminalId": "term_xyz" } }

// 获取输出
{ "method": "terminal/output", "params": { "terminalId": "term_xyz" } }

// 杀死进程
{ "method": "terminal/kill", "params": { "terminalId": "term_xyz" } }

// 释放资源
{ "method": "terminal/release", "params": { "terminalId": "term_xyz" } }
```

终端可以嵌入到工具调用中，Client 实时显示输出。

### 实现超时

```python
# Agent 端实现命令超时的模式
terminal_id = terminal_create(command, timeout=30)
try:
    result = wait_for_exit_or_timeout(terminal_id, timeout=30)
except TimeoutError:
    terminal_kill(terminal_id)
    output = terminal_output(terminal_id)
finally:
    terminal_release(terminal_id)
```

---

## 会话模式（Session Modes）

Agent 可以提供多种工作模式，影响系统提示词、工具可用性和权限行为：

```json
{
  "modes": {
    "currentModeId": "ask",
    "availableModes": [
      { "id": "ask", "name": "Ask", "description": "任何修改前先请求权限" },
      { "id": "architect", "name": "Architect", "description": "只做设计和规划，不写代码" },
      { "id": "code", "name": "Code", "description": "完整工具权限，直接写代码" }
    ]
  }
}
```

Client 和 Agent 都可以切换模式。常见场景：Agent 在 Architect 模式规划完后，通过权限请求确认是否切换到 Code 模式执行。

---

## 斜杠命令（Slash Commands）

Agent 可以注册快捷命令：

```json
{
  "sessionUpdate": "available_commands_update",
  "availableCommands": [
    { "name": "web", "description": "搜索网络信息", "input": { "hint": "搜索关键词" } },
    { "name": "test", "description": "运行项目测试" },
    { "name": "plan", "description": "创建详细实施计划", "input": { "hint": "计划描述" } }
  ]
}
```

用户在编辑器里输入 `/web agent client protocol`，Client 把它作为普通 prompt 发给 Agent，Agent 识别命令前缀并处理。

---

## 可扩展性

ACP 内置三种扩展机制，不破坏兼容性：

### 1. `_meta` 字段

所有类型都有 `_meta` 字段（`{ [key: string]: unknown }`），附加自定义信息：

```json
{
  "method": "session/prompt",
  "params": {
    "prompt": [...],
    "_meta": {
      "traceparent": "00-80e1afed...-7a085853722dc6d2-01",
      "zed.dev/debugMode": true
    }
  }
}
```

`traceparent`、`tracestate`、`baggage` 保留给 W3C Trace Context，兼容 OpenTelemetry。

### 2. 下划线方法名

以 `_` 开头的方法名保留给自定义扩展：

```json
{ "method": "_zed.dev/workspace/buffers", "params": { "language": "rust" } }
```

不识别的扩展方法返回标准 JSON-RPC `Method not found` 错误。

### 3. 能力声明

在 `initialize` 的能力对象中通过 `_meta` 声明扩展：

```json
{
  "agentCapabilities": {
    "loadSession": true,
    "_meta": {
      "zed.dev": { "workspace": true, "fileNotifications": true }
    }
  }
}
```

---

## 传输层

| 传输 | 状态 | 说明 |
|------|------|------|
| **stdio** | 稳定 | 默认，Agent 作为 Client 子进程，stdin/stdout 传 JSON-RPC，stderr 写日志 |
| **Streamable HTTP** | 草案中 | 支持远程 Agent |
| **自定义** | 允许 | 只要保持 JSON-RPC 格式和生命周期要求 |

stdio 的规则很简单：
- 每条消息一行，`\n` 分隔
- **不能**在消息中嵌入换行
- Agent 的 stdout **只能**写 ACP 消息
- Agent 的 stderr 可以写日志，Client 可捕获或忽略

---

## 当前生态

### 已接入的 Agent（14+）

Augment Code、Claude Code（via Zed adapter）、Codex CLI、Docker cagent、Gemini CLI、Goose、Kimi CLI、OpenCode、OpenHands 等。

### 已接入的 Client（15+）

Zed、JetBrains、Neovim（CodeCompanion / avante.nvim）、Emacs（agent-shell.el）、Obsidian、DuckDB、marimo notebook 等。

### 官方 SDK

Kotlin、Python、Rust、TypeScript 四种语言。

---

## 和 LSP 的类比

| 维度 | LSP | ACP |
|------|-----|-----|
| 标准化了什么 | 编辑器 ↔ 语言服务器 | 编辑器 ↔ AI Agent |
| 基础协议 | JSON-RPC 2.0 | JSON-RPC 2.0 |
| 解决了什么 | M×N 集成变 M+N | M×N 集成变 M+N |
| 传输方式 | stdio / TCP | stdio / HTTP |
| 扩展机制 | 自定义请求/通知 | `_meta` 字段 + `_` 前缀方法 |

---

## 关键设计决策

1. **UX-first** —— 不是抽象到没用的通用协议，而是专门解决 AI 编程的 UX 问题（diff 展示、实时进度、权限控制）
2. **MCP-friendly** —— 复用 MCP 的 JSON 类型，Agent 可以无缝转发 MCP 工具输出
3. **Trusted model** —— 假设用户信任 Agent，Agent 有本地文件和 MCP 访问权，但保留工具调用的权限控制
4. **流式优先** —— 大量使用 JSON-RPC notification 实现 Agent 到 Client 的实时流式推送
5. **双向调用** —— 不只是 Client 调 Agent，Agent 也能向 Client 发请求（如请求权限、读写文件）

---

## 参考资料

- 官方文档：https://agentclientprotocol.com
- GitHub：https://github.com/AcpProtocol/acp-spec（通过 Zed Industries / JetBrains 推进）
- 背景阅读：LSP (Language Server Protocol)、MCP (Model Context Protocol)
