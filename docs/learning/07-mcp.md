# Codex MCP (Model Context Protocol) 详解

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/codex-mcp/src/`, `codex-rs/core/src/mcp.rs`

## 什么是 MCP？

**MCP (Model Context Protocol)** 是一种让 AI 模型能够访问外部工具和数据的协议。你可以把它想象成"AI 的 USB 接口"：

```
传统 AI:
┌──────────────┐
│    AI 模型    │
└──────────────┘
      │
      ▼
   只有内置能力

MCP 加持:
┌──────────────┐       MCP        ┌──────────────┐
│    AI 模型    │ ◄──────────────► │  MCP Server  │
│              │                  │              │
│              │                  │  GitHub API  │
│              │                  │  数据库      │
│              │                  │  文件系统    │
└──────────────┘                  └──────────────┘
```

---

## MCP 的核心概念

### 1. MCP Server（MCP 服务器）

一个提供特定能力的服务：

```json
// 示例：GitHub MCP Server 配置
{
  "name": "github",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": {
    "GITHUB_TOKEN": "xxx"
  }
}
```

### 2. Tools（工具）

MCP Server 暴露的操作能力：

```json
// GitHub MCP Server 提供的工具
{
  "tools": [
    {
      "name": "create_issue",
      "description": "在 GitHub 上创建 Issue",
      "inputSchema": {
        "owner": "string",
        "repo": "string",
        "title": "string",
        "body": "string"
      }
    },
    {
      "name": "search_repositories",
      "description": "搜索 GitHub 仓库",
      "inputSchema": {
        "query": "string"
      }
    }
  ]
}
```

### 3. Resources（资源）

MCP Server 提供的只读数据：

```json
// GitHub MCP Server 的资源
{
  "resources": [
    {
      "uri": "github://user/repo",
      "name": "Repository Info",
      "mimeType": "application/json"
    }
  ]
}
```

### 4. Prompts（提示模板）

可复用的提示模板：

```json
{
  "prompts": [
    {
      "name": "review_pr",
      "description": "审查一个 GitHub PR",
      "arguments": [
        {"name": "repo", "description": "仓库名称"},
        {"name": "pr_number", "description": "PR 编号"}
      ]
    }
  ]
}
```

---

## Codex 中的 MCP

**文件：** `codex-rs/codex-mcp/src/`

Codex 的 MCP 实现包含多个组件：

### 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **McpRuntime** | `runtime.rs` | MCP 运行时管理 |
| **McpCatalog** | `catalog.rs` | MCP 服务器目录 |
| **McpBinding** | `binding.rs` | 到 MCP 服务器的绑定 |
| **ToolCatalog** | `tool_catalog_cache.rs` | 工具目录缓存 |
| **ResourceClient** | `resource_client.rs` | 资源访问客户端 |

### McpManager

**文件：** `core/src/mcp.rs`

```rust
pub struct McpManager {
    plugins_manager: Arc<PluginsManager>,        // 插件管理器
    extensions: Arc<ExtensionRegistry<Config>>,  // 扩展注册表
    codex_apps_tools_cache: ConnectorRuntimeManager<ToolInfo>,  // Codex Apps 工具缓存
    tool_catalog_cache: McpToolCatalogCache,     // 工具目录缓存
}
```

McpManager 负责：
1. 加载和配置 MCP 服务器
2. 管理 MCP 服务器的生命周期
3. 缓存工具目录
4. 处理 MCP 认证

---

## MCP 工具调用流程

```
用户: "帮我创建一个 GitHub Issue"

    │
    ▼
Codex 分析意图，选择 GitHub 工具
    │
    ▼
通过 MCP 协议调用 GitHub MCP Server
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              MCP 协议通信 (JSON-RPC)                      │
│                                                         │
│  Request:                                               │
│  {                                                      │
│    "jsonrpc": "2.0",                                    │
│    "method": "tools/call",                              │
│    "params": {                                          │
│      "name": "create_issue",                            │
│      "arguments": {                                     │
│        "owner": "myorg",                                │
│        "repo": "myrepo",                                │
│        "title": "Bug Report",                           │
│        "body": "..."                                    │
│      }                                                  │
│    }                                                    │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
    │
    ▼
GitHub MCP Server 处理请求
    │
    ▼
返回结果:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {"type": "text", "text": "Issue created: #123"}
    ]
  }
}
    │
    ▼
Codex 把结果展示给用户
```

---

## MCP 服务器配置

### config.toml 配置

```toml
# ~/.codex/config.toml
[mcp_servers]
# 官方 MCP 服务器
github = {
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-github"]
  env = { GITHUB_TOKEN = "xxx" }
}

filesystem = {
  command = "npx"
  args = ["-y", "@modelcontextprotocol/server-filesystem"]
  env = { }
}

# 自定义 MCP 服务器
my-db = {
  command = "python"
  args = ["/path/to/my/mcp_server.py"]
}
```

### 环境变量

```bash
# MCP 服务器可以访问这些环境变量
export GITHUB_TOKEN=ghp_xxxx
export MCP_SERVER_PATH=/custom/mcp/servers
```

---

## MCP 工具目录

**文件：** `codex-mcp/src/tool_catalog_cache.rs`

Codex 会缓存所有可用 MCP 工具的目录：

```rust
pub struct McpToolCatalogCache {
    /// 缓存的工具列表
    tools: RwLock<Vec<ToolInfo>>,

    /// 最后更新时间
    last_updated: AtomicU64,

    /// 缓存 TTL
    ttl: Duration,
}
```

### 工具发现

```rust
impl McpToolCatalogCache {
    /// 刷新工具目录
    pub async fn refresh(&self) -> Result<()> {
        // 1. 连接到所有配置的 MCP 服务器
        // 2. 调用 tools/list 获取可用工具
        // 3. 缓存工具规范
    }

    /// 搜索工具
    pub async fn search(&self, query: &str) -> Vec<ToolInfo> {
        // 在缓存中搜索匹配的工具
    }
}
```

---

## MCP 资源管理

**文件：** `codex-mcp/src/resource_client.rs`

MCP 服务器可以提供资源给 Codex：

```rust
pub struct McpResourceClient {
    server_name: String,
    http_client: HttpClient,
}
```

### 资源类型

| 类型 | 说明 |
|------|------|
| **静态资源** | 不变的文件/数据 |
| **动态资源** | 需要实时查询的数据 |
| **流式资源** | 长时间更新的数据 |

### 资源读取

```rust
// 通过 MCP 读取资源
let resource = client.read("github://user/repo/README.md").await?;
```

---

## MCP 认证

MCP 服务器可能需要认证：

```rust
pub struct McpAuthStatusEntry {
    pub server_name: String,
    pub auth_status: AuthStatus,
    pub last_checked: DateTime,
}

pub enum AuthStatus {
    Authenticated,
    RequiresLogin,
    Expired,
    Error(String),
}
```

### 认证流程

```
1. 用户配置 MCP 服务器（含认证信息）
        │
        ▼
2. Codex 启动时检查认证状态
        │
        ├─► 已认证 → 正常使用
        │
        ├─► 需要登录 → 发起 OAuth 流程
        │
        └─► 认证过期 → 提示重新认证
```

---

## Codex Apps MCP Server

Codex 内置了一个特殊的 MCP 服务器：**Codex Apps**：

**文件：** `codex-mcp/src/codex_apps.rs`

```rust
pub const CODEX_APPS_MCP_SERVER_NAME: &str = "codex_apps";
```

这个服务器提供：
- 文件系统访问
- Git 操作
- 代码搜索
- 项目分析

---

## MCP 与 Tools 的关系

```
┌─────────────────────────────────────────────────────────┐
│                    Codex 工具层                          │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────┐            │
│  │  内置工具        │    │   MCP 工具       │            │
│  │  (Read, Write)  │    │  (GitHub, DB)   │            │
│  └────────┬────────┘    └────────┬────────┘            │
│           │                      │                      │
│           └──────────┬───────────┘                      │
│                      ▼                                  │
│              ┌──────────────┐                          │
│              │  ToolRouter  │                          │
│              └──────────────┘                          │
└─────────────────────────────────────────────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  MCP Client  │  (统一调用)
              └──────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  GitHub │  │  Files  │  │  数据库  │
   │ Server  │  │  Server │  │  Server │
   └─────────┘  └─────────┘  └─────────┘
```

---

## 总结：MCP 架构

```
┌─────────────────────────────────────────────────────────┐
│                      Codex Core                         │
│                                                         │
│  McpManager ──► 管理 MCP 服务器配置和生命周期              │
│       │                                                  │
│       ▼                                                  │
│  ToolCatalogCache ──► 缓存所有 MCP 工具                   │
│       │                                                  │
│       ▼                                                  │
│  McpBinding ──► 与每个 MCP 服务器的连接                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
                          │
                          │ JSON-RPC over stdio/HTTP
                          ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   GitHub    │  │  Database   │  │  Filesystem │
│   Server    │  │   Server    │  │   Server    │
│             │  │             │  │             │
│ • Issues    │  │ • Query     │  │ • Read      │
│ • PRs       │  │ • Execute   │  │ • Write     │
│ • Actions   │  │ • Tables    │  │ • Glob      │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `codex-mcp/src/runtime.rs` | MCP 运行时 |
| `codex-mcp/src/catalog.rs` | MCP 服务器目录 |
| `codex-mcp/src/binding.rs` | MCP 绑定连接 |
| `codex-mcp/src/tools.rs` | MCP 工具定义 |
| `codex-mcp/src/resource_client.rs` | 资源客户端 |
| `core/src/mcp.rs` | Codex MCP 管理器 |
| `core/src/tools/handlers/mcp.rs` | MCP 工具处理器 |
