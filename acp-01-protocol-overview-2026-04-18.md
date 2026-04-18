# ACP 协议全景：AI 编程的 "USB 接口"

> 2015 年 LSP 统一了编辑器和语言服务器。2025 年 ACP 想做同一件事——统一编辑器和 AI Agent。一个协议，M+N 次集成替代 M×N 次定制。

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)

---

## 为什么需要 ACP

今天的 AI 编程工具生态有个致命问题：**每对编辑器+Agent 都要单独做集成**。

Claude Code 跑在终端里，Cursor 用自己的 Agent，Zed 又是另一套。如果你是一个 Agent 开发者，想让你的 Agent 在 Zed、JetBrains、Neovim、Emacs 里都能用——你得分别对接 5 个编辑器。反过来也一样。

```
❌ 没有 ACP：M × N 套集成代码

         Claude   Gemini   Codex   Goose
Zed       ✅       ❌       ✅       ❌     ← 每个 ✅ 都是一套定制代码
JetBrains  ❌       ✅       ❌       ✅
Neovim     ❌       ❌       ✅       ❌
Emacs      ✅       ❌       ❌       ❌

✅ 有了 ACP：M + N 次集成

Agent 实现 ACP 一次 → 所有编辑器都能用
编辑器实现 ACP 一次 → 所有 Agent 都能接
```

这和 2015 年 LSP 解决的问题一模一样。LSP 统一了编辑器与语言服务器，ACP 统一编辑器与 AI Agent。

---

## 三个核心角色

```
┌────────────┐  JSON-RPC 2.0  ┌────────────┐  MCP 协议  ┌────────────┐
│  Client    │ ◄──────────────►│   Agent    │◄──────────►│ MCP Server │
│ (编辑器)    │   stdio/HTTP   │ (AI 助手)   │  stdio/HTTP │ (工具/数据) │
└────────────┘                └────────────┘            └────────────┘
```

| 角色 | 干什么 | 例子 |
|------|--------|------|
| **Client** | 用户界面，管理 Agent 进程 | Zed、JetBrains、Neovim、Emacs |
| **Agent** | AI 编程助手，通常是 Client 的子进程 | Claude Code、Gemini CLI、Codex CLI、Goose |
| **MCP Server** | 外部工具和数据源 | 文件系统、数据库、搜索 API |

三个角色各有分工：

- **Client** 管环境——启动 Agent、显示输出、管权限、提供文件系统和终端
- **Agent** 管思考——把用户需求发给 LLM、执行工具调用、返回结果
- **MCP Server** 管工具——提供具体的能力（搜索、数据库查询等）

Agent 夹在中间，对 Client 用 ACP 通信，对 MCP Server 用 MCP 协议通信。

---

## 通信基础：JSON-RPC 2.0

ACP 的每条消息都是 JSON-RPC 2.0 格式。两种消息类型：

### Method（请求-响应）

```json
// Client → Agent：发请求
{
  "jsonrpc": "2.0",
  "id": 1,                        // 请求 ID，响应要带上
  "method": "session/new",
  "params": { "cwd": "/home/user/project" }
}

// Agent → Client：返回结果
{
  "jsonrpc": "2.0",
  "id": 1,                        // 对应请求的 ID
  "result": { "sessionId": "sess_abc123" }
}
```

### Notification（单向通知）

```json
// Agent → Client：通知，不需要响应
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": { "type": "text", "text": "我来帮你看看..." }
    }
  }
}
```

关键区别：**Method 有 `id`，Notification 没有**。Client 和 Agent 双方都能发这两种消息——ACP 是双向通信。

⚠️ ACP 不是简单的「Client 问 Agent 答」。Agent 也会向 Client 发请求（比如请求权限、读写文件）。这是一个**对等协议**。

---

## 通信生命周期

一个完整的 ACP 交互分四个阶段：

```
1. 初始化 (initialize)
   ├── 版本协商
   ├── 能力交换
   └── 实现信息

2. 会话创建 (session/new)
   ├── 设置工作目录
   └── 配置 MCP Server

3. 交互循环 (session/prompt)
   ├── 用户发消息
   ├── Agent 处理 → 可能调用工具
   ├── 流式返回 (session/update)
   └── 循环直到完成

4. 取消/结束 (session/cancel)
```

每个阶段有明确的规则。初始化必须在会话之前，会话必须在交互之前。跳过任何一步，协议不会工作。

各阶段对应的方法：

| 阶段 | Client 调用 | Agent 调用 |
|------|------------|------------|
| 初始化 | `initialize` | — |
| 会话 | `session/new`、`session/load` | — |
| 交互 | `session/prompt`、`session/cancel` | `session/update`、`session/request_permission` |
| 资源 | — | `fs/read_text_file`、`terminal/create` 等 |

---

## 传输层

ACP 消息通过传输层在 Client 和 Agent 之间传递。

| 传输 | 状态 | 说明 |
|------|------|------|
| **stdio** | 稳定 | 默认方式，Agent 作为子进程 |
| **Streamable HTTP** | 草案中 | 支持远程 Agent |
| **自定义** | 允许 | 保持 JSON-RPC 格式即可 |

### stdio 规则

stdio 是最常用的传输方式。Client 启动 Agent 作为子进程：

```bash
# Client 启动 Agent 子进程
/path/to/agent --stdio
# Client 的 stdin → Agent 的 stdin（ACP 消息）
# Agent 的 stdout → Client 的 stdout（ACP 消息）
# Agent 的 stderr → 日志输出（Client 可捕获或忽略）
```

三条硬规则：

1. 每条消息一行，`\n` 分隔
2. 消息中**不能**嵌入换行（JSON 要压缩成一行）
3. Agent 的 stdout **只能**写 ACP 消息，别的东西写 stderr

```
❌ 错误：Agent 在 stdout 写了调试日志

stdout: {"jsonrpc":"2.0","id":1,"result":{...}}
stdout: [DEBUG] Processing request...          ← 这会搞崩协议
stdout: {"jsonrpc":"2.0","method":"session/update","params":{...}}

✅ 正确：调试日志写 stderr

stdout: {"jsonrpc":"2.0","id":1,"result":{...}}
stderr: [DEBUG] Processing request...          ← 协议不受影响
stdout: {"jsonrpc":"2.0","method":"session/update","params":{...}}
```

---

## 双向调用的方法分布

ACP 不是「Client 调 Agent」的单向协议。两边都能发请求和通知。

### Agent 暴露的方法（Client 调用）

| 方法 | 必须？ | 说明 |
|------|--------|------|
| `initialize` | 是 | 初始化连接 |
| `session/new` | 是 | 创建新会话 |
| `session/prompt` | 是 | 发送用户消息 |
| `session/load` | 否 | 恢复已有会话 |
| `session/set_mode` | 否 | 切换工作模式 |

### Agent 发出的通知

| 通知 | 说明 |
|------|------|
| `session/update` | 流式推送：文本、工具调用、计划等 |
| `session/cancel` | 取消当前轮次 |

### Client 暴露的方法（Agent 调用）

| 方法 | 条件 | 说明 |
|------|------|------|
| `session/request_permission` | — | 请求用户确认工具调用 |
| `fs/read_text_file` | Client 声明 fs 能力 | 读文件（含未保存状态） |
| `fs/write_text_file` | Client 声明 fs 能力 | 写文件 |
| `terminal/create` | Client 声明 terminal 能力 | 创建终端 |
| `terminal/output` | 同上 | 获取终端输出 |
| `terminal/wait_for_exit` | 同上 | 等待命令结束 |
| `terminal/kill` | 同上 | 杀死进程 |
| `terminal/release` | 同上 | 释放资源 |

### Client 发出的通知

| 通知 | 说明 |
|------|------|
| `session/cancel` | 用户取消当前操作 |

---

## 当前生态

ACP 还很年轻（2025 年推出），但已经获得广泛支持。

### 已接入的 Agent

| Agent | 说明 |
|-------|------|
| Claude Code | Anthropic 的 CLI Agent |
| Gemini CLI | Google 的命令行 Agent |
| Codex CLI | OpenAI 的命令行 Agent |
| Goose | 开源 AI Agent |
| Augment Code | Augment 的编程助手 |
| OpenHands | 开源自主编程 Agent |
| Kimi CLI | 月之暗面的命令行 Agent |
| Docker cagent | Docker 的容器化 Agent |

### 已接入的 Client

| Client | 说明 |
|--------|------|
| Zed | 最早支持 ACP 的编辑器 |
| JetBrains | IntelliJ 等 IDE |
| Neovim | 通过 CodeCompanion / avante.nvim |
| Emacs | 通过 agent-shell.el |
| Obsidian | 笔记工具 |
| DuckDB | 数据库 |
| marimo | Python notebook |

### 官方 SDK

| 语言 | 说明 |
|------|------|
| TypeScript | 前端/Node.js |
| Python | 数据科学/后端 |
| Kotlin | Android/JVM |
| Rust | 系统编程 |

---

## ACP vs LSP：同一个思路，不同的问题域

| 维度 | LSP | ACP |
|------|-----|-----|
| 标准化了什么 | 编辑器 ↔ 语言服务器 | 编辑器 ↔ AI Agent |
| 基础协议 | JSON-RPC 2.0 | JSON-RPC 2.0 |
| 核心解法 | M×N 变 M+N | M×N 变 M+N |
| 传输方式 | stdio / TCP | stdio / HTTP |
| 双向通信 | 主要是 Client → Server | 双向对等 |
| 扩展机制 | 自定义请求/通知 | `_meta` + `_` 前缀方法 |
| 流式支持 | 有限 | 原生（大量 notification） |
| 权限控制 | 无 | `request_permission` |

最大的区别：**ACP 是对等的**。Agent 会向 Client 请求权限、读写文件、执行命令。LSP 中 Server 很少主动请求 Client 做事，但 ACP 中 Agent 是一个需要 Client 协助的「工作者」。

---

## 关键设计决策

理解 ACP 的设计理念，有助于正确实现：

1. **UX-first** —— 不是抽象的通用协议。每个方法都解决 AI 编程的具体 UX 问题（diff 展示、实时进度、权限控制）
2. **MCP-friendly** —— 复用 MCP 的 ContentBlock 结构，Agent 可以无缝转发 MCP 工具输出，不用转换格式
3. **流式优先** —— 大量使用 notification 实现实时推送（文本流、工具进度、计划更新）
4. **双向对等** —— Agent 也能向 Client 发请求（权限、文件、终端）
5. **渐进式能力** —— 基线功能所有实现必须支持，高级功能通过 capabilities 声明

---

## 速查表

### 协议基本规则

| 规则 | 说明 |
|------|------|
| 消息格式 | JSON-RPC 2.0，UTF-8 编码 |
| 文件路径 | 必须是绝对路径 |
| 行号 | 从 1 开始（1-based） |
| 版本号 | 整数，只有 MAJOR 版本 |
| 扩展 | `_meta` 字段 + `_` 前缀方法 |

### 生命周期速查

```
initialize → session/new → session/prompt ←→ session/update → 结束
                                         ↑          │
                                         └──────────┘
                                       (工具调用循环)
```

---

## 参考资料

- ACP 官方文档：https://agentclientprotocol.com
- ACP GitHub：https://github.com/AcpProtocol/acp-spec
- JSON-RPC 2.0 规范：https://www.jsonrpc.org/specification
- LSP（Language Server Protocol）：https://microsoft.github.io/language-server-protocol/
- MCP（Model Context Protocol）：https://modelcontextprotocol.io
- 快速概览：[Agent Client Protocol 5 分钟指南](agent-client-protocol-2026-04-06.md)

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)
