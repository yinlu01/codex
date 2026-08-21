# Codex 核心架构详解

> 深入学习日期：2026-08-21
> 对应代码：`codex-rs/core/src/`

## 什么是"核心"？

`codex-rs/core/` 是 Codex 的大脑。所有用户请求（"帮我写个函数"、"帮我 debug这段代码"）都经过这里处理。它负责：
1. **理解用户想做什么**（Agent 逻辑）
2. **决定使用什么工具**（Tools 路由）
3. **管理对话上下文**（Session/Context 管理）
4. **执行代码并返回结果**（Execution）

## 核心模块一览

```
codex-rs/core/src/
├── agent/           # Agent 控制：多线程/子 Agent 管理
├── session/         # 会话管理：一对一对话的生命周期
├── tools/           # 工具注册、路由、生命周期
├── context/         # 上下文管理：历史记录、token 预算
├── thread_manager/  # 线程管理器：所有会话的顶层协调
├── exec.rs          # 代码执行：沙箱中运行命令
├── compact.rs       # 压缩历史：当上下文太长时精简
├── mcp.rs           # MCP 协议：Model Context Protocol
└── ... 其他支持模块
```

## 1. Session（会话）— 一场对话的生命周期

**文件：** `session/session.rs`

当你对 Codex 说"帮我写个函数"，Codex 会创建一个 **Session（会话）**。

### Session 的核心职责

```rust
pub(crate) struct Session {
    pub(crate) thread_id: ThreadId,           // 唯一标识这场对话
    pub(crate) conversation: Arc<RealtimeConversationManager>,  // 对话内容
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>,  // 当前正在执行的任务
    pub(crate) services: SessionServices,     // 各种服务（模型、工具、MCP等）
    // ...
}
```

### 关键概念：Turn（回合）

一次"用户提问 + Codex 回答"叫做一个 **Turn（回合）**。

```
用户: "帮我写个 hello world"
       ↓
   [Turn 1]
       ↓
Codex: (执行工具，生成代码)
       ↓
用户: "改成函数"
       ↓
   [Turn 2]
       ↓
Codex: (修改代码)
```

每个 Turn 有：
- **输入**（用户说了什么）
- **执行过程**（调用了哪些工具）
- **输出**（返回什么结果）

### Session 的生命周期

```
Session 创建
    ↓
等待用户输入 (InputQueue)
    ↓
创建 Turn → 执行 → 返回结果
    ↓
可以继续对话 或 结束
```

---

## 2. ThreadManager（线程管理器）— 所有会话的指挥塔

**文件：** `thread_manager.rs`

Codex 不只处理一场对话。它同时管理多个用户的多个会话。**ThreadManager** 就是这个管理员。

### ThreadManager 的核心职责

```rust
pub(crate) struct ThreadManagerState {
    pub sessions: HashMap<ThreadId, Arc<Session>>,  // 所有活跃会话
    pub thread_store: Arc<dyn ThreadStore>,          // 持久化存储
    // ...
}
```

### 它做什么？

1. **创建新会话** - 当用户开始新对话时
2. **恢复旧会话** - 用户中断后回来继续
3. **管理并发** - 同时处理多个用户的请求
4. **持久化** - 保存对话历史到磁盘

---

## 3. Agent（代理）— 决策的大脑

**文件：** `agent/control.rs`, `agent/registry.rs`

**Agent** 是 Codex 的决策核心。它决定：
- 这个任务需要几个子 Agent？
- 每个 Agent 做什么？
- 结果如何汇总？

### 为什么需要多 Agent？

复杂任务可以分解给多个专业 Agent：

```
用户: "帮我做数据分析并画图"
         │
         ├──────────────────────┐
         ▼                      ▼
    [Data Agent]          [Chart Agent]
    (负责查数据)           (负责画图)
         │                      │
         └──────────────────────┘
                      │
                      ▼
                 [主Agent]
              (整合结果返回用户)
```

### AgentRegistry - 管理所有 Agent

```rust
pub(crate) struct AgentRegistry {
    active_agents: Mutex<ActiveAgents>,  // 正在运行的 Agent
    total_count: AtomicUsize,            // 总数统计
}
```

防止无限制创建子 Agent（资源保护）。

---

## 4. Tools（工具）— Codex 的能力扩展

**文件：** `tools/mod.rs`, `tools/registry.rs`

Codex 不只是聊天，它能**执行操作**（读写文件、运行命令等）。这些能力叫做 **Tools（工具）**。

### 工具类型

| 工具类型 | 示例 | 说明 |
|---------|------|------|
| **内置工具** | `Read`, `Write`, `Bash` | Codex 自带的核心能力 |
| **MCP 工具** | 文件搜索、数据库访问 | 通过 MCP 协议扩展 |
| **Hosted 工具** | GitHub API、Slack | 托管的外部服务 |
| **Skill 工具** | 自定义脚本 | 用户编写的自动化脚本 |

### 工具如何工作？

```
用户: "帮我读取 app.py 文件"
         │
         ▼
   [ToolRouter]  ← 工具路由器
         │
         ▼
   匹配到 "Read" 工具
         │
         ▼
   [ToolRegistry]  ← 查找工具实现
         │
         ▼
   执行 Read 逻辑
         │
         ▼
   返回文件内容
```

---

## 5. Context Management（上下文管理）

**文件：** `context_manager/mod.rs`, `session/turn_context.rs`

LLM 有 **token 限制**（比如 128K tokens）。对话太长会超出限制。**Context Management** 负责：
1. **追踪** - 当前用了多少 tokens
2. **压缩** - 当太满时精简历史
3. **截断** - 必要时丢弃最旧的内容

### Token 预算管理

```rust
pub(crate) struct TurnContext {
    pub model_info: ModelInfo,        // 模型信息（上下文窗口大小）
    pub token_budget: TokenBudget,    // 剩余可用 token
    pub history: Vec<RolloutItem>,    // 对话历史
    // ...
}
```

---

## 6. Execution（执行）— 沙箱中运行代码

**文件：** `exec.rs`, `sandboxing/`

Codex 在 **沙箱（Sandbox）** 中执行代码，保证安全：

```
┌─────────────────────────────────┐
│         你的电脑 (Host)          │
│                                 │
│  ┌───────────────────────────┐  │
│  │     Sandbox（沙箱）        │  │
│  │                           │  │
│  │   Codex 在这里执行代码     │  │
│  │   没有权限访问真实系统     │  │
│  │                           │  │
│  └───────────────────────────┘  │
│                                 │
└─────────────────────────────────┘
```

沙箱提供：
- **隔离** - 代码不会影响真实系统
- **权限控制** - 只能访问允许的文件
- **资源限制** - CPU、内存、时间的上限

---

## 模块协作流程

```
用户输入
    │
    ▼
ThreadManager ─── 创建/找到 Session
    │
    ▼
Session ─── 解析用户输入
    │
    ├──► Agent ─── 决定如何执行
    │         │
    │         ▼
    │    ToolRouter ─── 选择工具
    │         │
    │         ▼
    └──────► Tools ─── 执行具体操作
                      │
                      ▼
                 Sandbox (安全执行)
                      │
                      ▼
                 返回结果
                      │
                      ▼
              ContextManager ─── 更新上下文
                      │
                      ▼
                 返回给用户
```

---

## 核心概念总结

| 概念 | 文件位置 | 职责 |
|------|---------|------|
| **Session** | `session/session.rs` | 单个对话的生命周期管理 |
| **ThreadManager** | `thread_manager.rs` | 所有会话的顶层管理 |
| **Agent** | `agent/control.rs` | 多 Agent 协作决策 |
| **ToolRouter** | `tools/router.rs` | 工具选择和路由 |
| **TurnContext** | `session/turn_context.rs` | 单个回合的上下文 |
| **ContextManager** | `context_manager/mod.rs` | token 预算和历史压缩 |
| **Sandbox** | `sandboxing/` | 代码的安全执行环境 |

---

## 关键代码文件速查

| 如果你想看... | 去这里 |
|-------------|--------|
| Session 的完整定义 | `session/session.rs` |
| Turn 如何执行 | `session/turn.rs` |
| 工具如何注册和调用 | `tools/registry.rs`, `tools/router.rs` |
| Agent 如何做决策 | `agent/control.rs` |
| Token 预算如何管理 | `session/token_budget.rs` |
| 沙箱如何工作 | `sandboxing/mod.rs` |
| 多 Agent 如何协作 | `session/multi_agents.rs` |
