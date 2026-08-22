# Codex Context Management（上下文管理）

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/core/src/context_manager/`, `codex-rs/core/src/session/token_budget.rs`

## 什么是上下文管理？

LLM（如 GPT-4）有 **token 限制**。比如：
- GPT-4-128K：最多 128,000 tokens
- Claude 100K：最多 100,000 tokens

当你和 Codex 对话很多轮后，历史记录会越来越长，最终会超出限制。

**上下文管理** 就是解决这个问题的机制：
1. **追踪** 用了多少 tokens
2. **警告** 快用完时提醒
3. **压缩** 历史太长了就精简

```
对话轮数增加：
Turn 1: ████░░░░░░░░░░░░░░░░░░░  15%
Turn 5: ████████████████░░░░░░░  45%
Turn 10: ██████████████████████  95% ⚠️ 接近上限
Turn 11: → 触发压缩，精简历史，重新回到 50%
```

---

## 核心概念

### 1. Token（令牌）

**Token** 是 LLM 处理文本的基本单位。粗略估算：
- 1 个英文单词 ≈ 1.3 tokens
- 1 个中文字符 ≈ 2 tokens
- 1 行代码 ≈ 4-10 tokens

### 2. Context Window（上下文窗口）

LLM 一次能处理的最大 token 数：

| 模型 | 上下文窗口 |
|------|-----------|
| GPT-4-128K | 128,000 tokens |
| Claude 100K | 100,000 tokens |
| GPT-4 | 8,192 tokens |

### 3. Token Budget（Token 预算）

Codex 会追踪并管理 token 使用：

```rust
pub struct TokenUsageInfo {
    pub prompt_tokens: u64,      // 提示消耗的 tokens
    pub completion_tokens: u64,  // 生成消耗的 tokens
    pub total_tokens: u64,       // 总计
    pub context_window: u64,     // 上下文窗口大小
    pub remaining: i64,          // 剩余可用
}
```

---

## ContextManager（上下文管理器）

**文件：** `context_manager/history.rs`

```rust
pub(crate) struct ContextManager {
    /// 对话历史，最老的在前面
    items: Arc<Vec<ResponseItemEnvelope>>,

    /// 历史版本号，压缩/回滚时会增加
    history_version: u64,

    /// Token 使用信息
    token_info: Option<TokenUsageInfo>,

    /// 用于差量更新的基准上下文
    reference_context_item: Option<TurnContextItem>,

    /// 世界状态基线（最近的系统状态）
    world_state_baseline: Option<WorldStateSnapshot>,
}
```

### 主要功能

```rust
impl ContextManager {
    /// 创建新的上下文管理器
    pub(crate) fn new() -> Self

    /// 获取对话历史快照
    pub(crate) fn conversation_history_snapshot(&self) -> Arc<dyn ConversationHistorySnapshot>

    /// 获取当前 token 使用信息
    pub(crate) fn token_info(&self) -> Option<TokenUsageInfo>

    /// 添加新的对话项
    pub(crate) fn push_item(&mut self, item: ResponseItemEnvelope)

    /// 截断历史（保留最近 N 条）
    pub(crate) fn truncate_to(&mut self, max_items: usize)

    /// 压缩历史（合并/精简）
    pub(crate) fn compact(&mut self) -> Result<()>
}
```

---

## Token 预算管理

**文件：** `session/token_budget.rs`

Codex 有专门的 token 预算管理：

```rust
pub struct TokenBudgetConfig {
    /// 是否启用
    pub enabled: bool,

    /// 剩余多少 tokens 时开始提醒
    pub reminder_threshold_tokens: Option<u64>,

    /// 提醒消息模板
    pub reminder_message_template: Option<String>,

    /// 指导消息
    pub guidance_message: Option<String>,

    /// 自动压缩的备用提示词
    pub auto_compact_fallback_prompt: Option<String>,

    /// 自动压缩的缓冲区大小
    pub auto_compact_fallback_buffer_tokens: Option<u64>,
}
```

### 提醒机制

```rust
// 当 token 剩余量低于阈值时，Codex 会提醒用户
if remaining_tokens < config.reminder_threshold_tokens {
    // Codex 会说："上下文快满了，要我帮你清理一下历史吗？"
}
```

---

## 历史截断（Truncation）

当历史太长时，Codex 会截断最旧的内容：

```
原始历史（按时间顺序）:
[Turn 1] → [Turn 2] → [Turn 3] → [Turn 4] → [Turn 5]

截断后（保留最近 3 轮）:
[Turn 3] → [Turn 4] → [Turn 5]
```

```rust
pub(crate) fn truncate_to(&mut self, max_items: usize) {
    if self.items.len() > max_items {
        // 保留最近的 max_items 条
        let new_items: Vec<_> = self.items
            .iter()
            .rev()  // 反转：从最新开始数
            .take(max_items)
            .cloned()
            .collect();

        self.items = Arc::new(new_items.into_iter().rev().collect());
        self.history_version += 1;
    }
}
```

---

## 历史压缩（Compaction）

比截断更智能的方式是**压缩**：

```
压缩前:
[Turn 1: "帮我写个排序算法"] → 1000 tokens
[Turn 2: "加个注释"] → 200 tokens
[Turn 3: "再加点测试"] → 300 tokens

压缩后:
[Turn 1-3 摘要: "用户要求实现排序算法，包含注释和测试"] → 50 tokens
```

### 压缩策略

| 策略 | 说明 |
|------|------|
| **摘要压缩** | 把多轮对话合并成一个摘要 |
| **选择性保留** | 保留重要的（代码变更），删除次要的（闲聊） |
| **差量更新** | 只传递变化的部分，而不是全量历史 |

---

## TurnContext（回合上下文）

**文件：** `session/turn_context.rs`

每个 Turn（回合）都有自己的上下文：

```rust
pub struct TurnContext {
    /// 模型信息（上下文窗口、token 限制等）
    pub model_info: ModelInfo,

    /// 当前回合的 token 预算
    pub token_budget: TokenBudget,

    /// 对话历史
    pub history: Vec<ResponseItem>,

    /// 用户提供的上下文片段
    pub user_fragments: Vec<ContextualUserFragment>,

    /// 世界状态（当前项目结构等）
    pub world_state: WorldState,
}
```

### Turn 的生命周期

```
TurnContext 创建
    │
    ▼
加载历史到上下文
    │
    ▼
计算当前 token 使用量
    │
    ├──► 超限 → 触发压缩
    │
    ▼
执行用户请求
    │
    ▼
保存结果到历史
    │
    ▼
更新 token 统计
```

---

## 用户上下文片段

用户可以提供额外的上下文：

```rust
pub struct ContextualUserFragment {
    /// 片段内容
    pub content: String,

    /// 片段类型
    pub fragment_type: FragmentType,
}

pub enum FragmentType {
    /// 项目说明（如 README 内容）
    ProjectContext,

    /// 代码片段
    CodeSnippet,

    /// 文件内容
    FileContent,

    /// 其他参考
    Reference,
}
```

用户可以在对话中这样使用：

```
用户: @project_readme.md 请根据这个项目的结构来回答
用户: @src/main.py 这是我的主文件，帮我分析
```

---

## 世界状态（World State）

Codex 维护一个"世界状态"，记录当前项目的结构：

```rust
pub struct WorldState {
    /// 项目的目录结构
    pub directory_tree: DirectoryTree,

    /// 当前打开的文件
    pub open_files: Vec<FilePath>,

    /// 最近的修改
    pub recent_changes: Vec<Change>,

    /// 光标位置（如果支持）
    pub cursor_position: Option<CursorPosition>,
}
```

世界状态会作为上下文的一部分传给 LLM，让它"知道"项目的当前状态。

---

## 差量上下文更新

为了节省 tokens，Codex 使用**差量更新**：

```
传统方式（全量传递）:
每次都传递完整的历史 → 浪费大量重复 tokens

差量方式（只传变化）:
Turn N:
  历史基线: [A, B, C]
  变化: +[D]
  传递: [A, B, C, D]

Turn N+1:
  历史基线: [A, B, C, D]
  变化: +[E, F]
  传递: [E, F]  ← 只传新的！
```

```rust
pub(crate) fn reference_context_item(&self) -> Option<TurnContextItem> {
    self.reference_context_item.clone()
}

pub(crate) fn set_reference_context_item(&mut self, item: Option<TurnContextItem>) {
    self.reference_context_item = item;
}
```

---

## 总结：上下文管理流程

```
用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  TurnContext 创建                                        │
│  - 加载历史                                              │
│  - 计算 token 使用量                                      │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Token 预算检查                                          │
│  - 剩余是否充足？                                        │
│  - 是否低于提醒阈值？                                     │
└─────────────────────────────────────────────────────────┘
    │
    ├─► 低于阈值 → 发送提醒给用户
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  执行请求                                                │
│  - 生成响应                                              │
│  - 记录新的对话项                                         │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  历史更新                                                │
│  - 添加新项                                              │
│  - 更新 token 统计                                        │
│  - 必要时触发压缩                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `context_manager/history.rs` | 上下文管理器核心 |
| `context_manager/normalize.rs` | 历史规范化 |
| `session/turn_context.rs` | 单个回合的上下文 |
| `session/token_budget.rs` | Token 预算管理 |
| `compact.rs` | 历史压缩逻辑 |
| `session_prefix.rs` | 多轮对话的前缀管理 |
