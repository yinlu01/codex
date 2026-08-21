# Codex Tools（工具）系统详解

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/core/src/tools/`

## 什么是 Tools？

**Tools（工具）** 是 Codex 执行操作的能力。当用户说"帮我读取文件"，Codex 实际上是在调用 `Read` 工具。

```
用户: "帮我读取 app.py 并输出内容"

Codex 思考: "用户想读取文件，我需要调用 Read 工具"

实际执行:
  Tool: Read
  Arguments: {"path": "app.py"}
  │
  ▼
┌─────────────────────────────────────┐
│           工具执行                    │
│  1. 解析参数                         │
│  2. 在沙箱中读取文件                   │
│  3. 返回文件内容                      │
└─────────────────────────────────────┘
```

---

## 工具类型

Codex 支持多种类型的工具：

### 1. 内置工具（Built-in Tools）

Core 自带的核心工具，不依赖外部服务：

| 工具名 | 功能 | 对应代码 |
|--------|------|---------|
| **Read** | 读取文件内容 | `handlers/read.rs` |
| **Write** | 写入文件内容 | `handlers/write.rs` |
| **Bash/Exec** | 执行 shell 命令 | `handlers/unified_exec.rs` |
| **Glob** | 文件搜索（通配符） | `handlers/glob.rs` |
| **Grep** | 文本搜索 | `handlers/grep.rs` |
| **Edit** | 编辑文件 | `handlers/edit.rs` |
| **Notebook** | Jupyter notebook 操作 | `handlers/notebook.rs` |

### 2. MCP 工具（MCP Tools）

通过 **Model Context Protocol** 扩展的工具：

```
Codex ──MCP──► MCP Server ──► 外部服务
                 │
                 ├── GitHub API
                 ├── Filesystem
                 ├── Database
                 └── 任何外部系统
```

**文件位置：** `handlers/mcp.rs`

### 3. Skill 工具

用户编写的自定义脚本或函数：

```
用户定义 Skill: "my数据分析脚本.py"
    │
    ▼
Codex 执行这个脚本，返回结果
```

### 4. Hosted 工具

Codex 托管的外部服务集成：

| 工具 | 服务 |
|------|------|
| `github_*` | GitHub API 操作 |
| `slack_*` | Slack 消息发送 |

---

## 工具执行流程

```
用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ToolRouter (工具路由器)                                 │
│                                                         │
│  1. 解析 LLM 返回的工具调用                              │
│  2. 查找工具注册表                                        │
│  3. 匹配工具名 → 找到对应的 Handler                       │
│  4. 传递参数给 Handler                                   │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ToolHandler (工具处理器)                                 │
│                                                         │
│  1. 验证参数                                             │
│  2. 检查权限（用户是否允许执行这个操作）                    │
│  3. 在沙箱中执行                                          │
│  4. 收集输出                                             │
│  5. 返回结果                                             │
└─────────────────────────────────────────────────────────┘
    │
    ▼
结果返回给 LLM，继续对话
```

---

## 核心组件

### ToolRouter（工具路由器）

**文件：** `tools/router.rs`

负责把 LLM 的请求路由到正确的工具处理器：

```rust
pub struct ToolRouter {
    registry: ToolRegistry,           // 所有已注册的工具
    model_visible_specs: Arc<[ToolSpec]>,  // 对 LLM 可见的工具规范
}
```

核心方法：
- `route()` - 根据工具名找到对应处理器
- `list_tools()` - 列出所有可用工具
- `search_tools()` - 搜索匹配的工具

### ToolRegistry（工具注册表）

**文件：** `tools/registry.rs`

所有可用工具的注册表，维护工具元数据：

```rust
pub struct ToolRegistry {
    tools: BTreeMap<ToolName, ToolSpec>,  // 工具名 → 工具规范
    runtimes: Vec<Arc<dyn CoreToolRuntime>>,  // 工具运行时
}
```

### ToolHandler（工具处理器）

每种工具都有对应的 Handler，负责实际执行：

```rust
// 示例：Read 工具的 Handler 简化版
pub struct ReadHandler {
    sandbox: Arc<Sandbox>,  // 沙箱引用
}

impl ToolExecutor for ReadHandler {
    async fn execute(&self, args: ReadArgs) -> Result<String> {
        // 1. 验证路径安全性
        let safe_path = self.sandbox.validate_path(args.path)?;

        // 2. 在沙箱中读取文件
        let content = self.sandbox.read_file(safe_path).await?;

        // 3. 返回结果
        Ok(content)
    }
}
```

---

## 工具规范（ToolSpec）

每种工具都有一个规范，描述：
- 工具名称和描述
- 参数列表和类型
- 返回值类型
- 使用示例

```rust
pub struct ToolSpec {
    pub name: String,           // "Read"
    pub description: String,    // "读取文件内容"
    pub parameters: Vec<ParamSpec>,  // 参数定义
    pub returns: ReturnSpec,    // 返回值定义
}
```

示例：一个 "Read" 工具的规范可能像：

```json
{
  "name": "Read",
  "description": "读取指定路径的文件内容",
  "parameters": {
    "path": {
      "type": "string",
      "description": "要读取的文件路径",
      "required": true
    },
    "start_line": {
      "type": "number",
      "description": "起始行号（可选）",
      "required": false
    },
    "end_line": {
      "type": "number",
      "description": "结束行号（可选）",
      "required": false
    }
  },
  "returns": {
    "type": "string",
    "description": "文件的文本内容"
  }
}
```

---

## 执行命令工具（Exec）

**文件：** `handlers/unified_exec.rs`

最重要的工具之一，允许执行 shell 命令：

```rust
pub struct ExecCommandArgs {
    pub cmd: String,           // 要执行的命令
    pub shell: Option<String>, // 使用哪个 shell (bash/zsh/powershell)
    pub login: Option<bool>,   // 是否是 login shell
    pub tty: bool,             // 是否分配伪终端
    pub workdir: Option<String>,  // 工作目录
    pub max_output_tokens: Option<usize>,  // 最大输出 token 数
    pub sandbox_permissions: Option<SandboxPermissions>,  // 沙箱权限
}
```

### 执行流程

```
用户: "运行 ls -la"

Codex 解析为:
  Tool: Exec
  Args: {cmd: "ls -la", workdir: "/project"}

    │
    ▼
ToolRouter 找到 ExecHandler

    │
    ▼
ExecHandler 验证:
  1. 命令安全性检查（exec_policy）
  2. 工作目录权限
  3. 是否在允许列表中

    │
    ▼
沙箱执行:
  sandbox.run("ls -la", workdir="/project")
    │
    ▼
返回执行结果:
  {
    stdout: "total 12\ndrwxr-xr-x  3 user staff  96 Aug 22 app.py",
    stderr: "",
    exit_code: 0
  }
```

---

## 权限控制

Codex 有严格的权限控制，防止恶意操作：

### 1. 执行策略（Exec Policy）

```rust
pub enum ExecPolicyLevel {
    Allow,      // 允许执行
    Audit,      // 记录但不阻止
    Deny,       // 阻止执行
}
```

### 2. 权限配置示例

```toml
# ~/.codex/config.toml
[permissions]
# 允许读取的目录
allow_read = ["~/code", "/project/**"]
# 允许写入的目录
allow_write = ["~/code/**"]
# 允许执行的命令
allow_exec = ["git", "npm", "cargo"]
# 禁止的命令
deny_exec = ["rm -rf /", "dd if=/dev/zero"]
```

### 3. 沙箱权限

```rust
pub struct SandboxPermissions {
    pub read: bool,    // 允许读取文件
    pub write: bool,   // 允许写入文件
    pub exec: bool,    // 允许执行命令
    pub network: bool, // 允许网络访问
}
```

---

## MCP 集成

**MCP (Model Context Protocol)** 是一种扩展 Codex 能力的协议：

```
┌──────────────┐       MCP        ┌──────────────┐
│    Codex     │ ◄──────────────► │  MCP Server  │
│              │                  │              │
│  内置工具集   │                  │  GitHub      │
│              │                  │  Filesystem  │
│              │                  │  Database    │
└──────────────┘                  └──────────────┘
```

### MCP 工具的注册

```rust
// handlers/mcp.rs
pub async fn register_mcp_tools(
    server_name: &str,
    client: &McpClient,
) -> Result<Vec<ToolSpec>> {
    // 1. 连接到 MCP 服务器
    let tools = client.list_tools().await?;

    // 2. 转换为 Codex 工具规范
    let specs: Vec<ToolSpec> = tools
        .into_iter()
        .map(convert_to_codex_spec)
        .collect();

    // 3. 注册到 ToolRegistry
    registry.register_many(specs).await?;

    Ok(specs)
}
```

---

## 工具搜索

Codex 可以搜索合适的工具来完成任务：

```rust
// 当用户描述一个操作但没指定工具时
User: "我需要整理一下这个文件夹"

Codex 思考:
  "用户想要整理文件夹，可能需要：
   - Bash tool (用 rm/mv 命令)
   - 或专门的整理工具"

  执行工具搜索:
  search(query="organize files")
  │
  ▼
返回匹配的工具列表:
  - Bash (执行 shell 命令)
  - Glob (查找文件)
  - ...
```

---

## 总结：工具调用链路

```
LLM 返回工具调用
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ToolRouter::route()                                    │
│  - 解析 ToolCall {name, args, call_id}                  │
│  - 从 registry 查找工具规范                              │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ToolHandler::execute()                                 │
│  - 解析参数                                             │
│  - 权限检查                                             │
│  - 沙箱执行                                             │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  结果处理                                               │
│  - 格式化输出                                           │
│  - 执行 hooks (pre/post tool use)                      │
│  - 记录分析数据                                         │
└─────────────────────────────────────────────────────────┘
    │
    ▼
结果返回给 LLM，继续下一轮对话
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `tools/registry.rs` | 工具注册表 |
| `tools/router.rs` | 工具路由 |
| `tools/handlers/unified_exec.rs` | Shell 命令执行 |
| `tools/handlers/mcp.rs` | MCP 工具处理 |
| `tools/handlers/read.rs` | 文件读取 |
| `tools/handlers/write.rs` | 文件写入 |
| `tools/spec_plan.rs` | 工具规范生成 |
| `tools/lifecycle.rs` | 工具生命周期钩子 |
