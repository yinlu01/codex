# Codex Exec Server（远程执行服务器）

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/exec-server/src/`

## 什么是 Exec Server？

**Exec Server** 是 Codex 的远程代码执行服务。负责：
1. 在沙箱中安全执行用户代码
2. 管理执行环境（工作目录、环境变量）
3. 处理文件系统和进程操作
4. 提供执行结果的流式返回

```
┌─────────────────────────────────────────────────────────┐
│                     Codex Core                          │
│                                                         │
│  Session ←── 执行请求 ──→ Exec Server Client            │
└─────────────────────────────────────────────────────────┘
                          │
                          │ gRPC / WebSocket
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Exec Server                          │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Environment │  │   Process   │  │    File     │    │
│  │   Manager   │  │   Manager   │  │   System    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐                      │
│  │   Sandbox   │  │    RPC      │                      │
│  │   Manager   │  │   Server    │                      │
│  └─────────────┘  └─────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

---

## 为什么需要独立的 Exec Server？

### 架构演进

**早期：直接执行**
```
Codex Core ──► exec() 直接运行命令
```
问题：代码和 Core 在同一进程，不安全

**现在：独立服务**
```
Codex Core ──► Exec Server ──► 沙箱执行
```
优点：
- 隔离：执行崩溃不影响 Core
- 安全：独立的沙箱环境
- 可扩展：可以连接远程执行服务器

---

## 核心组件

### 1. Server

**文件：** `exec-server/src/server.rs`

主服务器实现：

```rust
pub struct ExecServer {
    /// RPC 服务器
    rpc: RpcServer,

    /// 环境管理器
    env_manager: EnvironmentManager,

    /// 进程管理器
    process_manager: ProcessManager,

    /// 文件系统
    fs: Arc<dyn ExecutorFileSystem>,
}
```

### 2. Environment Manager

**文件：** `exec-server/src/environment.rs`

管理工作目录和环境：

```rust
pub struct Environment {
    /// 环境 ID
    id: EnvironmentId,

    /// 工作目录
    cwd: PathBuf,

    /// 环境变量
    env: HashMap<String, String>,

    /// 沙箱配置
    sandbox: SandboxConfig,
}

pub struct EnvironmentManager {
    /// 已创建的环境
    environments: RwLock<HashMap<EnvironmentId, Environment>>,

    /// 环境注册表
    registry: EnvironmentRegistry,
}
```

### 3. Process Manager

**文件：** `exec-server/src/process.rs`

管理进程的生命周期：

```rust
pub struct Process {
    /// 进程 ID
    id: ProcessId,

    /// 实际进程
    child: Child,

    /// 输出流
    stdout: Stream,

    /// 错误流
    stderr: Stream,
}

pub struct ProcessManager {
    /// 运行中的进程
    processes: RwLock<HashMap<ProcessId, Process>>,

    /// 进程限制
    max_processes: usize,
}
```

---

## 执行请求流程

```
用户: "运行 python main.py"

    │
    ▼
Codex Core 创建 ExecRequest
    │
    ▼
Exec Server Client 发送请求
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ExecServer.rpc_server_requests                        │
│                                                         │
│  1. 验证请求权限                                         │
│  2. 获取或创建环境                                        │
│  3. 准备执行参数                                          │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  ProcessManager.spawn                                   │
│                                                         │
│  1. 在沙箱中 fork 进程                                    │
│  2. 设置工作目录和环境变量                                 │
│  3. 重定向 stdin/stdout/stderr                          │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  收集执行结果                                             │
│                                                         │
│  • stdout/stderr 流式返回                               │
│  • 退出码                                               │
│  • 执行时间                                              │
└─────────────────────────────────────────────────────────┘
    │
    ▼
返回给 Codex Core
```

---

## 环境管理

### 创建环境

```rust
impl EnvironmentManager {
    pub async fn create_environment(
        &self,
        config: EnvironmentConfig,
    ) -> Result<EnvironmentId> {
        let env = Environment {
            id: EnvironmentId::new(),
            cwd: config.workdir.clone(),
            env: config.env.clone(),
            sandbox: config.sandbox.clone(),
        };

        // 初始化沙箱
        env.init_sandbox().await?;

        // 注册环境
        self.environments.insert(env.id, env).await;

        Ok(env.id)
    }
}
```

### 环境配置

```rust
pub struct EnvironmentConfig {
    /// 工作目录
    pub workdir: PathBuf,

    /// 环境变量
    pub env: HashMap<String, String>,

    /// 沙箱类型
    pub sandbox_type: SandboxType,

    /// 权限配置
    pub permissions: Permissions,
}
```

---

## 文件系统操作

### ExecutorFileSystem Trait

**文件：** `exec-server/src/file_read.rs`

```rust
pub trait ExecutorFileSystem {
    /// 读取文件
    async fn read_file(&self, path: &Path) -> Result<Bytes>;

    /// 写入文件
    async fn write_file(&self, path: &Path, content: Bytes) -> Result<()>;

    /// 创建目录
    async fn create_directory(&self, path: &Path) -> Result<()>;

    /// 列出目录
    async fn read_directory(&self, path: &Path) -> Result<Vec<DirEntry>>;

    /// 获取元数据
    async fn metadata(&self, path: &Path) -> Result<Metadata>;
}
```

### 沙箱文件系统

**文件：** `exec-server/src/sandboxed_file_system.rs`

沙箱内的文件系统有额外限制：

```rust
pub struct SandboxedFileSystem {
    /// 基础文件系统
    base_fs: Arc<dyn ExecutorFileSystem>,

    /// 允许的路径
    allowed_paths: Vec<PathPattern>,

    /// 禁止的路径
    denied_paths: Vec<PathPattern>,
}

impl ExecutorFileSystem for SandboxedFileSystem {
    async fn read_file(&self, path: &Path) -> Result<Bytes> {
        // 1. 检查路径是否在允许列表中
        self.validate_path(path)?;

        // 2. 读取文件
        self.base_fs.read_file(path).await
    }
}
```

---

## RPC 协议

**文件：** `exec-server/src/rpc.rs`

使用 gRPC 通信：

```protobuf
// 简化的 proto 定义
service ExecServer {
    rpc Execute(ExecuteRequest) returns (stream ExecuteResponse);
    rpc CreateEnvironment(EnvConfig) returns (EnvironmentId);
    rpc ReadFile(ReadRequest) returns (ReadResponse);
    rpc WriteFile(WriteRequest) returns (WriteResponse);
    rpc ListProcesses(Empty) returns (ProcessList);
    rpc KillProcess(ProcessId) returns (Empty);
}
```

### 请求/响应示例

```rust
// Execute 请求
pub struct ExecuteRequest {
    pub environment_id: EnvironmentId,
    pub command: Vec<String>,
    pub working_directory: Option<String>,
    pub environment_vars: HashMap<String, String>,
    pub stdin: Option<Bytes>,
    pub deadline: Option<Duration>,
}

// Execute 响应（流式）
pub struct ExecuteResponse {
    pub stream_type: StreamType,
    pub data: Bytes,
    pub exit_code: Option<i32>,
}

pub enum StreamType {
    Stdout,
    Stderr,
    Exited,
}
```

---

## 连接管理

### Client

**文件：** `exec-server/src/client.rs`

```rust
pub struct ExecServerClient {
    /// HTTP 客户端
    http: HttpClient,

    /// WebSocket 连接
    ws: WebSocketClient,

    /// 重连策略
    recovery: ClientRecovery,
}

impl ExecServerClient {
    /// 连接到 Exec Server
    pub async fn connect(addr: &str) -> Result<Self>;

    /// 执行命令
    pub async fn execute(&self, req: ExecuteRequest) -> Result<ExecuteResponseStream>;

    /// 重连（自动恢复）
    pub async fn reconnect(&self) -> Result<()>;
}
```

### 自动恢复

**文件：** `exec-server/src/client_recovery.rs`

Exec Server Client 支持自动重连：

```rust
pub struct ClientRecovery {
    /// 最大重试次数
    max_retries: u32,

    /// 重试间隔
    retry_interval: Duration,

    /// 指数退避
    exponential_backoff: bool,
}
```

---

## 沙箱集成

Exec Server 使用底层沙箱能力：

```
ExecServer
    │
    ▼
ProcessManager.spawn()
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              平台特定沙箱实现                              │
│                                                         │
│  Linux:                                                 │
│    ├── bubblewrap 容器                                  │
│    ├── seccomp 过滤器                                   │
│    └── landlock 规则                                    │
│                                                         │
│  macOS:                                                 │
│    └── Seatbelt 沙箱                                    │
│                                                         │
│  Windows:                                               │
│    └── Windows Sandbox / Hyper-V                        │
└─────────────────────────────────────────────────────────┘
```

---

## 安全考虑

### 1. 路径验证

```rust
pub fn validate_path(path: &Path) -> Result<()> {
    // 标准化路径（防止 ../ 逃逸）
    let normalized = path.normalize();

    // 检查是否在允许目录内
    if !normalized.starts_with(&ALLOWED_ROOT) {
        return Err(Error::PathOutsideSandbox);
    }

    Ok(())
}
```

### 2. 命令验证

```rust
pub fn validate_command(cmd: &[String]) -> Result<()> {
    // 检查命令是否在白名单中
    if !ALLOWED_COMMANDS.contains(&cmd[0]) {
        return Err(Error::CommandNotAllowed(cmd[0].clone()));
    }

    Ok(())
}
```

### 3. 资源限制

```rust
pub struct ResourceLimits {
    /// 最大内存 (MB)
    pub max_memory_mb: u64,

    /// 最大 CPU 时间 (秒)
    pub max_cpu_seconds: u64,

    /// 最大输出大小 (bytes)
    pub max_output_bytes: u64,

    /// 最大进程数
    pub max_processes: usize,
}
```

---

## 总结：Exec Server 架构

```
┌─────────────────────────────────────────────────────────┐
│                      Codex Core                         │
│  Session ←─── Execute ──────────────────────────────►   │
└─────────────────────────────────────────────────────────┘
                           │
                           │ gRPC / WebSocket
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Exec Server                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              RPC Server                          │   │
│  │  • Execute    • CreateEnvironment                │   │
│  │  • ReadFile   • WriteFile                        │   │
│  │  • ListDir    • KillProcess                      │   │
│  └─────────────────────────────────────────────────┘   │
│                           │                             │
│         ┌─────────────────┼─────────────────┐          │
│         ▼                 ▼                 ▼          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Process    │  │  File       │  │  Environment│    │
│  │  Manager    │  │  System     │  │  Manager    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│         │                 │                 │          │
│         └─────────────────┼─────────────────┘          │
│                           ▼                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Sandbox Layer                       │   │
│  │  Linux: bubblewrap + seccomp + landlock         │   │
│  │  macOS: Seatbelt                               │   │
│  │  Windows: Windows Sandbox                      │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `exec-server/src/server.rs` | 主服务器 |
| `exec-server/src/client.rs` | 客户端 |
| `exec-server/src/environment.rs` | 环境管理 |
| `exec-server/src/process.rs` | 进程管理 |
| `exec-server/src/rpc.rs` | RPC 协议 |
| `exec-server/src/sandboxed_file_system.rs` | 沙箱文件系统 |
| `exec-server/src/client_recovery.rs` | 客户端恢复 |
