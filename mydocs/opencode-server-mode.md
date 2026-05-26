# OpenCode Server 模式：通过 HTTP API 远程使用 OpenCode

> 基于 `packages/opencode` 源码分析（`dev` 分支）

## 概述

OpenCode 内置了成熟的 **无头服务器模式**，可以作为 HTTP 服务运行，通过 REST API 远程完成对话、代码分析、项目探索等所有 Agent 操作。不再局限于本地终端 TUI。

### 核心能力

- 完整的 HTTP REST API（基于 Effect `HttpApi` 框架）
- WebSocket 实时通信（PTY 终端、事件流）
- SSE（Server-Sent Events）事件订阅
- ACP（Agent-Client Protocol）支持
- 多项目并发服务（通过 `x-opencode-directory` 头路由）
- HTTP Basic Auth 安全认证
- 官方 TypeScript SDK（`@opencode-ai/sdk`）

---

## 快速启动

### 无头服务器

```bash
# 推荐：设置密码后启动
OPENCODE_SERVER_PASSWORD=mysecret opencode serve --port 8080

# 监听所有网络接口（局域网/远程访问）
OPENCODE_SERVER_PASSWORD=mysecret opencode serve --port 8080 --hostname 0.0.0.0

# 带 Web UI 的服务器
opencode web --port 8080

# 启动 ACP 协议服务器
opencode acp
```

### 网络选项

| 参数 | 默认值 | 描述 |
|------|--------|------|
| `--port` | `0`（自动分配） | 监听端口 |
| `--hostname` | `127.0.0.1` | 监听地址 |
| `--mdns` | `false` | 启用 mDNS 服务发现 |
| `--mdns-domain` | `opencode.local` | mDNS 域名 |
| `--cors` | `[]` | 额外 CORS 域名 |

### 通过 SDK 编程式启动

```typescript
import { createOpencodeServer } from "@opencode-ai/sdk/v2"

const server = await createOpencodeServer({
  port: 4096,
  hostname: "127.0.0.1",
  config: { logLevel: "info" },
})
// => { url: "http://127.0.0.1:4096", close() { ... } }
```

---

## 架构总览

### 核心文件

| 文件 | 作用 |
|------|------|
| `src/cli/cmd/serve.ts` | `opencode serve` 命令入口 |
| `src/cli/cmd/web.ts` | `opencode web` 命令入口 |
| `src/server/server.ts` | HTTP 服务器核心（`node:http` + Effect HttpRouter） |
| `src/server/routes/instance/httpapi/server.ts` | 路由组装、中间件编排 |
| `src/server/routes/instance/httpapi/api.ts` | HttpApi 定义（路由树组成） |
| `src/server/routes/instance/httpapi/public.ts` | OpenAPI 规范生成 |
| `src/server/auth.ts` | 认证逻辑 |
| `src/cli/network.ts` | 网络配置解析 |
| `src/config/server.ts` | 服务器配置 Schema |

### 技术栈

- **Web 框架**：Effect `HttpRouter` / `HttpApi`（`effect/unstable/http` + `effect/unstable/httpapi`）
- **底层**：`node:http`（原生 `createServer()`）
- **序列化**：OpenAPI 自动生成
- **实时通信**：`effect/unstable/socket/Socket`（WebSocket）+ SSE
- **Pagination**：游标分页（V2 API）
- **实例隔离**：`InstanceState` + `ScopedCache`

### 请求路由机制

```
                         ┌──────────────────┐
                         │   node:http       │
                         │   createServer()  │
                         └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │  HttpRouter.serve │
                         │  (Effect)         │
                         └────────┬─────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
     ┌──────▼──────┐      ┌──────▼──────┐      ┌──────▼──────┐
     │   Middleware │      │  HttpApi    │      │   Raw Route │
     │  (Auth/CORS/ │      │  Routes     │      │  (WebSocket)│
     │   Compression)│      │ (typed)     │      │             │
     └──────────────┘      └──────┬──────┘      └─────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
             ┌──────▼──────┐ ┌───▼───┐ ┌──────▼──────┐
             │  Root API   │ │ Event │ │  Instance   │
             │ (/global/*) │ │ (/event)│ │  API        │
             └─────────────┘ └───────┘ │ (/session/*) │
                                       │ (/project/*) │
                                       │ (/file/*)    │
                                       │ (/api/*)     │
                                       └──────────────┘
                                              │
                                    ┌─────────▼─────────┐
                                    │ x-opencode-directory│
                                    │ header → Workspace │
                                    │ InstanceContext     │
                                    └────────────────────┘
```

---

## 完整 API 参考

### 全局 API（无需项目上下文）

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/global/health` | **健康检查** — 返回 `{"healthy":true,"version":"..."}` |
| `GET` | `/global/event` | **SSE 事件流** — 订阅全局事件 |
| `GET` | `/global/config` | 获取全局配置 |
| `PATCH` | `/global/config` | 更新全局配置 |
| `POST` | `/global/dispose` | 销毁所有实例 |
| `POST` | `/global/upgrade` | 升级 OpenCode 版本 |
| `GET` | `/doc` | **OpenAPI 规范** — 完整 API 文档 |

### 会话 API（V1 — 需要 `x-opencode-directory`）

#### 基础 CRUD

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/session` | 列出会话 |
| `GET` | `/session/status` | 获取会话状态 |
| `GET` | `/session/:id` | 获取单个会话 |
| `PATCH` | `/session/:id` | 更新会话（标题、权限、归档） |
| `DELETE` | `/session/:id` | 删除会话 |
| `POST` | `/session/:id/fork` | 派生会话 |
| `GET` | `/session/:id/children` | 获取子会话 |

#### 消息与交互

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/session/:id/message` | 获取消息列表 |
| `POST` | `/session/:id/message` | **发送消息/提示词** |
| `POST` | `/session/:id/init` | **初始化**会话（绑定模型） |
| `POST` | `/session/:id/prompt` | **发送提示词** |
| `POST` | `/session/:id/prompt-async` | 异步发送提示词 |
| `POST` | `/session/:id/command` | 执行命令 |
| `POST` | `/session/:id/shell` | 执行 Shell |
| `POST` | `/session/:id/revert` | 回退操作 |
| `POST` | `/session/:id/unrevert` | 取消回退 |

#### 其他操作

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/session/:id/todo` | 获取 Todo 列表 |
| `GET` | `/session/:id/diff` | 获取差异 |
| `POST` | `/session/:id/abort` | 终止会话 |
| `POST` | `/session/:id/summarize` | 总结会话 |
| `POST` | `/session/:id/share` | 分享会话 |
| `POST` | `/session/:id/unshare` | 取消分享 |
| `POST` | `/session/:id/permission-response` | 权限响应 |

### 会话 API（V2 — 新一代 API）

V2 API 前缀为 `/api/`，支持游标分页和更完善的 Schema。

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/api/session` | **列出会话**（游标分页，支持 `order`/`search`/`cursor`） |
| `POST` | `/api/session/:id/prompt` | **发送提示词**（`delivery`: `immediate` 或 `deferred`） |
| `POST` | `/api/session/:id/compact` | 压缩会话 |
| `POST` | `/api/session/:id/wait` | **等待会话空闲** |
| `GET` | `/api/session/:id/context` | 获取会话上下文 |
| `GET` | `/api/session/:id/message` | **获取消息列表**（支持游标分页） |
| `GET` | `/api/model` | 列出模型 |
| `GET` | `/api/provider` | 列出提供商 |

### 文件 API

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/file/read` | 读取文件 |
| `POST` | `/file/write` | 写入文件 |
| `POST` | `/file/edit` | 编辑文件 |
| `POST` | `/file/create` | 创建文件 |
| `POST` | `/file/delete` | 删除文件 |
| `POST` | `/file/rename` | 重命名文件 |
| `GET` | `/find/file` | 搜索文件 |

### 项目 API

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/project` | 列出项目 |
| `POST` | `/project` | **创建/打开项目** |
| `POST` | `/project/switch` | 切换活跃项目 |

### VCS/Git API

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/vcs` | 获取 VCS 状态 |
| `GET` | `/vcs/status` | 获取 Git 状态 |
| `GET` | `/vcs/diff` | 获取差异 |
| `GET` | `/vcs/diff/raw` | 获取原始差异 |
| `POST` | `/vcs/apply` | 应用补丁 |

### PTY 终端 API

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/pty` | 列出 PTY |
| `POST` | `/pty` | 创建 PTY 会话 |
| `GET` | `/pty/:id` | 获取 PTY 详情 |
| `PATCH` | `/pty/:id` | 更新 PTY |
| `DELETE` | `/pty/:id` | 删除 PTY |
| `GET` | `/pty/:id/connect` | **WebSocket 升级** — 实时终端 I/O |

### 其他 API

| 方法 | 路径 | 描述 |
|------|------|------|
| `GET` | `/provider` | 列出提供商 |
| `GET` | `/config/provider` | 获取提供商配置 |
| `GET` | `/agent` | 列出 Agent |
| `GET` | `/skill` | 列出技能 |
| `GET` | `/command` | 列出命令 |
| `GET` | `/lsp` | LSP 状态 |
| `GET` | `/formatter` | 格式化器状态 |
| `GET\|POST` | `/mcp` | MCP 服务器管理 |
| `POST` | `/event` | 事件推送 |
| `POST` | `/permission` | 权限响应 |
| `POST` | `/question` | 问题响应 |
| `POST` | `/workspace` | 工作区操作 |
| `POST` | `/sync` | 同步操作 |

---

## 认证

服务器使用 HTTP Basic Auth：

```bash
# 环境变量
export OPENCODE_SERVER_PASSWORD=mysecret
export OPENCODE_SERVER_USERNAME=opencode  # 可选，默认 "opencode"

# 请求头
Authorization: Basic $(echo -n "opencode:mysecret" | base64)
```

如果 `OPENCODE_SERVER_PASSWORD` 未设置，服务器会启动但打印警告：

```
Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured.
```

---

## 项目路由：x-opencode-directory

每次请求需要通过 `x-opencode-directory` 头部指定目标项目目录。这是 OpenCode 服务器**多项目并发**的关键机制。

### 传递方式

**HTTP Header**（推荐）：
```
x-opencode-directory: /path/to/project  (或 URL 编码)
```

**Query Parameter**：
```
GET /session?directory=/path/to/project
```

**SDK 自动处理**：
```typescript
const client = createOpencodeClient({
  baseUrl: "http://localhost:9999",
  directory: "/path/to/project",  // SDK 自动设置为请求头
  headers: { authorization: "Bearer mysecret" },
})
```

### 工作区路由（进阶）

V2 API 还支持 `x-opencode-workspace` 头/查询参数，用于在同一个项目内选择不同工作区。

---

## 实时通信

### SSE 事件流

```bash
# 订阅全局事件
curl -N http://localhost:9999/global/event -H "authorization: Basic ..."

# 订阅实例事件（需要 directory）
curl -N http://localhost:9999/event \
  -H "x-opencode-directory: /path/to/project" \
  -H "authorization: Basic ..."
```

返回 `text/event-stream` 格式，包含 `BusEvent` 和 `SyncEvent` 类型的事件。

### WebSocket PTY

```
GET /pty/:ptyID/connect?x-opencode-directory=/path/to/project
↑ WebSocket 升级，实时双向终端 I/O
```

使用 `effect/unstable/socket/Socket` 实现。

---

## SDK 使用

### 安装

```bash
npm install @opencode-ai/sdk
# 或
bun add @opencode-ai/sdk
```

### V2 客户端（推荐）

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk/v2/client"

const client = createOpencodeClient({
  baseUrl: "http://127.0.0.1:8080",
  directory: "/path/to/project",
  headers: {
    authorization: "Bearer mysecret",
  },
})

// 列出会话
const sessions = await client.v2.session.sessions({
  query: { limit: 10 },
})

// 发送提示词
const response = await client.v2.session.prompt({
  path: { sessionID: sessions.items[0].id },
  body: { prompt: { text: "分析这个项目的架构设计" } },
})

// 获取消息
const messages = await client.v2.message.messages({
  path: { sessionID: sessions.items[0].id },
  query: { limit: 50 },
})
```

### 服务器 + 客户端一站式

```typescript
import { createOpencode } from "@opencode-ai/sdk/v2"

const { server, client } = await createOpencode({
  port: 4096,
  hostname: "127.0.0.1",
})

// client 已自动连接到 server
const health = await client.global.health()
console.log(health)
```

---

## 实战示例

### 示例 1：远程分析项目

```bash
#!/bin/bash
PASSWORD="mysecret"
BASE="http://127.0.0.1:8080"
AUTH="Authorization: Basic $(echo -n opencode:$PASSWORD | base64)"
DIR="x-opencode-directory: /path/to/target-project"

# 1. 健康检查
curl -s "$BASE/global/health" | jq .

# 2. 创建/打开项目
curl -s -X POST "$BASE/project" -H "$AUTH" -H "$DIR" -H "content-type: application/json" \
  -d '{}' | jq .

# 3. 创建会话（通过 init）
INIT_RESP=$(curl -s -X POST "$BASE/session" -H "$AUTH" -H "$DIR" -H "content-type: application/json" \
  -d '{"providerID": "anthropic", "modelID": "claude-sonnet-4-20250514"}')
SESSION_ID=$(echo "$INIT_RESP" | jq -r '.id')
echo "Session ID: $SESSION_ID"

# 4. 发送分析请求
curl -s -X POST "$BASE/session/$SESSION_ID/message" \
  -H "$AUTH" -H "$DIR" -H "content-type: application/json" \
  -d '{"prompt": "分析这个项目的架构和代码质量，给出改进建议"}' | jq .

# 5. 获取完整消息记录
curl -s "$BASE/session/$SESSION_ID/message" -H "$AUTH" -H "$DIR" | jq .
```

### 示例 2：Node.js 程序方式

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk/v2/client"
import { createOpencodeServer } from "@opencode-ai/sdk/v2/server"

async function analyzeProject() {
  // 启动服务器
  const server = await createOpencodeServer({
    port: 4096,
    hostname: "127.0.0.1",
  })

  // 创建客户端
  const client = createOpencodeClient({
    baseUrl: server.url,
    directory: "/path/to/project",
    headers: { authorization: "Bearer mysecret" },
  })

  try {
    // 创建会话
    const session = await client.session.create({
      body: {
        providerID: "anthropic",
        modelID: "claude-sonnet-4-20250514",
      },
    })

    const sessionId = session.id

    // 发送提示词
    const result = await client.session.prompt({
      path: { sessionID: sessionId },
      body: { prompt: "分析此项目的技术栈和架构" },
    })

    console.log("分析结果:", result)

    // 获取消息历史
    const messages = await client.session.messages({
      path: { sessionID: sessionId },
    })

    for (const msg of messages.items) {
      console.log(`[${msg.role}]`, msg.content)
    }
  } finally {
    server.close()
  }
}
```

### 示例 3：SSE 实时事件监听

```typescript
async function watchEvents(baseUrl: string, password: string) {
  const response = await fetch(`${baseUrl}/global/event`, {
    headers: {
      Authorization: `Basic ${btoa(`opencode:${password}`)}`,
    },
  })

  const reader = response.body!.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const text = decoder.decode(value)
    for (const line of text.split("\n")) {
      if (line.startsWith("data:")) {
        const event = JSON.parse(line.slice(5))
        console.log("事件:", event.payload?.type, event.directory)
      }
    }
  }
}
```

---

## 架构文档（源码路径参考）

### 入口与命令

| 路径 | 说明 |
|------|------|
| `packages/opencode/src/index.ts` | CLI 主入口 |
| `packages/opencode/src/cli/cmd/serve.ts` | `serve` 命令 |
| `packages/opencode/src/cli/cmd/web.ts` | `web` 命令 |
| `packages/opencode/src/cli/cmd/acp.ts` | `acp` 命令 |
| `packages/opencode/src/cli/network.ts` | 网络选项 |
| `packages/opencode/src/cli/effect-cmd.ts` | Effect 化命令基类 |

### 服务器核心

| 路径 | 说明 |
|------|------|
| `packages/opencode/src/server/server.ts` | HTTP 服务器核心 |
| `packages/opencode/src/server/auth.ts` | 服务器认证 |
| `packages/opencode/src/server/cors.ts` | CORS 配置 |
| `packages/opencode/src/server/mdns.ts` | mDNS 服务发现 |
| `packages/opencode/src/config/server.ts` | 服务器配置 Schema |
| `packages/opencode/src/server/init-projectors.ts` | 启动时初始化 |

### HTTP API 基础设施

| 路径 | 说明 |
|------|------|
| `src/server/routes/instance/httpapi/server.ts` | 路由组装 + 中间件编排 |
| `src/server/routes/instance/httpapi/api.ts` | HttpApi 定义（Root / Instance / OpenCode） |
| `src/server/routes/instance/httpapi/public.ts` | OpenAPI 规范生成 |
| `src/server/routes/instance/httpapi/lifecycle.ts` | 请求生命周期 |
| `src/server/routes/instance/httpapi/websocket-tracker.ts` | WebSocket 连接跟踪 |
| `src/server/routes/instance/httpapi/errors.ts` | API 错误类型 |

### Middleware

| 路径 | 说明 |
|------|------|
| `.../middleware/authorization.ts` | HTTP Basic Auth |
| `.../middleware/instance-context.ts` | 按请求加载实例 |
| `.../middleware/workspace-routing.ts` | `x-opencode-directory` 路由 |
| `.../middleware/error.ts` | 错误处理 |
| `.../middleware/compression.ts` | 压缩 |
| `.../middleware/schema-error.ts` | Schema 校验错误 |
| `.../middleware/cors-vary.ts` | CORS 修复 |
| `.../middleware/fence.ts` | 路由保护 |
| `.../middleware/proxy.ts` | 代理中间件 |

### API 分组（groups）

| 路径 | 说明 |
|------|------|
| `.../groups/global.ts` | `/global/*` |
| `.../groups/session.ts` | `/session/*` |
| `.../groups/event.ts` | `/event` SSE |
| `.../groups/file.ts` | `/file/*` |
| `.../groups/project.ts` | `/project/*` |
| `.../groups/config.ts` | `/config/*` |
| `.../groups/instance.ts` | `/instance/*`, `/vcs/*`, `/agent`, `/command` |
| `.../groups/provider.ts` | `/provider/*` |
| `.../groups/pty.ts` | `/pty/*` |
| `.../groups/mcp.ts` | `/mcp/*` |
| `.../groups/question.ts` | `/question/*` |
| `.../groups/permission.ts` | `/permission/*` |
| `.../groups/sync.ts` | `/sync/*` |
| `.../groups/tui.ts` | `/tui/*` |
| `.../groups/workspace.ts` | `/workspace/*` |
| `.../groups/experimental.ts` | `/experimental/*` |
| `.../groups/control.ts` | 控制端路由 |
| `.../groups/v2.ts` | V2 API 容器 |
| `.../groups/v2/session.ts` | `/api/session` |
| `.../groups/v2/message.ts` | `/api/session/:id/message` |
| `.../groups/v2/model.ts` | `/api/model` |
| `.../groups/v2/provider.ts` | `/api/provider` |
| `.../groups/v2/location.ts` | `/api/location` |

### 各分组对应的 handlers（实现层）

| 路径 | 说明 |
|------|------|
| `.../handlers/session.ts` | 会话处理器（30+ 方法） |
| `.../handlers/session-errors.ts` | 会话错误映射 |
| `.../handlers/global.ts` | 全局处理器 |
| `.../handlers/event.ts` | 事件流处理器 |
| `.../handlers/config.ts` | 配置处理器 |
| `.../handlers/control.ts` | 控制处理器 |
| `.../handlers/file.ts` | 文件处理器 |
| `.../handlers/project.ts` | 项目处理器 |
| `.../handlers/instance.ts` | 实例处理器 |
| `.../handlers/pty.ts` | PTY 处理器（含 WebSocket 连接） |
| `.../handlers/mcp.ts` | MCP 处理器 |
| `.../handlers/question.ts` | 问题处理器 |
| `.../handlers/permission.ts` | 权限处理器 |
| `.../handlers/provider.ts` | 提供商处理器 |
| `.../handlers/sync.ts` | 同步处理器 |
| `.../handlers/tui.ts` | TUI 处理器 |
| `.../handlers/workspace.ts` | 工作区处理器 |
| `.../handlers/experimental.ts` | 实验性功能 |
| `.../handlers/v2.ts` | V2 统一处理器 |
| `.../handlers/v2/session.ts` | V2 会话处理器 |
| `.../handlers/v2/message.ts` | V2 消息处理器 |
| `.../handlers/v2/model.ts` | V2 模型处理器 |
| `.../handlers/v2/provider.ts` | V2 提供商处理器 |

### ACP（Agent Communication Protocol）

| 路径 | 说明 |
|------|------|
| `packages/opencode/src/acp/agent.ts` | ACP v1 Agent |
| `packages/opencode/src/acp/session.ts` | ACP 会话管理 |
| `packages/opencode/src/acp/runtime.ts` | ACP 运行时 |
| `packages/opencode/src/acp/types.ts` | ACP 类型定义 |
| `packages/opencode/src/acp-next/agent.ts` | ACP Next Agent |
| `packages/opencode/src/acp-next/service.ts` | ACP Next 服务 |
| `packages/opencode/src/acp-next/session.ts` | ACP Next 会话 |
| `packages/opencode/src/acp-next/event.ts` | ACP Next 事件 |

### SDK

| 路径 | 说明 |
|------|------|
| `packages/sdk/js/src/client.ts` | V1 客户端（`createOpencodeClient`） |
| `packages/sdk/js/src/v2/client.ts` | **V2 客户端**（推荐） |
| `packages/sdk/js/src/v2/server.ts` | **V2 服务器启动** |
| `packages/sdk/js/src/v2/index.ts` | V2 一站式 `createOpencode()` |
| `packages/sdk/js/src/process.ts` | 子进程管理 |
| `packages/sdk/js/src/gen/` | 自动生成的 SDK 代码 |

### Desktop

| 路径 | 说明 |
|------|------|
| `packages/desktop/src/main/index.ts` | Electron 入口 |
| `packages/desktop/src/main/server.ts` | 侧边服务器进程管理 |
| `packages/desktop/src/main/ipc.ts` | Electron IPC 处理器 |
| `packages/desktop/src/preload/index.ts` | Preload 脚本 |

---

## 限制与注意事项

1. **认证必须配置**：生产环境必须设置 `OPENCODE_SERVER_PASSWORD`，否则服务器不安全
2. **项目路由**：除 `/global/*` 和 `/doc` 外，大多数端点需要 `x-opencode-directory` 头
3. **并发模型**：使用 `InstanceState` + `ScopedCache` 按需加载项目实例，内存随项目数增长
4. **跨域**：可通过 `--cors` 参数或 `server.cors` 配置允许额外域名
5. **WebSocket**：PTY 连接使用 WebSocket，需确保代理/防火墙支持
6. **SSE 流**：事件流为长连接，资源开销与连接数成正比

---

## 参考链接

- [OpenCode GitHub](https://github.com/anomalyco/opencode)
- [OpenCode 文档](https://opencode.ai/docs)
- [npm: @opencode-ai/sdk](https://www.npmjs.com/package/opencode-ai)
- [OpenCode Discord](https://opencode.ai/discord)

---

> 本文档基于 `dev` 分支源码分析生成，具体 API 细节以实际运行时的 `GET /doc` 返回的 OpenAPI 规范为准。
