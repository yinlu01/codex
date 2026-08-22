# Codex Plugins（插件系统）

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/core-plugins/`, `codex-rs/core/src/plugins/`

## 什么是插件？

**插件（Plugin）** 是扩展 Codex 能力的模块。你可以把它理解为"App Store 应用"：

```
Codex 核心功能（内置）
    │
    ├── 文件读写
    ├── 代码执行
    ├── Git 操作
    └── ...

插件扩展（按需安装）
    │
    ├── 代码格式化插件
    ├── 静态分析插件
    ├── 数据库工具插件
    ├── AI 模型插件
    └── ...
```

---

## 插件 vs MCP 工具

| 特性 | Plugin | MCP |
|------|--------|-----|
| **安装方式** | 下载安装包 | 配置服务器地址 |
| **执行环境** | 和 Codex 一起运行 | 独立进程 |
| **能力** | 更深入的系统集成 | 标准的工具调用 |
| **性能** | 更快（无进程通信） | 有网络开销 |
| **适用场景** | 核心功能扩展 | 外部服务集成 |

---

## 插件系统架构

### 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **PluginsManager** | `plugins/manager.rs` | 插件生命周期管理 |
| **PluginLoader** | `plugins/loader.rs` | 插件加载 |
| **PluginStore** | `plugins/store.rs` | 插件存储 |
| **Marketplace** | `plugins/marketplace.rs` | 插件市场 |

### PluginsManager

```rust
pub struct PluginsManager {
    /// 已安装的插件
    installed: RwLock<HashMap<PluginId, LoadedPlugin>>,

    /// 插件配置
    config: PluginConfig,

    /// 插件市场
    marketplace: Arc<dyn Marketplace>,
}
```

---

## 插件的组成部分

### 1. Manifest（清单文件）

```json
// manifest.json
{
  "id": "my-formatter",
  "name": "代码格式化器",
  "version": "1.0.0",
  "description": "自动格式化代码",
  "author": "开发者",

  "permissions": [
    "file:read",
    "file:write",
    "exec:allow:prettier"
  ],

  "entry": "dist/index.js",

  "tools": [
    {
      "name": "format_code",
      "description": "格式化代码文件",
      "parameters": {
        "path": "string",
        "style": "string"
      }
    }
  ],

  "hooks": {
    "on_tool_call": "handle_tool_call"
  }
}
```

### 2. 入口文件

```javascript
// index.js
module.exports = {
  // 工具定义
  tools: [{
    name: 'format_code',
    execute: async (args) => {
      // 执行格式化
      return await prettier.format(args.path, { style: args.style });
    }
  }],

  // 钩子
  hooks: {
    onToolCall: (toolName, args) => {
      console.log(`Tool called: ${toolName}`);
    }
  }
};
```

---

## 插件市场

### Marketplace

```rust
pub trait Marketplace {
    /// 列出可用插件
    async fn list(&self) -> Result<Vec<PluginListing>>;

    /// 搜索插件
    async fn search(&self, query: &str) -> Result<Vec<PluginListing>>;

    /// 获取插件详情
    async fn get(&self, id: &str) -> Result<PluginDetail>;

    /// 下载插件
    async fn download(&self, id: &str, version: &str) -> Result<PluginBundle>;
}
```

### 官方市场

```
Marketplace 名称                      说明
openai-curated                      OpenAI 官方 curated 插件
openai-api-curated                  API 版本 curated 插件
openai-bundled                      捆绑插件
openai-primary-runtime              主要运行时插件
```

---

## 插件安装流程

```
用户: "安装代码格式化插件"

    │
    ▼
PluginsManager.install("formatter-plugin")
    │
    ▼
连接 Marketplace，搜索插件
    │
    ▼
下载插件包 (bundle)
    │
    ▼
验证插件签名和权限
    │
    ▼
解压并安装到 ~/.codex/plugins/
    │
    ▼
加载插件 (调用入口文件)
    │
    ▼
注册工具到 ToolRouter
    │
    ▼
插件可用
```

---

## 插件权限

### 权限模型

```rust
pub struct PluginPermissions {
    /// 允许读取的文件路径
    pub file_read: Vec<PathPattern>,

    /// 允许写入的文件路径
    pub file_write: Vec<PathPattern>,

    /// 允许执行的命令
    pub exec_allow: Vec<String>,

    /// 禁止的操作
    pub deny: Vec<Permission>,
}
```

### 权限示例

```json
// manifest.json 中的权限声明
{
  "permissions": {
    "file_read": ["**/*.py", "**/*.js"],
    "file_write": ["**/*.py", "**/*.js"],
    "exec_allow": ["prettier", "black", "ruff"]
  }
}
```

### 用户确认

```toml
# config.toml
[plugin_permissions]
# 插件安装时需要用户确认权限
prompt_on_install = true

# 信任的插件（跳过确认）
trusted = ["openai-curated/*"]
```

---

## 插件生命周期

```
┌─────────────────────────────────────────────────────────┐
│                    插件生命周期                           │
└─────────────────────────────────────────────────────────┘

install ──► load ──► enable ──► running ──► disable ──► uninstall
   │         │         │          │          │           │
   ▼         ▼         ▼          ▼          ▼           ▼
  下载    加载代码   注册工具   处理请求   暂停工具    清理资源
  验证    初始化     钩子注册   执行中     保留状态    删除文件
```

### 关键方法

```rust
impl Plugin {
    /// 安装插件
    pub async fn install(&self, marketplace: &Marketplace) -> Result<()>;

    /// 加载插件
    pub async fn load(&self) -> Result<LoadedPlugin>;

    /// 启用插件
    pub async fn enable(&self) -> Result<()>;

    /// 禁用插件
    pub async fn disable(&self) -> Result<()>;

    /// 卸载插件
    pub async fn uninstall(&self) -> Result<()>;
}
```

---

## 插件钩子（Hooks）

插件可以注册钩子，在特定事件发生时被调用：

### 可用钩子

| 钩子 | 触发时机 |
|------|---------|
| `onStartup` | Codex 启动时 |
| `onToolCall` | 工具被调用前/后 |
| `onMessage` | 消息收到时 |
| `onTurn` | 每个 Turn 开始/结束时 |
| `onError` | 发生错误时 |

### 钩子示例

```javascript
// 格式化插件的钩子
module.exports = {
  hooks: {
    onToolCall: async (toolCall, context) => {
      // 如果调用的是 Write 工具，自动格式化
      if (toolCall.name === 'Write') {
        const formatted = await prettier.format(toolCall.args.content);
        toolCall.args.content = formatted;
      }
    }
  }
};
```

---

## 内置插件

Codex 带了一些内置插件：

### 1. Artifact Operations

处理代码片段的存储和展示：

```rust
// artifact_operation.rs
pub enum ArtifactOperation {
    Create { content: String, language: String },
    Update { id: String, content: String },
    Read { id: String },
    Delete { id: String },
}
```

### 2. Git Policy

Git 操作策略控制：

```rust
// git_policy.rs
pub enum PluginGitMode {
    /// 不允许 Git 操作
    Disabled,
    /// 允许读操作
    ReadOnly,
    /// 允许所有操作
    Full,
}
```

### 3. Tool Suggest

工具建议元数据：

```rust
// tool_suggest_metadata.rs
pub struct ToolSuggestDiscoverablePlugin {
    pub plugin_id: String,
    pub tool_names: Vec<String>,
    pub discovery_hints: Vec<String>,
}
```

---

## 插件配置

### config.toml

```toml
[plugins]
# 启用/禁用插件
enabled = true

# 自动安装推荐的插件
auto_install_recommended = true

# 插件目录
install_dir = "~/.codex/plugins"

# 市场配置
[plugins.markets]
openai = "https://marketplace.codex.ai"
custom = "https://my-plugins.example.com"
```

### AGENTS.md 中的插件声明

```markdown
# AGENTS.md

## 启用的插件
- prettier-format: 代码格式化
- git-helper: Git 操作辅助
- code-search: 代码搜索
```

---

## 插件开发

### 创建新插件

1. **创建目录结构**

```
my-plugin/
├── manifest.json
├── src/
│   └── index.js
└── dist/
    └── index.js (编译后)
```

2. **编写 manifest**

```json
{
  "id": "my-plugin",
  "name": "我的插件",
  "version": "0.1.0",
  "entry": "dist/index.js"
}
```

3. **编写代码**

```javascript
// src/index.js
module.exports = {
  tools: [{
    name: 'hello',
    description: '说 hello',
    execute: async () => 'Hello, World!'
  }]
};
```

4. **打包发布**

```bash
npm run build
codex plugin publish
```

---

## 总结：插件系统架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户/配置                              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 PluginsManager                           │
│  • 生命周期管理                                           │
│  • 配置加载                                              │
│  • 权限验证                                              │
└─────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Plugin A   │  │  Plugin B   │  │  Plugin C   │
│             │  │             │  │             │
│ • Tools     │  │ • Hooks     │  │ • Tools     │
│ • Hooks     │  │ • Render    │  │             │
└─────────────┘  └─────────────┘  └─────────────┘
         │                │                │
         └────────────────┴────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Codex Core                            │
│  • ToolRouter ← 注册工具                                 │
│  • HookRuntime ← 执行钩子                                │
│  • Session ← 管理会话                                    │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `core-plugins/src/manager.rs` | 插件管理器 |
| `core-plugins/src/loader.rs` | 插件加载器 |
| `core-plugins/src/store.rs` | 插件存储 |
| `core-plugins/src/marketplace.rs` | 插件市场 |
| `core-plugins/src/manifest.rs` | 插件清单 |
| `core-plugins/src/hooks.rs` | 钩子系统 |
