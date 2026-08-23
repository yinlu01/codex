# Codex Authentication（认证）

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/login/src/`, `codex-rs/core/src/config/`

## 认证概览

Codex 支持多种认证方式：

| 认证方式 | 说明 |
|---------|------|
| **API Key** | 直接使用 OpenAI API Key |
| **Device Code** | OAuth 设备码流程（浏览器登录） |
| **Workload Identity** | 云环境自动认证（如 AWS IAM） |
| **ChatGPT Plus** | 使用 ChatGPT 订阅登录 |

---

## AuthManager

**文件：** `login/src/auth.rs`

核心认证管理器：

```rust
pub struct AuthManager {
    /// 认证状态
    state: Mutex<AuthState>,

    /// 令牌存储
    token_store: Arc<dyn TokenStore>,

    /// HTTP 客户端
    http: HttpClient,

    /// 认证配置
    config: AuthManagerConfig,
}

pub enum AuthState {
    /// 未登录
    LoggedOut,

    /// 登录中
    LoggingIn,

    /// 已登录
    LoggedIn(AuthCredentials),

    /// 登录失败
    Error(AuthError),
}
```

---

## 认证流程

### 1. API Key 登录

最简单的方式：

```
用户: codex login --api-key sk-xxx

    │
    ▼
AuthManager.login_with_api_key("sk-xxx")
    │
    ▼
验证 API Key 格式
    │
    ▼
存储到 token_store
    │
    ▼
AuthState::LoggedIn
```

```rust
pub fn login_with_api_key(&self, api_key: &str) -> Result<()> {
    // 1. 验证格式
    if !api_key.starts_with("sk-") {
        return Err(AuthError::InvalidKeyFormat);
    }

    // 2. 存储
    self.token_store.save(TokenData {
        api_key: api_key.to_string(),
        ..Default::default()
    })?;

    // 3. 更新状态
    self.set_state(AuthState::LoggedIn(...));

    Ok(())
}
```

### 2. Device Code 登录（OAuth）

适合没有 API Key 的用户：

```
codex login
    │
    ▼
请求设备码
    │
    ▼
显示:
┌─────────────────────────────────────┐
│  1. 打开浏览器访问:                  │
│     https://login.codex.ai/device   │
│                                     │
│  2. 输入码: XXXX-XXXX               │
│                                     │
│  3. 等待授权...                      │
└─────────────────────────────────────┘
    │
    ▼
用户浏览器完成授权
    │
    ▼
Codex 轮询验证
    │
    ▼
获取 token，登录成功
```

### Device Code 流程

**文件：** `login/src/device_code_auth.rs`

```rust
pub async fn run_device_code_login() -> Result<AuthCredentials> {
    // 1. 请求设备码
    let device_code = request_device_code().await?;

    // 2. 显示给用户
    println!("Open: {}", device_code.verification_uri);
    println!("Enter code: {}", device_code.user_code);

    // 3. 轮询等待授权
    loop {
        match check_auth_status(&device_code).await? {
            AuthStatus::Pending => {
                // 继续等待
                sleep(device_code.interval).await;
            }
            AuthStatus::Authorized => {
                // 获取 token
                return get_tokens(device_code).await;
            }
            AuthStatus::Denied => {
                return Err(AuthError::Denied);
            }
        }
    }
}
```

---

## Token 管理

### TokenData

**文件：** `login/src/token_data.rs`

```rust
pub struct TokenData {
    /// Access Token
    pub access_token: String,

    /// Refresh Token
    pub refresh_token: Option<String>,

    /// Token 过期时间
    pub expires_at: DateTime,

    /// Token 类型
    pub token_type: String,

    /// 关联的 API Key（如果有）
    pub api_key: Option<String>,
}
```

### Token 刷新

```rust
pub async fn refresh_if_needed(&self) -> Result<TokenData> {
    // 检查是否快过期（提前 5 分钟刷新）
    if self.is_expiring_soon() {
        return self.refresh().await;
    }
    Ok(self.clone())
}

pub async fn refresh(&self) -> Result<TokenData> {
    let new_tokens = http_client
        .post(REFRESH_TOKEN_URL)
        .json(RefreshRequest {
            refresh_token: self.refresh_token.as_ref().unwrap(),
        })
        .send()
        .await?;

    self.token_store.save(new_tokens)?;
    Ok(new_tokens)
}
```

---

## 认证存储

### TokenStore

```rust
pub trait TokenStore {
    /// 保存 token
    fn save(&self, data: TokenData) -> Result<()>;

    /// 加载 token
    fn load(&self) -> Result<TokenData>;

    /// 删除 token
    fn delete(&self) -> Result<()>;

    /// 检查是否存在
    fn exists(&self) -> bool;
}
```

### 实现方式

| 方式 | 说明 |
|------|------|
| **File Store** | 加密存储到文件 `~/.codex/credentials` |
| **Keyring** | 使用系统密钥链（macOS Keychain, Linux Secret Service） |
| **Memory** | 仅内存存储（不持久化） |

### Keyring 配置

```rust
pub enum AuthKeyringBackendKind {
    /// 自动选择最佳后端
    Auto,

    /// macOS Keychain
    #[cfg(target_os = "macos")]
    Keychain,

    /// Linux Secret Service
    #[cfg(target_os = "linux")]
    SecretService,

    /// Windows Credential Manager
    #[cfg(target_os = "windows")]
    WinCred,

    /// 文件加密存储
    File,
}
```

---

## 环境变量认证

Codex 可以通过环境变量读取认证信息：

```bash
# 设置 API Key
export CODEX_API_KEY=sk-xxx

# 或者直接用 OpenAI API Key
export OPENAI_API_KEY=sk-xxx

# Access Token（内部使用）
export CODEX_ACCESS_TOKEN=xxx
```

**文件：** `login/src/auth.rs`

```rust
pub fn read_codex_access_token_from_env() -> Option<String> {
    std::env::var(CODEX_ACCESS_TOKEN_ENV_VAR).ok()
}

pub fn read_openai_api_key_from_env() -> Option<String> {
    std::env::var(OPENAI_API_KEY_ENV_VAR).ok()
}
```

---

## 多认证源

Codex 支持配置多个认证源：

```toml
# config.toml
[auth]
# 主要认证方式
primary = "openai"

# 备用认证方式
fallback = [
    { type = "api_key", key = "$OPENAI_API_KEY" },
    { type = "device_code" },
]
```

### AuthManager 配置

```rust
pub struct AuthManagerConfig {
    /// 客户端 ID
    client_id: String,

    /// OAuth 配置
    oauth: OAuthConfig,

    /// API 端点
    api_url: String,

    /// Token 存储方式
    store_mode: AuthCredentialsStoreMode,
}
```

---

## Workload Identity（云环境认证）

在云环境（AWS、GCP、Azure）中，可以自动使用云提供商的 IAM 认证：

```rust
pub fn is_workload_identity_selected() -> bool {
    // 检查是否在云环境
    std::env::var("AWS_WEB_IDENTITY_TOKEN_FILE").is_ok()
        || std::env::var("GCP_SERVICE_ACCOUNT").is_ok()
        || std::env::var("AZURE_FEDERATED_TOKEN").is_ok()
}
```

### AWS IAM 认证流程

```
1. 环境变量设置
   AWS_WEB_IDENTITY_TOKEN_FILE=/path/to/token
   AWS_ROLE_ARN=arn:aws:iam::123456:role/CodexRole

2. Codex 自动使用 AWS SDK 获取临时凭证

3. 用临时凭证调用 OpenAI API
```

---

## 登录状态检查

### 检查命令

```bash
codex login --status
```

### 实现

```rust
pub async fn run_login_status() -> Result<LoginStatus> {
    let auth = AuthManager::load()?;

    match auth.get_state() {
        AuthState::LoggedOut => Ok(LoginStatus::LoggedOut),
        AuthState::LoggedIn(creds) => {
            // 检查 token 是否过期
            if creds.is_expired() {
                // 尝试刷新
                match creds.refresh().await {
                    Ok(new_creds) => Ok(LoginStatus::LoggedIn(new_creds)),
                    Err(_) => Ok(LoginStatus::Expired),
                }
            } else {
                Ok(LoginStatus::LoggedIn(creds))
            }
        }
        AuthState::Error(e) => Ok(LoginStatus::Error(e)),
        _ => Ok(LoginStatus::Unknown),
    }
}
```

---

## 登出

```rust
pub async fn logout() -> Result<()> {
    // 1. 撤销 refresh token（如果支持）
    if let Some(refresh_token) = self.token_store.load()?.refresh_token {
        http_client
            .post(REVOKE_TOKEN_URL)
            .json(RevokeRequest { refresh_token })
            .send()
            .await?;
    }

    // 2. 删除本地存储
    self.token_store.delete()?;

    // 3. 更新状态
    self.set_state(AuthState::LoggedOut);

    Ok(())
}
```

---

## 认证错误处理

```rust
pub enum AuthError {
    /// API Key 格式无效
    InvalidKeyFormat,

    /// Token 已过期
    TokenExpired,

    /// 刷新失败
    RefreshFailed,

    /// 认证被拒绝
    Denied,

    /// 网络错误
    NetworkError(String),

    /// 服务器错误
    ServerError(u16),
}

impl AuthError {
    pub fn user_message(&self) -> String {
        match self {
            Self::InvalidKeyFormat => "API Key 格式无效，请检查后重新输入".to_string(),
            Self::TokenExpired => "认证已过期，请重新登录".to_string(),
            Self::RefreshFailed => "认证刷新失败，请重新登录".to_string(),
            Self::Denied => "认证被拒绝，请允许访问".to_string(),
            Self::NetworkError(e) => format!("网络错误: {}", e),
            Self::ServerError(code) => format!("服务器错误: {}", code),
        }
    }
}
```

---

## 总结：认证架构

```
┌─────────────────────────────────────────────────────────┐
│                      用户交互                             │
│  codex login --api-key / codex login (device code)      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    AuthManager                          │
│  • 认证状态管理                                           │
│  • Token 刷新                                            │
│  • 错误处理                                              │
└─────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ API Key     │  │ Device Code │  │ Workload    │
│ 登录        │  │ OAuth       │  │ Identity    │
└─────────────┘  └─────────────┘  └─────────────┘
         │                │                │
         └────────────────┴────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    TokenStore                           │
│  • 文件加密                                              │
│  • Keychain                                             │
│  • Memory                                               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   OpenAI API                            │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `login/src/auth.rs` | 核心认证管理 |
| `login/src/device_code_auth.rs` | OAuth 设备码登录 |
| `login/src/token_data.rs` | Token 数据结构 |
| `login/src/server.rs` | 登录服务器 |
| `core/src/config/auth_keyring.rs` | 密钥链存储 |
| `core/src/config/auth_env_telemetry.rs` | 环境变量 telemetry |
