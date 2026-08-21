# Codex CLI 和 TUI（终端界面）

> 深入学习日期：2026-08-21
> 对应代码：`codex-rs/cli/src/main.rs`, `codex-rs/tui/src/`

## CLI 入口 — 用户如何启动 Codex

**文件：** `codex-rs/cli/src/main.rs`

当你运行 `codex` 命令时，实际执行的是这个 Rust 程序。

### 主命令结构

```rust
// codex-rs/cli/src/main.rs (简化)
fn main() {
    // 解析命令行参数
    let matches = Command::parse();

    match matches.subcommand() {
        "tui" => run_tui(),        // 交互式界面
        "exec" => run_exec(),      // 执行命令模式
        "login" => run_login(),    // 登录
        "app" => run_desktop(),    // 桌面应用
        // ...
    }
}
```

### 支持的命令

| 命令 | 用途 |
|------|------|
| `codex` (无参数) | 启动交互式 TUI |
| `codex tui` | 显式启动 TUI |
| `codex exec <command>` | 直接执行单条命令 |
| `codex login` | 登录认证 |
| `codex app` | 启动桌面应用模式 |
| `codex mcp` | MCP 服务器管理 |
| `codex plugin` | 插件管理 |
| `codex doctor` | 诊断检查 |

### 平台特定入口

在 macOS/Windows 上，还有一个 `codex app` 命令启动完整的桌面应用：

```
macOS/Windows:
├── codex         → TUI 模式（终端界面）
├── codex app     → 桌面应用模式（窗口 GUI）
└── VS Code 插件  → VS Code 内嵌模式
```

---

## TUI（终端用户界面）

**文件：** `codex-rs/tui/src/app.rs`

TUI 是 Codex 的**交互式终端界面**，用 Rust 编写，支持丰富的文本渲染。

### 为什么用 Rust 写 TUI？

常见的 TUI 库有 `cursive`、`ratatui`、`tui-rs`，但 Codex 选择 Rust 是因为：
1. **性能** - 和 core 同一个语言，零成本交互
2. **一致性** - 整个项目统一 Rust 技术栈
3. **自定义** - 需要深度定制渲染行为

### TUI 的布局结构

```
┌────────────────────────────────────────────────────────┐
│  ▌ ● ●  codex — zsh — 120×40              [████████] │  ← 标题栏
├────────────────────────────────────────────────────────┤
│                                                        │
│  Welcome to Codex!                                     │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │  Hello! I'm Codex, your coding assistant.      │  │
│  │  What would you like to work on today?         │  │
│  │                                                  │  │
│  │  Try:                                           │  │
│  │  • "Write a Python function to sort a list"    │  │
│  │  • "Explain this code"                         │  │
│  │  • "Find and fix bugs in my project"           │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  > Write a hello world in Python                      │  ← 用户输入
│                                                        │
├────────────────────────────────────────────────────────┤
│  [Turn 1]  ████████████░░░░░░░░  63%  tokens: 12,450  │  ← 状态栏
└────────────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **App** | `app.rs` | 主应用状态和事件循环 |
| **ChatWidget** | `chatwidget.rs` | 聊天消息显示 |
| **BottomPane** | `bottom_pane/` | 底部面板（文件树、终端等） |
| **AppCommand** | `app_command.rs` | 用户命令处理 |

### TUI 的事件循环

```rust
// 简化的 TUI 事件循环
loop {
    match event {
        Event::KeyPress(key) => {
            // 处理键盘输入
            handle_keypress(key);
        }
        Event::Mouse(mouse_event) => {
            // 处理鼠标事件
            handle_mouse(mouse_event);
        }
        Event::Resize(width, height) => {
            // 处理终端大小变化
            handle_resize(width, height);
        }
        _ => {}
    }

    // 渲染界面
    render();

    // 检查是否退出
    if should_exit() {
        break;
    }
}
```

### 和 Core 的交互

```
┌─────────────┐       gRPC/WebSocket        ┌─────────────┐
│    TUI      │ ◄─────────────────────────► │  AppServer  │
│  (前端渲染)  │                             │  (后端逻辑)  │
└─────────────┘                             └─────────────┘
                                                  │
                                                  ▼
                                            ┌─────────────┐
                                            │    Core     │
                                            │  (真正处理)  │
                                            └─────────────┘
```

TUI 不直接调用 Core，而是通过 **AppServer** 通信：
- TUI 发送用户输入 → AppServer
- AppServer 处理 → 返回结果
- TUI 渲染结果

---

## AppServer（应用服务器）

**文件：** `codex-rs/app-server/`

AppServer 是 Codex 的**后端服务**，处理：
- 用户认证
- 会话管理
- 多用户支持
- 权限控制

### 为什么需要独立的服务？

```
简单模式（单机）:
  codex <-> Core (直接调用)

生产模式（多用户）:
  codex <-> AppServer <-> Core
                │
                ├── 用户A的会话
                ├── 用户B的会话
                └── 用户C的会话
```

### AppServer 职责

| 职责 | 说明 |
|------|------|
| **认证** | 用户登录、API Key 管理 |
| **会话路由** | 把请求分发给正确的 Session |
| **多用户** | 支持多个用户同时使用 |
| **持久化** | 保存对话历史到数据库 |

---

## 命令执行流程

当你运行 `codex exec "ls -la"` 时：

```
┌──────────────────────────────────────────────────────────────┐
│                         流程                                  │
└──────────────────────────────────────────────────────────────┘

1. codex exec "ls -la"
       │
       ▼
2. CLI 解析命令 → ExecCommand
       │
       ▼
3. 创建 ExecCli，连接到 AppServer 或直接调用 Core
       │
       ▼
4. Core 创建 Session，执行命令
       │
       ▼
5. ExecServer 在沙箱中运行 "ls -la"
       │
       ▼
6. 返回结果给 CLI
       │
       ▼
7. CLI 打印输出，退出
```

### exec-server（执行服务器）

**文件：** `codex-rs/exec-server/`

专门的沙箱执行环境，用于运行用户命令：

```rust
// 简化的 exec 流程
pub async fn execute_command(cmd: &str) -> ExecResult {
    // 1. 验证命令安全性
    let policy = ExecPolicy::check(cmd)?;

    // 2. 在沙箱中执行
    let sandbox = Sandbox::new();
    let output = sandbox.run(cmd).await;

    // 3. 返回结果
    Ok(output)
}
```

---

## 认证流程

Codex 支持多种登录方式：

| 方式 | 说明 |
|------|------|
| **API Key** | 直接使用 OpenAI API Key |
| **Device Code** | OAuth 设备码流程（浏览器登录） |
| **Workload Identity** | 云环境自动认证 |
| **ChatGPT Plus** | 使用 ChatGPT 订阅 |

```
codex login
    │
    ├──► API Key 模式: 直接输入 key
    │
    ├──► Device Code: 打开浏览器验证
    │
    └──► ChatGPT: OAuth 登录
```

---

## 配置文件

Codex 的配置在 `~/.codex/` 目录：

```
~/.codex/
├── config.toml       # 用户配置
├── sessions/         # 会话历史
├── plugins/          # 插件
├── credentials/      # 认证凭据（加密存储）
└── logs/             # 日志
```

---

## 总结：架构层次

```
┌────────────────────────────────────────────────────────────┐
│                      用户交互层                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  CLI     │  │  TUI     │  │ VS Code  │  │ Desktop  │   │
│  │ (命令)   │  │ (终端UI) │  │  插件    │  │  应用    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        └─────────────┴──────┬──────┴─────────────┘
                             │
                    ┌────────▼────────┐
                    │   AppServer     │
                    │  (用户认证/路由)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │     Core        │
                    │  (Agent/工具/   │
                    │   执行/上下文)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  ExecServer     │
                    │  (沙箱执行)      │
                    └─────────────────┘
```
