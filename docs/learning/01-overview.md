# Codex 架构学习笔记

> 开始日期：2026-08-21
> 项目：https://github.com/openai/codex

## 项目概述

**Codex CLI** 是 OpenAI 的本地编码 Agent，运行在用户本地机器上。这是一个大型 monorepo，包含 CLI 应用、Python/TypeScript SDK、以及大量的 Rust 后端服务。

## 技术架构总览

```
codex (用户命令)
    │
    ▼
codex-cli (Node.js 包装层 - 平台检测+binary分发)
    │
    ▼
codex-rs (Rust 主代码库)
    ├── cli          # 命令行入口 (main.rs 171K 行)
    ├── tui          # 终端 UI (152 个子目录, 235K 行)
    ├── app-server   # 后端服务 (处理请求/扩展/消息)
    ├── exec-server  # 沙箱执行环境
    ├── core         # 核心逻辑 (Agent/工具/会话/上下文)
    └── 137 个 crates
```

## 关键目录说明

| 目录 | 用途 |
|------|------|
| `codex-rs/core/` | Agent 逻辑、工具执行、上下文管理 |
| `codex-rs/core/agent/` | Agent 行为编排 |
| `codex-rs/core/tools/` | 内置工具实现 (33 个子目录) |
| `codex-rs/core/ext/` | 扩展系统 (mcp, web-search, image-generation 等) |
| `codex-cli/` | Node.js 入口包装 |
| `sdk/` | Python + TypeScript SDK |
| `tools/` | 开发工具 (linter, buildifier) |

## 技术栈

- **语言：** Rust (主) + TypeScript + Python
- **构建：** Bazel + Cargo + pnpm
- **协议：** gRPC, Protocol Buffers, WebSocket
- **沙箱：** Linux sandbox, Seatbelt (macOS)

## 构建系统

### Bazel (`MODULE.bazel`, `BUILD.bazel`)
- 主构建系统
- 使用 `rules_rs` 构建 Rust
- 配置了 hermetic LLVM toolchain

### Cargo (`codex-rs/Cargo.toml`)
- Rust workspace，137 个 member crates
- Edition 2024 Rust

### Just (`justfile`)
- 开发者命令：
  - `just codex` - 运行 Codex
  - `just test` - 用 nextest 运行测试
  - `just fmt` - 格式化代码
  - `just clippy` / `just fix` - 代码检查

## 关键入口点

1. **`codex` 命令** - 用户-facing CLI
   - 入口：`codex-cli/bin/codex.js` (Node.js wrapper)
   - 分发到 Rust binary

2. **`cargo run --bin codex`** - 直接运行 Rust
   - Binary 定义在 `codex-rs/cli/`

3. **`codex exec`** - 在沙箱中执行命令
   - 使用 `exec-server` crate

4. **`codex app`** - 桌面应用体验
   - 通过 `app-server` 路由

## Mental Model

```
User runs `codex` or `codex-cli/bin/codex.js`
         │
         ▼
   ┌─────────────┐
   │  codex-cli  │ (Node.js wrapper - platform detection)
   └─────────────┘
         │
         ▼
   ┌─────────────┐
   │  codex-rs   │
   │    cli      │ (Command parsing, main.rs 171K lines)
   └─────────────┘
         │
         ├──────────────────────┬─────────────────────┐
         ▼                      ▼                     ▼
   ┌──────────┐          ┌──────────┐         ┌──────────┐
   │   tui    │          │app-server│         │   exec   │
   │(Terminal │          │(Backend  │         │ -server  │
   │   UI)    │          │ Server)  │         │(Sandbox) │
   └──────────┘          └──────────┘         └──────────┘
         │                     │                     │
         ▼                     ▼                     ▼
   ┌──────────────────────────────────────────────────────┐
   │                        core                            │
   │  (Agent logic, tools, context, session management)    │
   └──────────────────────────────────────────────────────┘
         │
         ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │  config  │     │  ext/    │     │  tools/  │
   │          │     │(plugins) │     │          │
   └──────────┘     └──────────┘     └──────────┘
```

## 待深入模块

- [ ] `codex-rs/core/agent/` - Agent 核心逻辑
- [ ] `codex-rs/core/tools/` - 内置工具实现
- [ ] `codex-rs/core/ext/` - 扩展系统
- [ ] `codex-rs/core/session/` - 会话管理
- [ ] `codex-rs/core/context/` - 上下文管理
- [ ] `codex-rs/tui/` - 终端 UI 实现
- [ ] `codex-rs/app-server/` - 后端服务
