# MCP Apps：让 AI 对话里长出交互界面

> MCP Apps 允许 MCP Server 返回可交互的 HTML 界面（图表、表单、仪表盘），直接渲染在聊天窗口里。不需要跳转网页，不需要单独部署前端，数据和 UI 都在对话上下文中。

---

## 为什么不直接做个网页？

你当然可以建个 Web App，把链接丢给用户。但 MCP Apps 有几个本质优势：

| 维度 | 独立 Web App | MCP App |
|------|-------------|---------|
| **上下文** | 用户切到新标签页，丢失对话上下文 | 界面就在对话里，所见即所得 |
| **数据流** | 需要自己建 API、做鉴权、管状态 | 复用 MCP 协议，直接调工具拿数据 |
| **能力集成** | 每个功能自己实现（发邮件、查日历...） | 委托给 Host，用用户已连接的能力 |
| **安全** | 要自己处理 XSS、CSRF、认证 | 沙箱 iframe 隔离，Host 控制权限 |

如果你的场景不需要这些特性，普通 Web App 更简单。但如果你要和 LLM 对话深度集成，MCP App 是更好的选择。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP Host (Claude / VS Code / ...)      │
│                                                             │
│  ┌──────────┐   JSON-RPC    ┌──────────┐   JSON-RPC        │
│  │  对话 UI  │ ◄──────────► │ MCP      │ ◄──────────► ...  │
│  │          │              │ Server   │                    │
│  └──────────┘              └──────────┘                    │
│       │                          │                         │
│       │   工具返回 UI 引用         │  提供 ui:// 资源         │
│       ◄──────────────────────────►                         │
│       │                                                    │
│       ▼                                                    │
│  ┌──────────────────────────────────────┐                  │
│  │     沙箱 iframe (MCP App)             │                  │
│  │  ┌────────────────────────────────┐  │                  │
│  │  │  HTML + JS + CSS               │  │                  │
│  │  │  (交互式图表/表单/仪表盘)       │  │                  │
│  │  │                                │  │                  │
│  │  │  ◄── postMessage ──► Host      │  │                  │
│  │  └────────────────────────────────┘  │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

三个核心角色：

| 角色 | 说明 | 例子 |
|------|------|------|
| **Host** | 运行 LLM 对话的客户端 | Claude Desktop、VS Code Copilot、Goose、Postman |
| **MCP Server** | 提供工具和数据，同时提供 UI 资源 | 你的业务 Server |
| **MCP App** | 沙箱内的交互式 HTML 界面 | 数据可视化、配置表单、文件预览 |

---

## 工作流程

当 LLM 决定调用一个支持 MCP App 的工具时，完整流程如下：

```
1. LLM 选择调用工具
   └── 工具描述里包含 _meta.ui.resourceUri（指向 ui:// 资源）
       │
2. Host 预加载 UI 资源
   └── 从 MCP Server 拿到 HTML 页面（含 JS/CSS）
       │
3. 沙箱渲染
   └── Host 把 HTML 放进 sandboxed iframe
   └── 可配置 permissions（摄像头、麦克风）和 csp（外部资源白名单）
       │
4. 双向通信
   └── App 通过 postMessage ↔ Host 通信（JSON-RPC 协议）
   └── App 可调用 MCP 工具、更新上下文、接收推送数据
```

关键点：App 被隔离在 iframe 里，不能访问父页面 DOM、Cookie、localStorage。所有通信走 postMessage，Host 控制哪些工具可以被调用。

---

## 核心协议：UI 通信

MCP App 使用自己的 JSON-RPC 方言，和核心 MCP 协议共享部分消息类型：

### 初始化

```json
// Host → App：初始化握手
{
  "jsonrpc": "2.0",
  "method": "ui/initialize",
  "params": {
    "capabilities": {
      "tools": { "listChanged": true },
      "displayModes": ["inline", "fullscreen", "pip"]
    }
  }
}

// App → Host：确认能力
{
  "jsonrpc": "2.0",
  "result": {
    "capabilities": {
      "toolInvocation": true,
      "sendMessageBox": true
    }
  }
}
```

### App 调用工具

```json
// App → Host：请求调用 MCP 工具
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_sales_data",
    "arguments": { "region": "east", "quarter": "Q1" }
  }
}

// Host → App：返回结果
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "{\"east_q1\": 125000}" }
    ]
  }
}
```

### Host 推送数据给 App

```json
// Host → App：工具执行结果推送到 App
{
  "jsonrpc": "2.0",
  "method": "ui/toolResult",
  "params": {
    "toolCallId": "call_001",
    "content": [
      { "type": "text", "text": "{\"visits\": 4218, "conversions\": 83}" }
    ]
  }
}
```

### 消息类型一览

| 方向 | 方法 | 说明 |
|------|------|------|
| Host → App | `ui/initialize` | 初始化握手 |
| App → Host | `tools/call` | 调用 MCP 工具（复用核心协议） |
| Host → App | `ui/toolResult` | 推送工具结果到 App |
| App → Host | `ui/sendMessage` | App 发消息给对话 |
| App → Host | `ui/requestDisplayMode` | 请求切换显示模式 |
| Host → App | `ui/displayModeChanged` | 显示模式变更通知 |

---

## 显示模式

MCP App 支持三种显示模式，适应不同交互场景：

```
┌─────────────────────────────────────┐
│  Inline（内嵌）                      │
│  在对话气泡中显示，适合摘要卡片        │
│  ┌───────────────────────────────┐  │
│  │  销售概览: ¥125,000 (+12%)   │  │
│  │  [展开详情]                   │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Fullscreen（全屏）                  │
│  占满整个内容区域，适合复杂仪表盘      │
│  ┌───────────────────────────────┐  │
│  │                               │  │
│  │   完整的数据分析仪表盘          │  │
│  │   图表 / 筛选器 / 明细表       │  │
│  │                               │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  PiP（画中画）                      │
│  浮动小窗口，适合监控类场景           │
│                    ┌──────────┐     │
│  继续对话...       │ 实时指标  │     │
│                    │ CPU 42%  │     │
│                    └──────────┘     │
└─────────────────────────────────────┘
```

App 可以主动请求切换模式：

```javascript
// 使用 sunpeak 框架
import { useDisplayMode } from 'sunpeak';

function Dashboard() {
  const { displayMode, requestDisplayMode } = useDisplayMode();

  return (
    <div>
      {displayMode === 'inline' && (
        <>
          <SummaryCard />
          <button onClick={() => requestDisplayMode('fullscreen')}>
            展开完整视图
          </button>
        </>
      )}
      {displayMode === 'fullscreen' && <FullDashboard />}
    </div>
  );
}
```

---

## 设计模式：Resource 不是 Tool

最常见的设计错误是 **一个 Tool 对应一个 Resource（UI 页面）**。这会导致组件重复、共享状态混乱。

正确做法是围绕 **视图（View）** 设计 Resource，围绕 **操作** 设计 Tool：

```
❌ 错误：1:1 映射

  Tool: get_overview  → Resource: OverviewResource
  Tool: get_breakdown → Resource: BreakdownResource
  Tool: refresh_data  → Resource: RefreshResource
  （三个 Resource，大量重复 UI 逻辑）


✅ 正确：多个 Tool 共享一个 Resource

  Tool: get_overview  ──┐
  Tool: get_breakdown ──┼──→ Resource: DashboardResource（一个组件，多种数据）
  Tool: refresh_data  ──┘
```

Resource 是「屏幕」，Tool 是「让屏幕出现或更新数据的操作」。保持 Resource 数量少，UI 逻辑集中。

---

## 实战：构建一个 MCP App

### 项目结构

```
my-mcp-app/
├── src/
│   ├── resources/
│   │   └── dashboard.tsx        # UI 组件（React/Vue/Svelte/...）
│   ├── tools/
│   │   ├── get-overview.ts      # 工具：获取概览数据
│   │   └── get-breakdown.ts     # 工具：获取明细数据
│   └── server.ts                # MCP Server 入口
├── package.json
└── tsconfig.json
```

### 定义工具

```typescript
// src/tools/get-overview.ts
import { z } from 'zod';
import type { AppToolConfig } from '@modelcontextprotocol/ext-apps';

export const tool: AppToolConfig = {
  resource: 'dashboard',    // 指向哪个 Resource
  title: '获取销售概览',
  description: '显示指定时间范围的销售数据概览',
  annotations: { readOnlyHint: true },
};

export const schema = {
  timeRange: z.string().describe('时间范围，如 "7d"、"30d"'),
};

export default async function (args: { timeRange: string }) {
  return {
    structuredContent: {
      visits: 4218,
      conversions: 83,
      bounceRate: 0.41,
    },
  };
}
```

### 定义 UI 资源

```tsx
// src/resources/dashboard.tsx
import {
  useToolData,
  useHostContext,
  useDisplayMode,
  AppProvider,
} from '@modelcontextprotocol/ext-apps';
import { BarChart, Card } from './components';

function DashboardResource() {
  const { data } = useToolData();
  const { theme } = useHostContext();
  const { displayMode } = useDisplayMode();

  return (
    <div className={theme === 'dark' ? 'dark' : 'light'}>
      <Card title="销售概览">
        <BarChart
          data={data}
          compact={displayMode === 'inline'}
        />
      </Card>
    </div>
  );
}

export default DashboardResource;
```

### Server 注册资源

```typescript
// src/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { registerApp } from '@modelcontextprotocol/ext-apps';

const server = new McpServer({ name: 'sales-dashboard', version: '1.0.0' });

// 注册工具
server.tool('get_overview', '获取销售概览', schema, getOverviewHandler);

// 注册 UI 资源（HTML bundle）
registerApp(server, {
  resources: { dashboard: DashboardResource },
  tools: { get_overview: getOverviewTool },
});

server.run();
```

---

## 安全模型

MCP App 的安全基于**三层隔离**：

```
┌─────────────────────────────────────────┐
│  Host（主应用）                           │
│  ┌─────────────────────────────────────┐ │
│  │  sandboxed iframe                    │ │
│  │  ┌───────────────────────────────┐  │ │
│  │  │  MCP App                      │  │ │
│  │  │                               │  │ │
│  │  │  ❌ 不能访问：                 │  │ │
│  │  │  - 父页面 DOM                 │  │ │
│  │  │  - Host 的 Cookie             │  │ │
│  │  │  - localStorage               │  │ │
│  │  │  - 导航父页面                  │  │ │
│  │  │                               │  │ │
│  │  │  ✅ 只能通过：                 │  │ │
│  │  │  - postMessage 和 Host 通信    │  │ │
│  │  │  - Host 授权的工具调用         │  │ │
│  │  └───────────────────────────────┘  │ │
│  └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

| 层 | 机制 | 作用 |
|----|------|------|
| **iframe sandbox** | 浏览器原生沙箱 | 隔离 DOM、Cookie、存储 |
| **CSP** | `_meta.ui.csp` 配置 | 控制可加载的外部资源来源 |
| **Host 网关** | postMessage 代理 | 控制哪些工具可被调用、哪些能力可使用 |

Host 可以进一步限制：
- 禁用特定工具调用
- 禁用 `sendOpenLink`（防止跳转）
- 拒绝 `permissions` 请求（摄像头、麦克风等）

---

## 测试策略

MCP App 跑在沙箱 iframe 里，不能直接在浏览器打开测试。用 **模拟文件（Simulation）** 解决：

```json
// simulations/get-dashboard.json
{
  "tool": "get_overview",
  "userMessage": "显示上周的销售数据",
  "toolInput": { "timeRange": "7d" },
  "toolResult": {
    "structuredContent": {
      "visits": 4218,
      "conversions": 83,
      "bounceRate": 0.41
    }
  }
}
```

测试覆盖每种 UI 状态：

```bash
pnpm test          # 单元测试（Vitest）
pnpm test:e2e      # 端到端测试（Playwright）
```

**关键原则**：模拟文件测的是标准运行时，不是某个特定 Host。测试通过 = 在所有 Host 上都能用，不需要逐个平台手动验证。

---

## 框架支持

MCP App 的底层是标准 Web 原语（postMessage + JSON-RPC），所以可以用任何框架或不用框架：

| 框架 | 说明 |
|------|------|
| React | 官方示例 + `@modelcontextprotocol/ext-apps` 的 `App` 类 |
| Vue | 官方示例模板 |
| Svelte | 官方示例模板 |
| Preact | 官方示例模板 |
| Solid | 官方示例模板 |
| Vanilla JS | 直接用 postMessage，零依赖 |

`@modelcontextprotocol/ext-apps` 的 `App` 类是便利封装，不是必须的。你可以直接实现 postMessage 协议，获得更小的 bundle 和更精确的控制。

---

## 适用场景

### 适合用 MCP App

- **复杂数据探索** —— 用户问"按地区看销售额"，返回可交互地图，点击下钻
- **多选项配置** —— 部署配置涉及几十个互相关联的选项，表单一次展示，实时验证
- **富媒体预览** —— PDF 查看器、3D 模型旋转、图片画廊，文本描述替代不了
- **实时监控** —— 持续更新的仪表盘，不需要反复问"现在什么状态"
- **多步审批** —— 逐条审核报销单、代码变更、工单分流，有导航和持久状态

### 不适合用 MCP App

- 纯文本就能回答的问题
- 一次性、不需要交互的结果
- 需要脱离对话独立使用的功能（这种情况做独立 Web App）

---

## 当前生态

### 支持 MCP App 的 Host

Claude、Claude Desktop、VS Code GitHub Copilot、Goose、Postman、MCPJam 等。

### 官方示例（ext-apps 仓库）

| 类别 | 示例 |
|------|------|
| **3D / 可视化** | map-server（CesiumJS 地球）、threejs-server（Three.js 场景）、shadertoy-server（着色器效果） |
| **数据探索** | cohort-heatmap-server、customer-segmentation-server、wiki-explorer-server |
| **业务应用** | scenario-modeler-server、budget-allocator-server |
| **媒体** | pdf-server、video-resource-server、sheet-music-server、say-server（TTS） |
| **工具** | qr-server、system-monitor-server、transcript-server（STT） |

---

## 构建跨平台 App 的正确顺序

如果你想一次构建、所有 Host 都能用，遵循这个顺序：

```
1. 定义数据结构
   └── 每个工具返回什么？先写类型定义

2. 写模拟文件
   └── 每种 UI 状态一个：正常、空数据、错误、不同显示模式

3. 构建 Resource 组件（只用标准 API）
   └── useToolData / useHostContext / useDisplayMode
   └── 在 Inspector 里验证

4. 写测试
   └── 用模拟文件跑测试，通过 = 全平台可用

5. 加 Host 特定增强
   └── 通过子路径导入（如 sunpeak/chatgpt）
   └── 这是增强，不是核心依赖

6. 接入真实 Host
   └── Claude Desktop、VS Code、ChatGPT 逐个验证
```

**核心原则**：先用标准 API 把可移植层做对，再考虑特定 Host 的增强功能。反过来做会导致代码和特定平台纠缠，后续移植代价很大。

---

## 和 MCP 核心协议的关系

MCP App 是 MCP 协议的**扩展**，不是替代：

| 维度 | MCP 核心协议 | MCP Apps 扩展 |
|------|-------------|---------------|
| **返回类型** | text / image / resource | 交互式 HTML UI |
| **传输** | stdio / HTTP | postMessage（iframe 内） |
| **通信** | Server ↔ Host | App ↔ Host（通过 Host 代理） |
| **消息格式** | JSON-RPC 2.0 | JSON-RPC 2.0（ui/ 前缀） |
| **安全** | 进程级隔离 | iframe sandbox + CSP |

MCP Server 可以同时支持传统工具和 MCP App。现有 Server 加上 `_meta.ui.resourceUri` 字段就能升级，不需要重写。

---

## 参考资料

- 官方文档：https://modelcontextprotocol.io/extensions/apps/overview
- 构建指南：https://modelcontextprotocol.io/extensions/apps/building
- 示例仓库：https://github.com/modelcontextprotocol/ext-apps
- 客户端 SDK：`@mcp-ui/client`（React 组件）、`@modelcontextprotocol/ext-apps`（App 开发工具）
