# Codex 学习总结与开发指南

> 完成日期：2026-08-23
> 分支：`docs/codex-architecture-notes`

---

## 一、已完成的 12 篇学习笔记

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | `01-overview.md` | 项目概览、架构总览、技术栈 |
| 02 | `02-core-architecture.md` | Session、Agent、Tools、Execution 协作 |
| 03 | `03-cli-and-tui.md` | CLI 入口、TUI 界面、AppServer |
| 04 | `04-tools-system.md` | 工具注册、路由、执行、权限 |
| 05 | `05-sandboxing.md` | 沙箱安全执行、各平台实现 |
| 06 | `06-context-management.md` | Token 预算、历史压缩、TurnContext |
| 07 | `07-mcp.md` | MCP 协议、外部工具集成 |
| 08 | `08-multi-agent.md` | 多 Agent 协作、消息传递 |
| 09 | `09-plugins.md` | 插件系统、市场、生命周期 |
| 10 | `10-exec-server.md` | 远程执行服务器、RPC |
| 11 | `11-authentication.md` | 认证流程、Token 管理 |
| 12 | `12-prompts-and-config.md` | 提示词管理、配置系统 |

---

## 二、Codex 核心架构全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户交互层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  CLI     │  │  TUI     │  │ VS Code  │  │ Desktop  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        └─────────────┴──────┬──────┴─────────────┘
                             │
                    ┌────────▼────────┐
                    │   AppServer     │  ← 认证、多用户、会话路由
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │     Core        │  ← Agent、Session、Tools、Context
                    │                 │
                    │  ┌───────────┐  │
                    │  │   Agent   │  │  ← 多 Agent 协作决策
                    │  └───────────┘  │
                    │  ┌───────────┐  │
                    │  │  Session  │  │  ← 会话生命周期管理
                    │  └───────────┘  │
                    │  ┌───────────┐  │
                    │  │   Tools   │  │  ← 工具注册、路由、执行
                    │  └───────────┘  │
                    │  ┌───────────┐  │
                    │  │  Context  │  │  ← Token 预算、历史压缩
                    │  └───────────┘  │
                    │  ┌───────────┐  │
                    │  │   Hooks   │  │  ← 生命周期钩子
                    │  └───────────┘  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  ToolRouter   │  │  Sandbox      │  │   MCP         │
│               │  │  (安全执行)    │  │   (外部工具)   │
└───────────────┘  └───────────────┘  └───────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ 内置 + MCP    │  │ Linux/macOS/  │  │ GitHub 等     │
│ 工具执行      │  │ Windows 沙箱  │  │ MCP Servers   │
└───────────────┘  └───────────────┘  └───────────────┘
```

---

## 三、核心模块依赖关系

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ThreadManager                                           │
│  - 创建/恢复 Session                                     │
│  - 管理多用户并发                                         │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Session                                                 │
│  - TurnContext (回合上下文)                              │
│  - AgentControl (多 Agent 控制)                          │
│  - McpManager (MCP 管理)                                │
└─────────────────────────────────────────────────────────┘
    │
    ├──────────────────┐
    ▼                  ▼
┌──────────┐    ┌──────────┐
│  Agent   │    │  Tools   │
│  决策    │    │  执行    │
└────┬─────┘    └────┬─────┘
     │               │
     ▼               ▼
┌──────────┐    ┌──────────┐
│ ToolRouter│    │ Sandbox  │
│ 工具路由  │    │ 安全执行 │
└────┬─────┘    └────┬─────┘
     │               │
     └───────┬───────┘
             ▼
      ┌──────────┐
      │ Context  │
      │ 管理     │
      └──────────┘
```

---

## 四、定制化开发路线图

### 阶段 1：环境搭建（1-2 天）

**目标：** 让 Codex 在本地跑起来

```bash
# 1. 克隆代码
git clone https://github.com/openai/codex.git
cd codex

# 2. 安装依赖
just setup  # 或参考 README.md

# 3. 运行测试
just test

# 4. 启动开发模式
just dev
```

### 阶段 2：核心流程（3-5 天）

**目标：** 理解执行流程，尝试修改

| 天数 | 内容 | 关键文件 |
|------|------|---------|
| Day 1 | 走通整个流程 | `core/src/session/turn.rs` |
| Day 2 | 修改提示词 | `prompts/src/`, `config/` |
| Day 3 | 添加简单工具 | `core/src/tools/handlers/` |
| Day 4 | 调试沙箱 | `sandboxing/`, `exec-server/` |
| Day 5 | 配置系统 | `config/schema.rs` |

### 阶段 3：高级定制（1-2 周）

**目标：** 实现自己的功能

| 周数 | 内容 | 关键文件 |
|------|------|---------|
| Week 1 | MCP 集成 | `codex-mcp/src/`, `core/src/mcp.rs` |
| Week 1 | 插件开发 | `core-plugins/src/` |
| Week 2 | 多 Agent | `session/multi_agents.rs` |

### 阶段 4：生产部署

- 认证系统集成
- 多用户支持
- 安全加固
- 性能优化

---

## 五、常见定制场景

### 场景 1：添加新工具

```rust
// 1. 在 handlers/ 创建新工具
// core/src/tools/handlers/my_tool.rs

pub struct MyToolHandler;

impl ToolExecutor for MyToolHandler {
    async fn execute(&self, args: MyToolArgs) -> Result<String> {
        // 工具逻辑
        Ok("result".to_string())
    }
}

// 2. 在 mod.rs 注册
pub mod my_tool;

// 3. 添加工具规范
pub const MY_TOOL_SPEC: ToolSpec = ToolSpec {
    name: "my_tool",
    description: "我的自定义工具",
    parameters: [...],
};
```

### 场景 2：修改提示词

```rust
// prompts/src/my_feature.rs

pub const MY_FEATURE_PROMPT: &str = r#"
你是一个 XX 领域的专家...
"#;

// 在 TurnContext 创建时注入
```

### 场景 3：添加 MCP 服务器

```toml
# config.toml
[mcp_servers]
my_server = {
    command = "npx"
    args = ["-y", "@my/mcp-server"]
}
```

### 场景 4：自定义权限

```toml
# config.toml
[permissions]
allow_read = ["~/code/**", "/project/**"]
allow_write = ["~/code/**/*.py"]
allow_exec = ["git", "python", "npm"]
deny = ["~/.ssh/**", "/etc/**"]
```

---

## 六、关键文件速查表

| 功能 | 文件 |
|------|------|
| CLI 入口 | `cli/src/main.rs` |
| TUI 主循环 | `tui/src/app.rs` |
| Session 管理 | `core/src/session/session.rs` |
| Turn 执行 | `core/src/session/turn.rs` |
| Agent 控制 | `core/src/agent/control.rs` |
| 工具路由 | `core/src/tools/router.rs` |
| 工具注册 | `core/src/tools/registry.rs` |
| 沙箱执行 | `core/src/sandboxing/mod.rs` |
| 上下文管理 | `core/src/context_manager/history.rs` |
| Token 预算 | `core/src/session/token_budget.rs` |
| MCP 管理 | `core/src/mcp.rs` |
| 多 Agent | `core/src/session/multi_agents.rs` |
| 插件系统 | `core-plugins/src/manager.rs` |
| 认证 | `login/src/auth.rs` |
| 提示词 | `prompts/src/*.rs` |
| 配置 | `core/src/config/mod.rs` |

---

## 七、调试技巧

### 1. 启用详细日志

```bash
RUST_LOG=debug codex
```

### 2. 查看 Token 使用

Codex 会在状态栏显示：
```
tokens: 12,450 / 128,000  (9.7%)
```

### 3. 沙箱调试

```bash
# 查看沙箱日志
codex doctor --sandbox

# 测试沙箱执行
codex exec --sandbox-enabled "echo hello"
```

### 4. 工具调用追踪

```bash
# 启用工具追踪
CODEX_TRACE_TOOLS=true codex
```

---

## 八、学习资源

### 1. 官方文档

- README.md - 项目说明
- AGENTS.md - Agent 开发指南
- docs/ - 详细文档

### 2. 测试代码

- `core/src/**/*_tests.rs` - 单元测试
- `tests/` - 集成测试

### 3. 构建命令

```bash
just -l  # 列出所有命令
just test        # 运行测试
just fmt         # 格式化代码
just clippy      # 代码检查
just build       # 构建
```

---

## 九、下一步

你现在已经掌握了 Codex 的核心架构，可以：

1. **深入某个模块** — 根据兴趣选择特定模块深入研究
2. **尝试修改代码** — 从简单的提示词修改开始
3. **添加新功能** — 按照路线图逐步实现
4. **贡献上游** — 修复 Bug 或添加功能

---

**祝你学习愉快！**
