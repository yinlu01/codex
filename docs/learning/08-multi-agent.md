# Codex Multi-Agent（多代理协作）

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/core/src/session/multi_agents.rs`, `codex-rs/core/src/agent/`

## 为什么需要多代理？

单代理的局限性：

```
用户: "帮我分析这个项目，然后写测试，最后提交 PR"

单代理处理:
  Agent 一个人做所有事:
  1. 分析代码结构
  2. 理解业务逻辑
  3. 编写测试用例
  4. 执行测试
  5. 提交 PR

问题：
- 一个人做太多事，效率低
- 上下文太长，token 消耗大
- 难以并行处理
```

**多代理** 把任务分配给多个专业 Agent：

```
用户: "帮我分析这个项目，然后写测试，最后提交 PR"

多代理处理:
  ┌─────────────────────────────────────────┐
  │  Root Agent (主代理)                      │
  │  负责任务分解和协调                        │
  └──────────────────┬──────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ 分析 Agent │ │ 测试 Agent │ │ Git Agent │
  │          │ │          │ │          │
  │ 分析代码  │ │ 编写测试  │ │ 提交 PR  │
  │ 结构      │ │          │ │          │
  └────┬─────┘ └────┬─────┘ └────┬─────┘
       │            │            │
       └────────────┴────────────┘
                     │
                     ▼
              结果汇总给用户
```

---

## Multi-Agent 架构

### 核心概念

| 概念 | 说明 |
|------|------|
| **Root Agent** | 主代理，任务分解和协调者 |
| **Sub-Agent** | 子代理，执行具体子任务 |
| **spawn_agent** | 创建新的子代理 |
| **send_message** | 向其他代理发送消息 |
| **followup_task** | 给已有代理分配新任务 |
| **wait_agent** | 等待某个代理完成 |

### Agent 层级

```
/root                    ← 根代理（主对话）
│
├── /root/analyst        ← 分析代理
│   └── /root/analyst/code   ← 分析代理的子代理
│
├── /root/tester         ← 测试代理
│
└── /root/git            ← Git 代理
```

---

## Agent 通信协议

### 消息类型

| 类型 | 说明 |
|------|------|
| **NEW_TASK** | 分配新任务 |
| **MESSAGE** | 发送消息 |
| **FINAL_ANSWER** | 返回最终结果 |

### 消息格式

```json
{
  "type": "MESSAGE",
  "to": "/root/tester",           // 目标代理
  "from": "/root",                // 来源代理
  "payload": "请为 src/utils.ts 编写单元测试",
  "task_id": "task-123"
}
```

---

## spawn_agent 工具

创建新的子代理：

```rust
pub struct SpawnAgentArgs {
    /// 子代理的名字/标识
    pub name: String,

    /// 子代理的角色描述
    pub role: String,

    /// 传递的历史范围
    pub fork_turns: ForkTurns,

    /// 使用哪个模型
    pub model: Option<String>,

    /// 推理努力程度
    pub reasoning_effort: Option<ReasoningEffort>,
}

pub enum ForkTurns {
    All,           // 传递全部历史
    None,          // 不传递历史
    LastN(usize),  // 只传递最近 N 轮
}
```

### 使用示例

```
用户: "帮我并行分析这三个文件的逻辑"

Codex 分解任务:
  1. spawn_agent("analyst_file_a", "分析 file_a.ts", fork_turns="none")
  2. spawn_agent("analyst_file_b", "分析 file_b.ts", fork_turns="none")
  3. spawn_agent("analyst_file_c", "分析 file_c.ts", fork_turns="none")

  三个代理并行工作，结果汇总
```

---

## Agent 控制

**文件：** `core/src/agent/control.rs`

```rust
pub struct AgentControl {
    registry: Arc<AgentRegistry>,  // Agent 注册表
    active_agents: Mutex<HashMap<ThreadId, LiveAgent>>,  // 活跃代理
}

pub struct LiveAgent {
    pub thread_id: ThreadId,
    pub metadata: AgentMetadata,
    pub status: AgentStatus,
}
```

### AgentRegistry

**文件：** `core/src/agent/registry.rs`

管理所有 Agent 的注册和限制：

```rust
pub struct AgentRegistry {
    /// 当前活跃的 Agent
    active_agents: Mutex<ActiveAgents>,

    /// Agent 总数
    total_count: AtomicUsize,
}

impl AgentRegistry {
    /// 预留 Agent 槽位
    pub fn reserve_spawn_slot(&self, max_threads: Option<usize>) -> Result<SpawnReservation>;

    /// 释放 Agent 槽位
    pub fn release_spawn_slot(&self, reservation: SpawnReservation);

    /// 检查是否超过限制
    pub fn exceeds_thread_spawn_depth_limit(&self, depth: i32) -> bool;
}
```

---

## 消息传递

### send_message

向运行中的代理发送消息（不触发新 turn）：

```json
// 发送消息
{
  "tool": "send_message",
  "args": {
    "to": "/root/tester",
    "content": "分析完成了，这是结果..."
  }
}
```

### followup_task

给代理分配新任务并触发 turn：

```json
// 分配新任务
{
  "tool": "followup_task",
  "args": {
    "to": "/root/tester",
    "task": "现在为分析出的函数编写测试",
    "context": "分析结果包含 5 个函数..."
  }
}
```

### wait_agent

等待代理完成：

```rust
// 等待代理完成
{
  "tool": "wait_agent",
  "args": {
    "agent": "/root/tester",
    "timeout_seconds": 300
  }
}
```

---

## 多代理执行流程

```
用户输入
    │
    ▼
Root Agent 接收任务
    │
    ▼
任务分析
    │
    ├─► 简单任务 → 直接执行
    │
    └─► 复杂任务 → 分解子任务
            │
            ▼
    spawn_agent("analyst") ──► spawn_agent("tester") ──► spawn_agent("git")
            │                          │                        │
            ▼                          ▼                        ▼
      分析代码                    编写测试                  提交 PR
            │                          │                        │
            └──────────────────────────┼────────────────────────┘
                                        │
                                        ▼
                                  wait_agent
                                  等待所有子代理完成
                                        │
                                        ▼
                                  汇总结果
                                        │
                                        ▼
                                  返回给用户
```

---

## Agent 状态

| 状态 | 说明 |
|------|------|
| **Initializing** | 正在初始化 |
| **Running** | 运行中 |
| **Waiting** | 等待消息/任务 |
| **Completed** | 已完成 |
| **Error** | 出错 |
| **Interrupted** | 被中断 |

---

## 协作工具命名空间

协作工具在 `functions.collaboration` 命名空间：

```
functions.collaboration.spawn_agent     // 创建新代理
functions.collaboration.send_message    // 发送消息
functions.collaboration.followup_task   // 分配任务
functions.collaboration.wait_agent      // 等待代理
functions.collaboration.list_agents     // 列出代理
functions.collaboration.interrupt_agent // 中断代理
```

---

## 配置

### config.toml

```toml
[multi_agent]
# 启用多代理
enabled = true

# 最大子代理数量
max_sub_agents = 10

# 最大 Agent 层级深度
max_depth = 5

# 默认模型
default_model = "gpt-4o"

# 推理努力程度
reasoning_effort = "medium"
```

### AGENTS.md 中的角色定义

```markdown
# AGENTS.md

## /root
你是主代理，协调其他代理的工作。

## /code_analyzer
你是代码分析专家，负责：
- 理解代码结构
- 识别关键函数和类
- 分析依赖关系
```

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **并行代码分析** | 同时分析多个文件 |
| **分工测试** | 一个人写测试，一个人 review |
| **复杂重构** | 不同代理处理不同模块 |
| **多语言支持** | 不同代理处理不同语言的代码 |

---

## 限制和安全

### 层级深度限制

防止无限递归创建子代理：

```rust
pub const MAX_THREAD_SPAWN_DEPTH: i32 = 10;

pub fn exceeds_thread_spawn_depth_limit(depth: i32) -> bool {
    depth > MAX_THREAD_SPAWN_DEPTH
}
```

### 资源限制

```toml
[multi_agent]
# 最大并发代理数
max_concurrent_agents = 10

# 每个代理的最大 token 数
max_tokens_per_agent = 50000
```

---

## 总结：多代理协作模式

```
┌─────────────────────────────────────────────────────────┐
│                    用户请求                               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Root Agent                            │
│  • 理解任务                                              │
│  • 分解子任务                                            │
│  • 协调代理                                              │
└─────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Sub-Agent  │  │  Sub-Agent  │  │  Sub-Agent  │
│  (分析)      │  │  (测试)      │  │  (提交)      │
└─────────────┘  └─────────────┘  └─────────────┘
         │                │                │
         └────────────────┴────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    结果汇总                               │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `session/multi_agents.rs` | 多代理配置和提示词 |
| `agent/control.rs` | Agent 控制平面 |
| `agent/registry.rs` | Agent 注册表 |
| `session/turn.rs` | Turn 执行逻辑 |
| `tools/handlers/multi_agents.rs` | 协作工具处理器 |
