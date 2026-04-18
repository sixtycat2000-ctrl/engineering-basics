# ACP 会话控制与扩展：模式、命令和自定义能力

> 协议是死的，场景是活的。同一个 Agent，有时候你想让它「只看不动手」，有时候你想让它「放手去干」。会话模式管权限，斜杠命令管快捷操作，扩展机制管「协议里没定义的东西」。

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)

---

## 为什么需要会话控制

Agent 的「工作模式」不是一成不变的。一个 Agent 可能同时支持：

- **Ask 模式** —— 只回答问题，不改代码。适合咨询
- **Architect 模式** —— 做设计和规划，不写代码。适合架构讨论
- **Code 模式** —— 完整权限，直接写代码。适合执行

没有会话控制，Agent 要么什么都问权限（太烦），要么什么都不问（太危险）。模式让用户按需调节。

---

## Session Modes

### 三种典型模式

| 模式 | 说明 | 权限行为 |
|------|------|---------|
| **Ask** | 只回答问题 | 任何修改前先请求权限 |
| **Architect** | 设计和规划 | 不写代码，只做方案 |
| **Code** | 直接写代码 | 完整工具权限 |

不同模式影响三个东西：**系统提示词**（Agent 怎么想）、**工具可用性**（Agent 能用什么）、**权限行为**（要不要问用户）。

### 初始化时声明模式

`session/new` 的响应中，Agent 可以返回可用模式：

```json
{
  "result": {
    "sessionId": "sess_abc123",
    "modes": {
      "currentModeId": "ask",               // 当前模式
      "availableModes": [
        {
          "id": "ask",
          "name": "Ask",
          "description": "任何修改前先请求权限"
        },
        {
          "id": "architect",
          "name": "Architect",
          "description": "只做设计和规划，不写代码"
        },
        {
          "id": "code",
          "name": "Code",
          "description": "完整工具权限，直接写代码"
        }
      ]
    }
  }
}
```

### 切换模式

模式可以随时切换，Agent 在工作也没关系。

**Client 端切换**（用户主动选）：

```json
// Client → Agent
{
  "method": "session/set_mode",
  "params": {
    "sessionId": "sess_abc123",
    "modeId": "code"                        // 切到 Code 模式
  }
}
```

**Agent 端切换**（Agent 自己决定）：

```json
// Agent → Client（notification）
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "current_mode_update",
      "modeId": "code"
    }
  }
}
```

### 从规划模式退出

一个常见场景：Agent 在 Architect 模式做完规划，想切换到 Code 模式执行。

Agent 通过 `request_permission` 请求用户确认切换：

```json
{
  "method": "session/request_permission",
  "params": {
    "sessionId": "sess_abc123",
    "toolCall": {
      "toolCallId": "call_switch_001",
      "title": "准备开始实现",
      "kind": "switch_mode",
      "status": "pending",
      "content": [
        {
          "type": "text",
          "text": "## 实施计划\n1. 重构入口文件\n2. 添加类型标注\n3. 写单元测试"
        }
      ]
    },
    "options": [
      {
        "optionId": "code",
        "name": "开始实现，自动接受所有操作",
        "kind": "allow_always"
      },
      {
        "optionId": "ask",
        "name": "开始实现，手动确认每一步",
        "kind": "allow_once"
      },
      {
        "optionId": "reject",
        "name": "不，继续规划",
        "kind": "reject_once"
      }
    ]
  }
}
```

用户选了 `code` 或 `ask` → Agent 切换模式，发 `current_mode_update` 通知。
用户选了 `reject` → Agent 继续在 Architect 模式。

---

## Session Config Options

Config Options 是比 Session Modes **更通用**的配置机制。不只是模式，还能配置模型、推理级别等任何选项。

### 为什么需要 Config Options

Session Modes 只能切换模式。但 Agent 可能还有其他可配置的东西：

- 用哪个模型？（快的小模型 vs 强的大模型）
- 推理深度？（浅层思考 vs 深度推理）
- 语言偏好？（中文 vs 英文）

Config Options 把这些统一成一个通用框架。

### 配置结构

`session/new` 的响应中，Agent 可以返回配置选项：

```json
{
  "result": {
    "sessionId": "sess_abc123",
    "configOptions": [
      {
        "id": "mode",
        "name": "工作模式",
        "description": "控制 Agent 的权限行为",
        "category": "mode",
        "type": "select",
        "currentValue": "ask",
        "options": [
          { "value": "ask", "name": "Ask", "description": "修改前先确认" },
          { "value": "code", "name": "Code", "description": "直接写代码" }
        ]
      },
      {
        "id": "model",
        "name": "模型",
        "category": "model",
        "type": "select",
        "currentValue": "model-1",
        "options": [
          { "value": "model-1", "name": "快速模型", "description": "响应最快" },
          { "value": "model-2", "name": "强力模型", "description": "能力最强" }
        ]
      }
    ]
  }
}
```

### 标准分类

| 分类 | 说明 |
|------|------|
| `mode` | 工作模式选择器 |
| `model` | 模型选择器 |
| `thought_level` | 思考/推理级别 |

分类帮 Client 做 UI——`mode` 类可以绑定快捷键，`model` 类可以放在特定位置。

⚠️ 以 `_` 开头的分类名是自定义的（如 `_my_category`），不以下划线开头的保留给 ACP 规范。

### 修改配置

**Client 端修改**：

```json
// Client → Agent
{
  "method": "session/set_config_option",
  "params": {
    "sessionId": "sess_abc123",
    "configId": "mode",                    // 选项 ID
    "value": "code"                        // 新值
  }
}

// Agent 响应（返回完整配置列表）
{
  "result": {
    "configOptions": [
      { "id": "mode", "currentValue": "code", ... },
      { "id": "model", "currentValue": "model-1", ... }
    ]
  }
}
```

**Agent 端修改**：

```json
// Agent → Client（notification）
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "config_option_update",
      "configOptions": [
        { "id": "mode", "currentValue": "code", ... },
        { "id": "model", "currentValue": "model-2", ... }  // 模型变了
      ]
    }
  }
}
```

Agent 主动修改配置的常见原因：
- 规划完切到 Code 模式
- 模型限流，自动降级到小模型
- 发现上下文太长，调整推理级别

### 优雅降级

Agent **必须**为每个配置项提供默认值。这样即使 Client 不支持 Config Options，Agent 也能正常工作。

```
Client 不支持 Config Options？
  → Agent 用默认值，功能正常 ✅

Client 不认识某个 option type？
  → Client 忽略该选项，Agent 用默认值 ✅

Client 没显示某些选项？
  → Agent 用默认值，不受影响 ✅
```

### 和 Session Modes 的关系

Config Options 是 Session Modes 的**升级版**。过渡期间，Agent 应该**同时发送两者**：

```json
{
  "result": {
    "sessionId": "sess_abc123",
    "modes": { ... },                    // 旧 API，给不支持 Config 的 Client
    "configOptions": [ ... ]             // 新 API，给支持的 Client
  }
}
```

规则：

| Client 支持 Config Options？ | 用哪个 |
|---------------------------|--------|
| 支持 | 用 `configOptions`，忽略 `modes` |
| 不支持 | 用 `modes` |

Agent 要保持两者同步，不管 Client 用哪个，行为都一样。

---

## 斜杠命令（Slash Commands）

Agent 可以注册快捷命令，用户输入 `/command` 直接触发。

### 注册命令

`session/new` 后，Agent 通过 `session/update` 注册命令：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "available_commands_update",
      "availableCommands": [
        {
          "name": "web",
          "description": "搜索网络信息",
          "input": { "hint": "搜索关键词" }
        },
        {
          "name": "test",
          "description": "运行项目测试"
        },
        {
          "name": "plan",
          "description": "创建详细实施计划",
          "input": { "hint": "计划描述" }
        }
      ]
    }
  }
}
```

| 字段 | 说明 |
|------|------|
| `name` | 命令名（不含 `/`） |
| `description` | 给用户看的说明 |
| `input.hint` | 输入框的占位提示（可选） |

### 动态更新

命令列表可以随时更新。Agent 可以根据上下文添加或删除命令：

```
项目里有 pytest → Agent 注册 /test 命令
项目里没有 pytest → Agent 不注册 /test 命令

Agent 发现了数据库配置 → 注册 /db-query 命令
Agent 发现没有数据库 → 移除 /db-query 命令
```

### 执行命令

用户输入 `/web agent client protocol`，Client 把它作为普通 prompt 发给 Agent：

```json
{
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_abc123",
    "prompt": [
      {
        "type": "text",
        "text": "/web agent client protocol"   // 原样发送
      }
    ]
  }
}
```

⚠️ Client **不解析**命令。它只是把 `/web ...` 作为文本发给 Agent，Agent 自己识别命令前缀并处理。

命令可以和图片、文件等混合发送：

```json
{
  "prompt": [
    { "type": "text", "text": "/analyze" },
    { "type": "image", "data": "...", "mimeType": "image/png" }
  ]
}
```

---

## 扩展机制

ACP 不可能预见所有需求。三种扩展机制让实现者加自定义功能，不破坏兼容性。

### 1. `_meta` 字段

所有协议类型都有一个 `_meta` 字段（`{ [key: string]: unknown }`）：

```json
{
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_abc123",
    "prompt": [...],
    "_meta": {
      "traceparent": "00-80e1afed...-7a085853722dc6d2-01",  // W3C Trace
      "zed.dev/debugMode": true                              // 自定义字段
    }
  }
}
```

三个保留字段（给 W3C Trace Context）：

| 字段 | 用途 |
|------|------|
| `traceparent` | Trace ID（兼容 OpenTelemetry） |
| `tracestate` | Trace 状态 |
| `baggage` | 上下文传播 |

⚠️ `_meta` 之外的根级别字段**禁止**自定义。所有字段名都保留给未来协议版本。

```
❌ 错误：在根级别加自定义字段

{
  "jsonrpc": "2.0",
  "method": "session/prompt",
  "myCustomField": true,     ← 不允许
  "params": { ... }
}

✅ 正确：自定义字段放 _meta

{
  "jsonrpc": "2.0",
  "method": "session/prompt",
  "params": {
    "_meta": { "myCustomField": true }   ← 放这里
  }
}
```

### 2. `_` 前缀自定义方法

以 `_` 开头的方法名保留给扩展。不会和未来协议方法冲突：

```json
// 自定义请求
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "_zed.dev/workspace/buffers",     // _ 前缀 = 自定义
  "params": { "language": "rust" }
}

// 响应
{
  "jsonrpc": "2.0",
  "id": 10,
  "result": {
    "buffers": [
      { "id": 0, "path": "/home/user/src/main.rs" },
      { "id": 1, "path": "/home/user/src/editor.rs" }
    ]
  }
}
```

不识别的自定义方法返回标准 JSON-RPC 错误：

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "error": {
    "code": -32601,                     // Method not found
    "message": "Method not found"
  }
}
```

自定义通知（没有 `id`）：

```json
{
  "jsonrpc": "2.0",
  "method": "_zed.dev/file_opened",
  "params": { "path": "/home/user/src/editor.rs" }
}
```

不识别的通知 → **应该忽略**，不报错。

```
❌ 不识别的自定义请求 → 返回 Method not found 错误
✅ 不识别的自定义通知 → 静默忽略
```

### 3. 能力声明

在 `initialize` 的能力对象中通过 `_meta` 声明扩展：

```json
{
  "agentCapabilities": {
    "loadSession": true,
    "_meta": {
      "zed.dev": {                    // 扩展命名空间
        "workspace": true,            // 支持工作区操作
        "fileNotifications": true     // 支持文件变更通知
      }
    }
  }
}
```

Client 调自定义方法前，先检查能力：

```python
# 检查 Agent 是否支持 zed.dev 扩展
caps = init_result["agentCapabilities"]
zed_caps = caps.get("_meta", {}).get("zed.dev", {})
if zed_caps.get("workspace"):
    # 可以调 _zed.dev/workspace/buffers
    pass
```

---

## 扩展的最佳实践

| 做法 | 原因 |
|------|------|
| 用域名做命名空间（如 `zed.dev`） | 避免冲突 |
| 先声明能力，再调自定义方法 | 别盲调 |
| 自定义通知要容忍被忽略 | 不是所有 Client 都支持 |
| `_meta` 里放元数据，不放核心数据 | 核心数据用标准字段 |

---

## 实战：构建一个自定义扩展

假设 Zed 想加一个「获取当前打开的文件列表」功能。

### 步骤 1：声明能力

```json
// Agent 在 initialize 响应中声明
{
  "agentCapabilities": {
    "_meta": {
      "zed.dev": {
        "workspace": true    // 我支持 workspace 扩展
      }
    }
  }
}
```

### 步骤 2：Client 检查能力后调用

```json
// Client → Agent
{
  "method": "_zed.dev/workspace/buffers",
  "params": { "language": "rust" }
}
```

### 步骤 3：Agent 响应

```json
{
  "result": {
    "buffers": [
      { "id": 0, "path": "/home/user/src/main.rs" },
      { "id": 1, "path": "/home/user/src/editor.rs" }
    ]
  }
}
```

如果 Agent 不支持这个扩展 → 返回 `-32601 Method not found`，Client 优雅降级。

---

## 速查表

### Session Modes API

| 操作 | 方法/通知 | 方向 |
|------|----------|------|
| 声明模式 | `session/new` 响应 | Agent → Client |
| Client 切换 | `session/set_mode` | Client → Agent |
| Agent 切换 | `session/update` (current_mode_update) | Agent → Client |

### Config Options API

| 操作 | 方法/通知 | 方向 |
|------|----------|------|
| 声明配置 | `session/new` 响应 | Agent → Client |
| Client 修改 | `session/set_config_option` | Client → Agent |
| Agent 修改 | `session/update` (config_option_update) | Agent → Client |

### Slash Commands API

| 操作 | 方法/通知 | 方向 |
|------|----------|------|
| 注册命令 | `session/update` (available_commands_update) | Agent → Client |
| 执行命令 | `session/prompt`（文本含 `/command`） | Client → Agent |
| 更新命令 | 再发一次 `available_commands_update` | Agent → Client |

### 扩展机制

| 机制 | 用途 | 例子 |
|------|------|------|
| `_meta` 字段 | 附加自定义元数据 | traceparent, debugMode |
| `_` 前缀方法 | 自定义方法/通知 | `_zed.dev/workspace/buffers` |
| 能力 `_meta` | 声明扩展支持 | `agentCapabilities._meta.zed.dev` |

### 保留规则

| 规则 | 说明 |
|------|------|
| 根级别不自定义 | 所有自定义放 `_meta` |
| `_` 方法名保留 | 不以 `_` 开头的方法名留给协议 |
| 不识别的请求 → 错误 | 返回 `-32601` |
| 不识别的通知 → 忽略 | 静默丢弃 |

---

## 参考资料

- Session Modes 规范：https://agentclientprotocol.com/protocol/session-modes
- Session Config Options 规范：https://agentclientprotocol.com/protocol/session-config-options
- Slash Commands 规范：https://agentclientprotocol.com/protocol/slash-commands
- Extensibility 规范：https://agentclientprotocol.com/protocol/extensibility

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)
