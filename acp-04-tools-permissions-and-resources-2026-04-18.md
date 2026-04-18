# ACP 工具系统：Agent 怎么读写文件、跑命令

> LLM 不是直接读写文件的。它说「我要读 /foo/bar.py」，Agent 把这句话变成一个工具调用，通过 ACP 协议让 Client 去读。为什么？因为文件在 Client 的编辑器里——可能有未保存的修改，Agent 自己读磁盘读不到。

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)

---

## 为什么工具调用需要协议

Agent 调用工具不是简单的函数调用。它面临三个问题：

1. **权限** —— Agent 要写文件、跑命令，但用户可能不想让它乱改
2. **展示** —— 用户想实时看到 Agent 在干什么（在读哪个文件、跑什么命令）
3. **资源** —— 文件和终端在 Client 那边，Agent 需要通过协议请求 Client 操作

ACP 的工具系统解决了这三个问题：通过 `session/update` 实时汇报进度，通过 `session/request_permission` 请求用户确认，通过 `fs/*` 和 `terminal/*` 方法借用 Client 的资源。

---

## 工具调用生命周期

一个工具调用从创建到完成：

```
1. 创建 (tool_call)
   "读取配置文件" (pending)

2. 请求权限 (request_permission) —— 可选
   "Agent 想读配置文件，允许？"

3. 开始执行 (tool_call_update)
   status: in_progress

4. 执行完成 (tool_call_update)
   status: completed, content: [...]
```

### 状态流转

```
pending → in_progress → completed
                      → failed
```

| 状态 | 含义 |
|------|------|
| `pending` | 等待执行（输入还在流式传输或等待权限） |
| `in_progress` | 正在执行 |
| `completed` | 成功完成 |
| `failed` | 执行失败 |

---

## 创建工具调用

LLM 决定调工具时，Agent 通过 `session/update` 创建工具调用：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "tool_call",
      "toolCallId": "call_001",           // 唯一 ID
      "title": "读取配置文件",              // 给用户看的标题
      "kind": "read",                      // 工具类型
      "status": "pending"
    }
  }
}
```

### ToolKind —— 工具分类

| Kind | 说明 | Client 可以 |
|------|------|------------|
| `read` | 读取文件/数据 | 显示文件图标 |
| `edit` | 修改文件 | 显示编辑图标 |
| `delete` | 删除文件 | 显示警告 |
| `move` | 移动/重命名 | 显示移动图标 |
| `search` | 搜索 | 显示搜索图标 |
| `execute` | 运行命令 | 显示终端图标 |
| `think` | 内部推理 | 显示思考气泡 |
| `fetch` | 获取外部数据 | 显示下载图标 |
| `other` | 其他 | 默认图标 |

`kind` 不是给 Agent 自己用的，是帮 Client **选择合适的图标和展示方式**。

---

## 更新工具调用

工具执行过程中，Agent 发送 `tool_call_update` 推送进度：

```json
// 开始执行
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

// 执行完成
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

更新只包含变化的字段。`toolCallId` 必须有，其他字段可选。

---

## 权限请求

Agent 可以在执行工具**之前**请求用户确认。这是 ACP 的安全机制。

### 请求权限

```json
// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "session/request_permission",
  "params": {
    "sessionId": "sess_abc123",
    "toolCall": {
      "toolCallId": "call_001",
      "title": "修改 config.json",
      "kind": "edit",
      "status": "pending"
    },
    "options": [
      {
        "optionId": "allow-once",
        "name": "允许一次",
        "kind": "allow_once"
      },
      {
        "optionId": "reject",
        "name": "拒绝",
        "kind": "reject_once"
      }
    ]
  }
}
```

### 用户选择

```json
// Client → Agent
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "outcome": {
      "outcome": "selected",            // 选择了某个选项
      "optionId": "allow-once"          // 选择的选项 ID
    }
  }
}
```

### 四种权限选项

| Kind | 含义 | 说明 |
|------|------|------|
| `allow_once` | 允许这一次 | 下次还要问 |
| `allow_always` | 永远允许 | 记住选择，以后不问 |
| `reject_once` | 拒绝这一次 | 下次可能还问 |
| `reject_always` | 永远拒绝 | 以后也不问 |

`kind` 是给 Client 的 UI 提示——`allow_always` 可以用绿色的「总是允许」按钮，`reject_always` 可以用红色的「总是拒绝」。

### 权限取消

如果用户在权限请求待回复时取消了 turn：

```json
{
  "result": {
    "outcome": {
      "outcome": "cancelled"           // turn 被取消了
    }
  }
}
```

Client **必须**把所有未回复的权限请求都返回 `cancelled`。

### 自动权限

Client 可以根据用户设置**自动**允许或拒绝，不弹 UI：

```
用户设置了 "总是允许读文件" → Agent 请求读文件权限
→ Client 直接返回 allow_always，不弹确认框
```

---

## 工具内容类型

工具调用完成后，可以产出不同类型的内容：

### 普通内容

文本、图片、资源等标准 ContentBlock：

```json
{
  "type": "content",
  "content": {
    "type": "text",
    "text": "分析完成：找到 3 个问题"
  }
}
```

### Diff（文件变更）

展示文件修改前后的差异：

```json
{
  "type": "diff",
  "path": "/home/user/config.json",
  "oldText": "{\n  \"debug\": false\n}",
  "newText": "{\n  \"debug\": true\n}"
}
```

Client 可以用这个渲染 diff 视图——高亮显示删除和新增的行。

| 字段 | 说明 |
|------|------|
| `path` | 被修改的文件（绝对路径） |
| `oldText` | 原始内容（新文件为 null） |
| `newText` | 修改后的内容 |

### Terminal（终端输出）

把终端嵌入工具调用：

```json
{
  "type": "terminal",
  "terminalId": "term_xyz789"
}
```

Client 会实时显示终端输出。终端释放后，Client **应该**继续显示已有输出。

---

## Follow Along（跟随功能）

工具调用可以报告文件位置，Client 实现"跟随"效果——用户看到 Agent 正在读/改哪个文件：

```json
{
  "path": "/home/user/src/main.py",
  "line": 42
}
```

Client 收到后，在编辑器里跳到 `main.py` 第 42 行。用户可以实时看到 Agent 的工作位置。

这和 IDE 的"跟随调试"类似，只不过跟的是 AI 而不是断点。

---

## 文件系统操作

Agent 通过 Client 的能力读写文件。这不是直接访问磁盘——**是通过 ACP 协议请求 Client 操作**。

### 为什么 Agent 不直接读文件

因为 Client 的编辑器里可能有**未保存的修改**：

```
用户打开了 main.py，改了几行但没保存
  → 磁盘上的 main.py = 旧版本
  → 编辑器里的 main.py = 新版本（未保存）

Agent 直接读磁盘 → 读到旧版本 ❌
Agent 通过 Client 读 → 读到编辑器里的最新版本 ✅
```

### 前提条件

Client 必须在 `initialize` 中声明文件系统能力：

```json
{
  "clientCapabilities": {
    "fs": {
      "readTextFile": true,       // 支持 fs/read_text_file
      "writeTextFile": true       // 支持 fs/write_text_file
    }
  }
}
```

Agent 调用前**必须**检查能力。`readTextFile` 是 `false` 或没出现，就不能调。

### 读文件

```json
// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "fs/read_text_file",
  "params": {
    "sessionId": "sess_abc123",
    "path": "/home/user/src/main.py",    // 绝对路径
    "line": 10,                          // 从第 10 行开始（可选）
    "limit": 50                          // 最多读 50 行（可选）
  }
}

// Client → Agent
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": "def hello_world():\n    print('Hello, world!')\n"
  }
}
```

行号从 1 开始。`line` 和 `limit` 都是可选的——不传就读整个文件。

### 写文件

```json
// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "fs/write_text_file",
  "params": {
    "sessionId": "sess_abc123",
    "path": "/home/user/config.json",    // 绝对路径
    "content": "{\n  \"debug\": true\n}"
  }
}

// Client → Agent（成功）
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": null                         // 空结果 = 成功
}
```

文件不存在？Client **必须**创建它。

---

## 终端操作

Agent 通过 Client 的终端能力执行命令。比直接 `system()` 强得多——可以实时获取输出、控制进程生命周期。

### 前提条件

```json
{
  "clientCapabilities": {
    "terminal": true              // 支持所有 terminal/* 方法
  }
}
```

### 完整 API

```
terminal/create    → 启动命令，立刻返回 terminalId
terminal/output    → 获取当前输出（不等待完成）
terminal/wait_for_exit → 等待命令结束
terminal/kill      → 杀死进程（不释放资源）
terminal/release   → 释放所有资源（会先 kill）
```

### 创建终端

```json
// Agent → Client
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "terminal/create",
  "params": {
    "sessionId": "sess_abc123",
    "command": "npm",                     // 要执行的命令
    "args": ["test", "--coverage"],       // 命令参数
    "env": [                              // 环境变量
      { "name": "NODE_ENV", "value": "test" }
    ],
    "cwd": "/home/user/project",          // 工作目录
    "outputByteLimit": 1048576            // 输出上限（字节）
  }
}

// Client → Agent（立刻返回）
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "terminalId": "term_xyz789"           // 后续操作用这个 ID
  }
}
```

命令后台运行，Agent 可以继续做别的事。

`outputByteLimit` 控制输出缓冲区大小。超出后，Client 从开头截断。Client 保证截断在字符边界上。

### 获取输出

```json
// Agent → Client
{
  "method": "terminal/output",
  "params": {
    "sessionId": "sess_abc123",
    "terminalId": "term_xyz789"
  }
}

// Client → Agent
{
  "result": {
    "output": "Running tests...\n✓ All tests passed (42 total)\n",
    "truncated": false,                   // 输出是否被截断
    "exitStatus": {                       // 命令已结束才有
      "exitCode": 0,
      "signal": null
    }
  }
}
```

`exitStatus` 只在命令结束后出现。命令还在跑的时候，`exitStatus` 不存在。

### 等待退出

```json
// Agent → Client（阻塞直到命令结束）
{
  "method": "terminal/wait_for_exit",
  "params": {
    "sessionId": "sess_abc123",
    "terminalId": "term_xyz789"
  }
}

// Client → Agent（命令结束后返回）
{
  "result": {
    "exitCode": 0,                        // 正常退出
    "signal": null                        // 被信号终止时有值
  }
}
```

### 杀死进程

```json
{
  "method": "terminal/kill",
  "params": {
    "sessionId": "sess_abc123",
    "terminalId": "term_xyz789"
  }
}
```

Kill 后终端还是有效的——可以继续调 `output` 和 `wait_for_exit`。用完了**必须**调 `release`。

### 释放终端

```json
{
  "method": "terminal/release",
  "params": {
    "sessionId": "sess_abc123",
    "terminalId": "term_xyz789"
  }
}
```

释放 = kill（如果还在跑） + 清理资源。之后 `terminalId` 失效，所有 `terminal/*` 方法都不能再用了。

⚠️ 如果终端被嵌入了工具调用的 content 中，Client **应该**在 release 后继续显示已有输出。

---

## 终端超时模式

Agent 想限制命令执行时间，可以用超时模式：

```python
import asyncio

async def run_command_with_timeout(session, command, timeout=30):
    # 1. 创建终端
    result = await session.request("terminal/create", {
        "command": command,
        "outputByteLimit": 1048576
    })
    terminal_id = result["terminalId"]

    try:
        # 2. 并发等待：命令完成 vs 超时
        done, pending = await asyncio.wait(
            [
                session.request("terminal/wait_for_exit", {
                    "terminalId": terminal_id
                }),
                asyncio.sleep(timeout)
            ],
            return_when=asyncio.FIRST_COMPLETED
        )

        # 3. 超时了 → 杀掉进程
        if any(t == sleep_task for t in pending):
            await session.request("terminal/kill", {
                "terminalId": terminal_id
            })
            output = await session.request("terminal/output", {
                "terminalId": terminal_id
            })
            return {"timed_out": True, "output": output["output"]}

        # 4. 正常完成
        return done.pop().result()

    finally:
        # 5. 不管怎样都释放
        await session.request("terminal/release", {
            "terminalId": terminal_id
        })
```

关键步骤：create → 并发等待 → 超时 kill → 拿输出 → release。

---

## 终端嵌入工具调用

终端可以嵌入到工具调用的 content 中，实现实时输出：

```json
{
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123",
    "update": {
      "sessionUpdate": "tool_call",
      "toolCallId": "call_002",
      "title": "运行测试",
      "kind": "execute",
      "status": "in_progress",
      "content": [
        {
          "type": "terminal",
          "terminalId": "term_xyz789"    // 嵌入终端
        }
      ]
    }
  }
}
```

Client 会把这个工具调用和终端关联起来，实时显示命令输出。用户看到的效果就是：Agent 旁边跑着一个终端窗口。

---

## 实战：Agent 执行一个命令的完整流程

Agent 想执行 `npm test` 并分析结果：

```
1. Agent → Client: session/update (tool_call)
   "运行测试" (call_002, kind=execute, status=pending)

2. Agent → Client: session/request_permission
   "Agent 想执行 npm test，允许？"

3. Client → Agent: 允许

4. Agent → Client: terminal/create
   command: npm, args: ["test"]

5. Client → Agent: terminalId: "term_abc"

6. Agent → Client: session/update (tool_call_update)
   call_002: status=in_progress, content=[terminal:"term_abc"]

7. Agent → Client: terminal/wait_for_exit
   (等待测试跑完)

8. Client → Agent: exitCode: 1  (测试失败了)

9. Agent → Client: terminal/output
   (获取失败详情)

10. Agent → Client: terminal/release

11. Agent → Client: session/update (tool_call_update)
    call_002: status=completed, content="3 tests failed..."

12. Agent → Client: session/update (agent_message_chunk)
    "测试有 3 个失败，我来修复..."

13. (Agent 把结果喂回 LLM，继续处理)
```

---

## 速查表

### 工具调用 API

| 操作 | 方法 | 方向 |
|------|------|------|
| 创建 | `session/update` (tool_call) | Agent → Client |
| 更新 | `session/update` (tool_call_update) | Agent → Client |
| 请求权限 | `session/request_permission` | Agent → Client |
| 用户选择 | response | Client → Agent |

### 权限选项

| Kind | 含义 |
|------|------|
| `allow_once` | 允许一次 |
| `allow_always` | 永远允许 |
| `reject_once` | 拒绝一次 |
| `reject_always` | 永远拒绝 |

### 文件系统 API

| 方法 | 前提条件 | 说明 |
|------|---------|------|
| `fs/read_text_file` | `fs.readTextFile: true` | 读文件（含未保存状态） |
| `fs/write_text_file` | `fs.writeTextFile: true` | 写文件（不存在则创建） |

### 终端 API

| 方法 | 前提条件 | 说明 |
|------|---------|------|
| `terminal/create` | `terminal: true` | 创建终端，后台运行 |
| `terminal/output` | 同上 | 获取当前输出 |
| `terminal/wait_for_exit` | 同上 | 阻塞等待完成 |
| `terminal/kill` | 同上 | 杀死进程 |
| `terminal/release` | 同上 | kill + 释放资源 |

### 终端生命周期

```
create → (output × N) → wait_for_exit → release
                      ↘ kill → output → release
```

---

## 参考资料

- Tool Calls 规范：https://agentclientprotocol.com/protocol/tool-calls
- File System 规范：https://agentclientprotocol.com/protocol/file-system
- Terminals 规范：https://agentclientprotocol.com/protocol/terminals

---

**系列导航**：[第 1 篇：协议全景](acp-01-protocol-overview-2026-04-18.md) → [第 2 篇：初始化握手](acp-02-initialization-and-sessions-2026-04-18.md) → [第 3 篇：交互循环](acp-03-prompt-turn-and-content-2026-04-18.md) → [第 4 篇：工具系统](acp-04-tools-permissions-and-resources-2026-04-18.md) → [第 5 篇：会话控制与扩展](acp-05-session-control-and-extensibility-2026-04-18.md)
