# Codex Sandbox（沙箱）安全执行系统

> 深入学习日期：2026-08-22
> 对应代码：`codex-rs/linux-sandbox/`, `codex-rs/sandboxing/`

## 为什么需要沙箱？

**沙箱（Sandbox）** 是 Codex 安全执行代码的核心机制。简单说：

```
┌─────────────────────────────────────────────────────────────┐
│                    你的电脑 (Host)                           │
│                                                             │
│   用户说: "帮我删除所有文件"                                   │
│                                                             │
│   如果没有沙箱 → Codex 真会删除你电脑上的所有文件！               │
│   如果有沙箱   → Codex 只在沙箱里"假装"删除了，安全！             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**沙箱的目标：**
1. **隔离** - 代码运行在隔离环境，不影响真实系统
2. **权限控制** - 只允许访问明确授权的资源
3. **资源限制** - 防止恶意代码耗尽 CPU/内存
4. **审计** - 记录所有操作用于安全分析

---

## 各平台沙箱实现

Codex 使用平台特定的沙箱技术：

| 平台 | 技术 | 说明 |
|------|------|------|
| **Linux** | Bubblewrap + Seccomp | 文件系统隔离 + 系统调用过滤 |
| **macOS** | Seatbelt (沙箱配置文件) | Apple 沙箱机制 |
| **Windows** | Windows Sandbox | Windows 虚拟化沙箱 |

### Linux 沙箱

**文件：** `codex-rs/linux-sandbox/src/`

Linux 使用 **Bubblewrap** + **Landlock** + **Seccomp**：

```
┌─────────────────────────────────────────────────────────┐
│                    Linux 系统                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Bubblewrap 沙箱                        │   │
│  │                                                  │   │
│  │  - 用户命名空间 (user namespace)                 │   │
│  │  - 网络命名空间 (network namespace)              │   │
│  │  - PID 命名空间 (process isolation)              │   │
│  │  - 文件系统视图 (只读 /proc, /sys)               │   │
│  │                                                  │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Seccomp 过滤器                       │   │
│  │                                                  │   │
│  │  允许: read, write, open, close, mmap, ...      │   │
│  │  禁止: mount, syslog, reboot, ...               │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Landlock 规则                        │   │
│  │                                                  │   │
│  │  只允许访问授权的目录树                            │   │
│  │  deny /home/user/.ssh                            │   │
│  │  allow /home/user/code/**                        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### macOS 沙箱

macOS 使用 Apple 的 **Seatbelt** 沙箱机制：

```xml
<!-- 简化的沙箱配置文件示例 -->
<SBProfile>
    <allowed>
        <!-- 允许读取用户代码目录 -->
        <path>/Users/*/code/**</path>
        <!-- 允许执行某些命令 -->
        <command>/usr/bin/git</command>
        <command>/usr/bin/python3</command>
    </allowed>
    <denied>
        <!-- 禁止访问敏感目录 -->
        <path>/Users/*/.ssh/**</path>
        <path>/Users/*/.aws/**</path>
    </denied>
</SBProfile>
```

### Windows 沙箱

Windows 使用 **Windows Sandbox** 或 **Hyper-V** 隔离：

```
Windows 沙箱模式:
1. 轻量级 VM (Windows Sandbox) - 每次创建新环境
2. 进程隔离模式 - 用 Windows token 限制权限
```

---

## 沙箱的工作原理

### 1. 创建沙箱

```rust
// 伪代码：创建沙箱
let sandbox = Sandbox::new()
    .with_workspace("/project")      // 工作目录
    .with_permissions(vec![          // 权限配置
        Permission::Read("/project/**"),
        Permission::Write("/project/**/*.py"),
        Permission::Exec(["git", "python", "npm"]),
    ])
    .with_network(true)              // 是否允许网络
    .build()?;
```

### 2. 在沙箱中执行命令

```rust
// 在沙箱中执行命令
let result = sandbox.exec("rm -rf /").await;

// 实际执行的是:
/usr/bin/bwrap \
    --dev-bind / / \
    --ro-bind /project /project \
    --unshare-user \
    --unshare-pid \
    rm -rf /
```

**注意：** `rm -rf /` 在沙箱中不会影响真实系统！

### 3. 沙箱的文件系统视图

```
真实文件系统:
/
├── home/user/
│   ├── code/          ← 允许访问
│   ├── .ssh/          ← 禁止访问
│   └── documents/     ← 禁止访问
└── ...

沙箱内只能看到:
/
├── home/user/code/    ← 映射进来
└── (其他目录被隐藏或只读)
```

---

## 权限模型

### SandboxPermissions

```rust
pub struct SandboxPermissions {
    pub read: bool,           // 允许读取文件
    pub write: bool,          // 允许写入文件
    pub exec: bool,           // 允许执行命令
    pub network: bool,        // 允许网络访问
}
```

### PermissionProfile（权限配置）

```rust
pub struct PermissionProfile {
    pub id: String,                    // 配置文件 ID
    pub name: String,                  // 显示名称
    pub allowed_reads: Vec<PathPattern>,   // 允许读取的路径
    pub allowed_writes: Vec<PathPattern>,  // 允许写入的路径
    pub allowed_commands: Vec<String>,     // 允许执行的命令
    pub denied_paths: Vec<PathPattern>,    // 明确禁止的路径
}
```

### 路径匹配模式

```toml
# 示例权限配置
[permissions]
# 允许读取整个代码目录
allow_read = [
    "~/code/**",
    "/project/**",
    "~/code/**/*.py",
]

# 只允许写入特定类型文件
allow_write = [
    "~/code/**/*.py",
    "~/code/**/*.js",
]

# 允许执行的命令（只允许白名单）
allow_exec = [
    "git",
    "python",
    "python3",
    "npm",
    "cargo",
]

# 明确禁止（优先级最高）
deny = [
    "~/.ssh/**",
    "~/.aws/**",
    "/etc/passwd",
]
```

---

## 网络隔离

沙箱可以控制网络访问：

```rust
// 禁用网络
sandbox.with_network(false);

// 启用网络（但只允许特定域）
sandbox.with_allowed_hosts([
    "api.github.com",
    "pypi.org",
    "npmjs.org",
]);
```

### Linux 网络命名空间

```bash
# 沙箱禁用网络后的效果
$ curl https://google.com
curl: (7) Couldn't connect to server

# 因为沙箱使用独立的网络命名空间
# 没有外网连接权限
```

---

## 执行流程完整图

```
用户输入: "帮我读取 config.py"
    │
    ▼
ToolRouter 选择 Read 工具
    │
    ▼
权限检查:
  - 用户是否授权读取这个路径？
  - 这个路径在允许列表中吗？
    │
    ▼
创建 ExecRequest:
  {
    command: ["cat", "config.py"],
    cwd: "/project",
    sandbox: LinuxSandbox,
    permissions: {read: true, write: false, ...}
  }
    │
    ▼
沙箱执行:
  bwrap \
    --dev-bind / / \
    --ro-bind /project /project \
    --unshare-user --unshare-pid \
    --seccomp 2 \
    cat config.py
    │
    ▼
捕获输出，返回结果
    │
    ▼
清理沙箱
```

---

## 安全机制层次

```
┌─────────────────────────────────────────────────────────┐
│                    第一层：用户认证                       │
│         用户必须登录才能使用 Codex                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    第二层：权限配置                       │
│    用户配置允许访问的目录和命令                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    第三层：沙箱隔离                       │
│    即使恶意代码逃过权限限制，也被沙箱困住                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    第四层：审计日志                       │
│    所有操作都被记录，异常行为会被检测                       │
└─────────────────────────────────────────────────────────┘
```

---

## 常见问题

### Q: 沙箱会影响性能吗？

A: 有一定开销，但很小。现代内核的命名空间切换非常快。Bubblewrap 创建一个轻量级容器，启动时间 < 50ms。

### Q: 沙箱能完全阻止恶意代码吗？

A: 没有 100% 安全的系统，但沙箱大大降低了风险：
- 防止文件系统破坏
- 防止网络攻击
- 防止权限提升
- 审计所有操作

### Q: 可以完全禁用沙箱吗？

A: 对于某些受信任的环境，可以设置 `sandbox: false`，但**不推荐**。这会完全移除安全保护。

---

## 关键代码文件

| 文件 | 职责 |
|------|------|
| `linux-sandbox/src/bwrap.rs` | Bubblewrap 封装 |
| `linux-sandbox/src/seccomp.rs` | Seccomp 过滤器 |
| `linux-sandbox/src/landlock.rs` | Landlock 规则 |
| `core/src/sandboxing/mod.rs` | 沙箱执行请求 |
| `exec-server/` | 远程执行服务器 |
| `sandboxing/` | 沙箱配置和策略 |
