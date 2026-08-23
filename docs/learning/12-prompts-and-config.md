# Codex Prompts（提示词）和 Config（配置）

> 深入学习日期：2026-08-23
> 对应代码：`codex-rs/prompts/src/`, `codex-rs/core/src/config/`

---

## Part 1: Prompts（提示词）

**文件：** `codex-rs/prompts/src/`

Codex 的提示词管理非常系统化，所有提示词都集中管理，方便维护和更新。

### Prompt 模块

| 模块 | 文件 | 用途 |
|------|------|------|
| **compact** | `compact.rs` | 历史压缩提示词 |
| **goals** | `goals.rs` | 目标/限制提示词 |
| **permissions** | `permissions_instructions.rs` | 权限审批提示词 |
| **realtime** | `realtime.rs` | 实时对话提示词 |
| **review_exit** | `review_exit.rs` | 审核退出提示词 |
| **review_request** | `review_request.rs` | 审核请求提示词 |

### 压缩提示词

**文件：** `prompts/src/compact.rs`

当对话历史太长需要压缩时使用的提示词：

```rust
/// 摘要生成提示词
pub const SUMMARIZATION_PROMPT: &str = r#"
请简洁地总结以下对话的核心内容：

原始对话：
{history}

要求：
1. 保留关键决策和结论
2. 移除重复的讨论
3. 保留重要的代码变更
4. 总结不超过 500 字

摘要："#;

/// 摘要前缀
pub const SUMMARY_PREFIX: &str = "[对话摘要] ";
```

### 目标提示词

**文件：** `prompts/src/goals.rs`

管理预算限制和目标相关的提示词：

```rust
/// 预算限制提示词
pub fn budget_limit_prompt(remaining_tokens: u64) -> String {
    format!(
        "注意：你只剩下约 {} tokens 的上下文空间。
请简洁回答，聚焦核心内容。",
        remaining_tokens
    )
}

/// 继续提示词（当任务未完成时）
pub fn continuation_prompt() -> String {
    "请继续完成上一个任务。".to_string()
}

/// 目标更新提示词
pub fn objective_updated_prompt(new_objective: &str) -> String {
    format!(
        "任务目标已更新：{}\n\n请根据新目标调整你的工作。",
        new_objective
    )
}
```

### 权限审批提示词

**文件：** `prompts/src/permissions_instructions.rs`

```rust
/// 权限请求上下文
pub struct ApprovalPromptContext {
    /// 请求的操作
    pub operation: String,

    /// 涉及的文件/资源
    pub resource: String,

    /// 操作原因
    pub justification: Option<String>,

    /// 风险级别
    pub risk_level: RiskLevel,
}

/// 生成权限请求提示词
pub fn build_approval_prompt(ctx: &ApprovalPromptContext) -> String {
    match ctx.risk_level {
        RiskLevel::Low => format!(
            "Codex 请求执行：{}\n\n资源：{}\n\n这是低风险操作。",
            ctx.operation, ctx.resource
        ),
        RiskLevel::Medium => format!(
            "Codex 请求执行：{}\n\n资源：{}\n\n原因：{}\n\n请确认是否允许。",
            ctx.operation,
            ctx.resource,
            ctx.justification.as_deref().unwrap_or("用户未说明")
        ),
        RiskLevel::High => format!(
            "⚠️ 高风险操作请求\n\n操作：{}\n资源：{}\n原因：{}\n\n请仔细审核！",
            ctx.operation,
            ctx.resource,
            ctx.justification.as_deref().unwrap_or("用户未说明")
        ),
    }
}
```

### 审核请求提示词

**文件：** `prompts/src/review_request.rs`

```rust
pub struct ReviewRequest {
    /// 请求类型
    pub review_type: ReviewType,

    /// 代码内容
    pub code: String,

    /// 相关文件
    pub file: Option<String>,
}

pub enum ReviewType {
    CodeReview,      // 代码审查
    SecurityCheck,   // 安全检查
    PerformanceCheck, // 性能检查
    BestPractices,   // 最佳实践检查
}

/// 生成审核提示词
pub fn review_prompt(request: &ReviewRequest) -> String {
    match request.review_type {
        ReviewType::CodeReview => format!(
            "请审查以下代码：\n\n```{}\n{}\n```\n\n文件：{}\n\n检查：\n1. 逻辑正确性\n2. 边界条件\n3. 错误处理",
            request.language.unwrap_or(""),
            request.code,
            request.file.as_deref().unwrap_or("未指定")
        ),
        ReviewType::SecurityCheck => format!(
            "安全审查：\n\n{}\n\n检查潜在安全漏洞：注入、泄露、权限等",
            request.code
        ),
        // ...
    }
}
```

### 实时对话提示词

**文件：** `prompts/src/realtime.rs`

```rust
/// 后端提示词
pub const BACKEND_PROMPT: &str = r#"
你是一个专业的编程助手。
你能够帮助用户：
- 编写和调试代码
- 分析项目结构
- 执行 Git 操作
- 解释技术概念

请始终：
1. 提供准确的信息
2. 解释你的推理过程
3. 在不确定时承认不知道
"#;

/// 开始指令
pub const START_INSTRUCTIONS: &str = r#"
对话开始。

当前环境：
{environment}

当前工作目录：
{cwd}

可用工具：
{available_tools}
"#;

/// 结束指令
pub const END_INSTRUCTIONS: &str = r#"
任务完成。

如果用户需要更多帮助，请继续。
"#;
```

---

## Part 2: Config（配置系统）

**文件：** `codex-rs/core/src/config/`

### 配置层次

Codex 使用**分层配置**系统：

```
┌─────────────────────────────────────────────────────────┐
│                   默认配置 (代码)                         │
│  提供最保守的默认值                                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 系统配置 (~/.codex/config.toml)           │
│  系统级配置，影响所有用户                                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 用户配置 (~/.codex/config.toml)           │
│  用户个性化设置                                           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 项目配置 (.codex.toml)                    │
│  项目级配置，仅在特定项目中生效                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 命令行参数 (最高优先级)                     │
│  命令行覆盖任何配置文件                                    │
└─────────────────────────────────────────────────────────┘
```

### 配置结构

**文件：** `config/schema.md`

```toml
# ~/.codex/config.toml 示例

# 模型配置
[model]
provider = "openai"
model = "gpt-4o"
temperature = 0.7
max_tokens = 8192

# 特性开关
[features]
code_mode = true
multi_agent = true
token_budget = true

# MCP 服务器
[mcp_servers]
github = { command = "npx", args = ["-y", "@modelcontextprotocol/server-github"] }

# 插件
[plugins]
enabled = true
auto_install = true

# 权限配置
[permissions]
allow_read = ["~/code/**", "/project/**"]
allow_write = ["~/code/**"]
allow_exec = ["git", "npm", "cargo", "python"]

# Token 预算
[token_budget]
reminder_threshold_tokens = 10000
guidance_message = "上下文快满了，要我帮你清理历史吗？"

# 多代理
[multi_agent]
enabled = true
max_sub_agents = 10
max_depth = 5

# 日志
[logging]
level = "info"
file = "~/.codex/logs/codex.log"
```

### ConfigToml 结构

**文件：** `codex_config/config_toml.rs`

```rust
pub struct ConfigToml {
    /// 模型配置
    pub model: Option<ModelConfig>,

    /// 特性配置
    pub features: Option<FeaturesToml>,

    /// MCP 服务器
    pub mcp_servers: HashMap<String, McpServerConfig>,

    /// 插件配置
    pub plugins: Option<PluginsConfig>,

    /// 权限配置
    pub permissions: Option<PermissionsToml>,

    /// Token 预算
    pub token_budget: Option<TokenBudgetConfigToml>,

    /// 多代理配置
    pub multi_agent: Option<MultiAgentV2ConfigToml>,

    /// 日志配置
    pub logging: Option<LoggingConfig>,
}
```

### 配置加载

**文件：** `config/loader.rs`

```rust
pub async fn load_config_layers_state(
    options: ConfigLoadOptions,
) -> Result<ConfigLayerStack> {
    let mut layers = Vec::new();

    // 1. 加载默认配置
    layers.push(ConfigLayer::default());

    // 2. 加载系统配置
    if let Some(system_config) = load_system_config()? {
        layers.push(system_config);
    }

    // 3. 加载用户配置
    if let Some(user_config) = load_user_config()? {
        layers.push(user_config);
    }

    // 4. 加载项目配置
    if let Some(project_config) = load_project_config(options.cwd)? {
        layers.push(project_config);
    }

    // 5. 合并层次
    ConfigLayerStack::merge(layers)
}
```

### 配置验证

**文件：** `config/schema.rs`

```rust
pub fn validate_config(config: &ConfigToml) -> Result<(), ConfigError> {
    // 验证模型配置
    if let Some(ref model) = config.model {
        validate_model_config(model)?;
    }

    // 验证权限配置
    if let Some(ref permissions) = config.permissions {
        validate_permissions(permissions)?;
    }

    // 验证 MCP 服务器
    if let Some(ref mcp) = config.mcp_servers {
        for (name, server) in mcp {
            validate_mcp_server(name, server)?;
        }
    }

    Ok(())
}
```

### 配置编辑

**文件：** `config/edit.rs`

支持通过代码修改配置：

```rust
pub struct ConfigEditsBuilder {
    edits: Vec<ConfigEdit>,
}

impl ConfigEditsBuilder {
    pub fn new() -> Self;

    pub fn set_model(&mut self, model: &str) -> &mut Self;
    pub fn set_feature(&mut self, feature: &str, enabled: bool) -> &mut Self;
    pub fn add_mcp_server(&mut self, name: &str, config: &McpServerConfig) -> &mut Self;
    pub fn remove_mcp_server(&mut self, name: &str) -> &mut Self;

    pub fn apply(&self, config: &mut ConfigToml) -> Result<()>;
}
```

### 权限配置

**文件：** `config/permissions.rs`

```rust
pub struct PermissionsToml {
    /// 允许读取的路径
    pub allow_read: Vec<PathPattern>,

    /// 允许写入的路径
    pub allow_write: Vec<PathPattern>,

    /// 允许执行的命令
    pub allow_exec: Vec<String>,

    /// 禁止的路径
    pub deny: Vec<PathPattern>,
}

pub struct PathPattern {
    /// 原始模式
    pattern: String,

    /// 是否是目录
    is_directory: bool,

    /// 是否递归
    recursive: bool,
}
```

---

## Part 3: 配置和提示词的协作

```
用户输入 + 配置
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  ConfigLayerStack — 合并多层配置                          │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  TurnContext — 创建回合上下文                              │
│  • 加载模型配置                                           │
│  • 加载功能开关                                           │
│  • 加载权限配置                                           │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  PromptBuilder — 构建提示词                               │
│  • 基础提示词 (prompts/src/)                              │
│  • 权限提示词                                             │
│  • 审核提示词                                             │
│  • 实时指令                                               │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  LLM — 发送完整提示词                                      │
└─────────────────────────────────────────────────────────┘
```

---

## 总结

### Prompts 核心文件

| 文件 | 用途 |
|------|------|
| `prompts/src/compact.rs` | 历史压缩提示词 |
| `prompts/src/goals.rs` | 目标和限制提示词 |
| `prompts/src/permissions_instructions.rs` | 权限审批提示词 |
| `prompts/src/realtime.rs` | 实时对话提示词 |
| `prompts/src/review_request.rs` | 审核请求提示词 |

### Config 核心文件

| 文件 | 用途 |
|------|------|
| `config/schema.md` | 配置 JSON Schema 说明 |
| `config/schema.rs` | 配置验证逻辑 |
| `config/edit.rs` | 配置编辑 |
| `config/permissions.rs` | 权限配置 |
| `config/mod.rs` | 配置主模块 |

---

## 自定义开发建议

### 1. 添加新的提示词

在 `prompts/src/` 对应模块添加：

```rust
// prompts/src/my_feature.rs
pub const MY_FEATURE_PROMPT: &str = r#"..."#;
```

### 2. 添加新的配置项

1. 在 `ConfigToml` 添加字段：

```rust
// codex_config/config_toml.rs
pub struct ConfigToml {
    // ...existing fields...
    pub my_feature: Option<MyFeatureConfig>,
}
```

2. 添加验证：

```rust
// config/schema.rs
fn validate_my_feature(config: &MyFeatureConfig) -> Result<()> {
    // 验证逻辑
}
```

3. 更新 JSON Schema：

```bash
just write-config-schema
```
