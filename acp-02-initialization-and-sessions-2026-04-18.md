# ACP 初始化握手：Agent 和编辑器的 "第一次对话"

> 连接建立后不能直接发消息。先握手——协商版本、交换能力、确认身份。握手失败就别继续了，版本不对什么都白搭。

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)

---

## 为什么需要握手

Agent 和 Client 可能由不同团队开发，版本不同、能力不同。不先对齐就开干，会出现：

- **Agent 发了 `terminal/create`，但 Client 根本不支持终端** → 请求失败
- **Client 发了 `session/load`，但 Agent 不支持恢复会话** → 方法不存在
- **Client 用的是协议 v2，Agent 只支持 v1** → 消息格式对不上

握手解决这些问题：双方先说清楚「我支持什么」，再开始工作。

---

## 握手流程

```
Client                                     Agent
  │                                          │
  │  ── initialize (版本+能力+身份) ──────►  │
  │                                          │
  │  ◄── initialize response ────────────    │
  │      (选定版本+能力+身份+认证方式)         │
  │                                          │
  │  版本和能力都 OK？                        │
  │    ├── 是 → 创建会话                      │
  │    └── 否 → 断开连接，提示用户             │
```

握手只有一步：Client 调用 `initialize`，Agent 响应。但这一步里包含了大量信息。

---

## initialize 详解

### Client 发送

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": 1,                   // Client 支持的最新版本
    "clientCapabilities": {
      "fs": {
        "readTextFile": true,               // 支持读文件
        "writeTextFile": true               // 支持写文件
      },
      "terminal": true                      // 支持终端操作
    },
    "clientInfo": {
      "name": "zed",                        // 程序化标识
      "title": "Zed Editor",               // 给用户看的名字
      "version": "1.0.0"                    // 版本号
    }
  }
}
```

### Agent 响应

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": 1,                   // 最终使用的版本
    "agentCapabilities": {
      "loadSession": true,                  // 支持恢复会话
      "promptCapabilities": {
        "image": true,                      // 支持图片输入
        "audio": false,                     // 不支持音频
        "embeddedContext": true             // 支持嵌入资源
      },
      "mcpCapabilities": {
        "http": true,                       // MCP 支持 HTTP 传输
        "sse": false                        // 不支持 SSE
      }
    },
    "agentInfo": {
      "name": "claude-code",
      "title": "Claude Code",
      "version": "1.0.0"
    },
    "authMethods": []                       // 认证方式（空=不需要认证）
  }
}
```

每个字段都有明确用途：

| 字段 | 谁发 | 目的 |
|------|------|------|
| `protocolVersion` | 双方 | 确保协议兼容 |
| `clientCapabilities` | Client | 告诉 Agent 你能提供什么 |
| `agentCapabilities` | Agent | 告诉 Client 你支持什么 |
| `clientInfo` / `agentInfo` | 双方 | 身份标识，用于 UI 展示和调试 |
| `authMethods` | Agent | 告诉 Client 需要什么认证 |

---

## 版本协商

ACP 的版本号是一个**整数**，没有 minor 和 patch。只有引入 breaking change 才会递增。

```
Client 支持 v2, Agent 支持 v1~v2
  → Agent 响应 v2, 双方用 v2 通信 ✅

Client 支持 v2, Agent 只支持 v1
  → Agent 响应 v1
  → Client 发现 v1 自己能接受 → 用 v1 ✅
  → Client 发现 v1 自己不支持 → 断开连接，提示用户 ⚠️

Client 支持 v1, Agent 支持 v2
  → Agent 响应 v2（Agent 最新版本）
  → Client 发现 v2 不支持 → 断开连接 ⚠️
```

规则很简单：

1. Client 发自己支持的**最新**版本
2. Agent 如果支持，原样返回；不支持，返回自己最新的版本
3. Client 拿到响应后检查版本——不支持就断开，别硬上

⚠️ 新功能不算 breaking change。新能力通过 capabilities 声明，不是通过版本号。省略的能力 = 不支持。

---

## 能力交换

握手最核心的部分是能力交换。双方说清楚「我能干什么」，后面才知道该用什么方法。

### Client 能力

| 能力 | 字段 | 说明 |
|------|------|------|
| **文件读取** | `fs.readTextFile` | Agent 能通过 Client 读文件 |
| **文件写入** | `fs.writeTextFile` | Agent 能通过 Client 写文件 |
| **终端** | `terminal` | Agent 能通过 Client 执行命令 |

Client 的能力是**资源类**的——告诉 Agent 「我能给你什么资源」。

### Agent 能力

| 能力 | 字段 | 默认值 | 说明 |
|------|------|--------|------|
| **会话恢复** | `loadSession` | false | 支持加载历史会话 |
| **图片输入** | `promptCapabilities.image` | false | prompt 中能发图片 |
| **音频输入** | `promptCapabilities.audio` | false | prompt 中能发音频 |
| **嵌入资源** | `promptCapabilities.embeddedContext` | false | prompt 中能嵌文件内容 |
| **MCP HTTP** | `mcpCapabilities.http` | false | MCP Server 能用 HTTP |
| **MCP SSE** | `mcpCapabilities.sse` | false | MCP Server 能用 SSE |

Agent 的能力是**功能类**的——告诉 Client 「我能处理什么样的输入」。

### 基线能力 vs 可选能力

有些能力是所有实现**必须**支持的：

| 基线（必须） | 可选（需要声明） |
|-------------|-----------------|
| `session/new` | `session/load` |
| `session/prompt` | `promptCapabilities.image` |
| `session/cancel` | `promptCapabilities.audio` |
| `session/update` | `mcpCapabilities.http` |
| `ContentBlock::Text` | `fs.readTextFile` |
| `ContentBlock::ResourceLink` | `terminal` |

```
❌ 错误：假设对方支持所有能力

Agent 直接调用 terminal/create，没检查 Client 是否声明了 terminal 能力
→ 请求失败，Agent 崩溃

✅ 正确：先检查能力再调用

// initialize 响应中 clientCapabilities.terminal === true ?
// 是 → 可以调用 terminal/*
// 否 → 不能调用，改用其他方式
```

---

## 实现信息

`clientInfo` 和 `agentInfo` 三个字段：

| 字段 | 用途 | 例子 |
|------|------|------|
| `name` | 程序化标识，做逻辑判断 | `"claude-code"` |
| `title` | 给用户看的，UI 展示用 | `"Claude Code"` |
| `version` | 调试和指标收集用 | `"1.0.0"` |

`name` 和 `title` 的区别：

- `name` 是给代码用的——做判断、做兼容性处理
- `title` 是给人用的——在 UI 上显示

`title` 没提供就用 `name` 显示。

---

## 会话创建（session/new）

握手完成后，Client 创建会话才能开始交互。

```json
// Client → Agent
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "session/new",
  "params": {
    "cwd": "/home/user/project",           // 工作目录（必须绝对路径）
    "mcpServers": [                         // MCP Server 配置
      {
        "name": "filesystem",               // 人类可读的名字
        "command": "/path/to/mcp-server",   // 可执行文件路径
        "args": ["--stdio"],                // 命令行参数
        "env": []                           // 环境变量
      }
    ]
  }
}

// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "sessionId": "sess_abc123def456"        // 会话唯一 ID
  }
}
```

`sessionId` 是后续所有操作的凭证。发 prompt、取消操作、加载会话都用它。

### 工作目录（cwd）

`cwd` 有三条规则：

1. **必须是绝对路径**
2. 不管 Agent 子进程从哪里启动，都用这个目录
3. Agent 应该把它作为文件操作的边界

```
❌ 错误：用相对路径
"cwd": "./project"

✅ 正确：用绝对路径
"cwd": "/home/user/project"
```

---

## MCP Server 配置

Agent 处理用户请求时，可能需要外部工具。Client 在创建会话时告诉 Agent 可以连接哪些 MCP Server。

### 三种传输方式

| 传输 | 必须支持？ | 配置字段 |
|------|-----------|---------|
| **stdio** | 是 | `command` + `args` + `env` |
| **HTTP** | 否（需声明） | `type:"http"` + `url` + `headers` |
| **SSE** | 否（已弃用） | `type:"sse"` + `url` + `headers` |

### stdio 配置

最常用。Agent 启动 MCP Server 作为子进程：

```json
{
  "name": "filesystem",
  "command": "/usr/local/bin/mcp-server-fs",
  "args": ["--stdio", "--root", "/home/user"],
  "env": [
    { "name": "API_KEY", "value": "secret123" }
  ]
}
```

### HTTP 配置

Agent 需要声明 `mcpCapabilities.http: true` 才能用：

```json
{
  "type": "http",
  "name": "api-server",
  "url": "https://api.example.com/mcp",
  "headers": [
    { "name": "Authorization", "value": "Bearer token123" }
  ]
}
```

### SSE 配置

已被 MCP 规范弃用，但 ACP 仍支持。需要 `mcpCapabilities.sse: true`：

```json
{
  "type": "sse",
  "name": "event-stream",
  "url": "https://events.example.com/mcp",
  "headers": [
    { "name": "X-API-Key", "value": "apikey456" }
  ]
}
```

⚠️ 新项目应该用 HTTP 传输，别用 SSE。SSE 已被 MCP 规范弃用。

### 检查传输支持

```
// 初始化时检查
agentCapabilities.mcpCapabilities.http === true ?
  → 可以配置 HTTP 类型的 MCP Server
  → 不行，只能用 stdio
```

---

## 恢复会话（session/load）

如果 Agent 声明了 `loadSession: true`，Client 可以恢复之前的对话。

### 检查是否支持

```json
// initialize 响应中
{
  "agentCapabilities": {
    "loadSession": true    // ← 有这个才能调 session/load
  }
}
```

`loadSession` 是 `false` 或没出现 → **不能**调 `session/load`。

### 加载会话

```json
// Client → Agent
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "session/load",
  "params": {
    "sessionId": "sess_old_conversation",  // 要恢复的会话 ID
    "cwd": "/home/user/project",
    "mcpServers": [...]                    // 重新配置 MCP Server
  }
}
```

### Agent 重放历史

Agent 收到 `session/load` 后，通过 `session/update` 通知**逐条重放**对话历史：

```json
// 重放用户消息
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_old_conversation",
    "update": {
      "sessionUpdate": "user_message_chunk",
      "content": { "type": "text", "text": "之前问的问题" }
    }
  }
}

// 重放 Agent 回复
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_old_conversation",
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": { "type": "text", "text": "之前的回答" }
    }
  }
}

// 全部重放完毕，响应 session/load 请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": null
}
```

重放完成后，Client 可以像正常会话一样继续发 prompt。

---

## 实战：完整的初始化流程

把握手和会话创建串起来，用伪代码展示整个流程：

```python
# 1. 启动 Agent 子进程
agent = spawn("claude-code --stdio")

# 2. 握手：发送 initialize
result = agent.request("initialize", {
    "protocolVersion": 1,
    "clientCapabilities": {
        "fs": {"readTextFile": True, "writeTextFile": True},
        "terminal": True
    },
    "clientInfo": {"name": "my-editor", "version": "1.0.0"}
})

# 3. 检查版本
if result["protocolVersion"] != expected_version:
    warn_user("Agent 版本不兼容")
    agent.close()
    return

# 4. 记录 Agent 能力
agent_caps = result["agentCapabilities"]
supports_load = agent_caps.get("loadSession", False)
supports_image = agent_caps.get("promptCapabilities", {}).get("image", False)

# 5. 创建会话
session = agent.request("session/new", {
    "cwd": "/home/user/project",
    "mcpServers": [
        {"name": "fs", "command": "mcp-server-fs", "args": ["--stdio"]}
    ]
})
session_id = session["sessionId"]

# 6. 现在可以交互了
print(f"会话已创建: {session_id}")
print(f"支持恢复会话: {supports_load}")
print(f"支持图片输入: {supports_image}")
```

---

## 速查表

### 初始化参数

| 参数 | 必须？ | 说明 |
|------|--------|------|
| `protocolVersion` | 是 | Client 支持的最新版本 |
| `clientCapabilities` | 否 | Client 能提供的资源 |
| `clientInfo` | 否（建议） | Client 身份信息 |

### 初始化响应

| 字段 | 必须？ | 说明 |
|------|--------|------|
| `protocolVersion` | 是 | 最终使用的版本 |
| `agentCapabilities` | 否 | Agent 支持的功能 |
| `agentInfo` | 否（建议） | Agent 身份信息 |
| `authMethods` | 是 | 认证方式（空=不需要） |

### 会话创建参数

| 参数 | 必须？ | 说明 |
|------|--------|------|
| `cwd` | 是 | 工作目录（绝对路径） |
| `mcpServers` | 否 | MCP Server 列表 |

### MCP Server 传输

| 传输 | Agent 需要声明 | 关键字段 |
|------|---------------|---------|
| stdio | 无（默认支持） | `command`, `args`, `env` |
| HTTP | `mcpCapabilities.http` | `type:"http"`, `url`, `headers` |
| SSE | `mcpCapabilities.sse` | `type:"sse"`, `url`, `headers` |

---

## 参考资料

- 初始化规范：https://agentclientprotocol.com/protocol/initialization
- 会话管理规范：https://agentclientprotocol.com/protocol/session-setup
- JSON-RPC 2.0：https://www.jsonrpc.org/specification
- MCP 传输方式：https://modelcontextprotocol.io/docs/concepts/transports

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)
