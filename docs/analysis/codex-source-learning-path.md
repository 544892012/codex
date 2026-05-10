# Codex 源码学习路径

## 阅读顺序（由浅入深）

### 第1层：基础类型 — `codex-rs/protocol/`
零依赖 crate，只有共享类型定义（`ThreadId`、`SessionId`、`Op`、`Event` 等）。先搞清楚核心概念叫什么、长什么样，后续读其他 crate 才能看懂类型。

### 第2层：完整链路 — `codex-rs/exec/`
约 1000 行，无头模式的入口。从 `main.rs` → `cli.rs` → `lib.rs` 顺着读：
- 如何通过 JSON-RPC 驱动核心引擎
- exec 模式自动拒绝哪些交互请求
- JSONL 事件流的格式

### 第3层：配置系统 — `codex-rs/config/`
多层 TOML 合并逻辑，理解 MCP 配置、profile 切换、`.rules` 文件的加载机制。

### 第4层：核心循环
顺着 `codex-rs/core/src/session/turn.rs:383` 的主循环往下读：
- `run_turn` → `run_sampling_request` → `try_run_sampling_request`（模型调用 + 流处理）
- `codex-rs/core/src/tools/parallel.rs` + `registry.rs`（工具分发与执行）
- `drain_in_flight`（工具结果写回对话历史）

### 第5层：按兴趣深入子系统
| 关注点 | 入口文件 |
|--------|----------|
| MCP 集成 | `codex-rs/codex-mcp/src/connection_manager.rs` |
| 沙箱 | `codex-rs/sandboxing/` |
| SQLite 持久化 | `codex-rs/state/` |
| 生命周期钩子 | `codex-rs/hooks/` |
| 插件系统 | `codex-rs/core-plugins/src/manager.rs` |
| 执行策略 DSL | `codex-rs/execpolicy/` |

## 实操建议

1. **别从头啃 core** — `codex-core` 有 220K+ 行，是最大的 crate，CLAUDE.md 也明确说"抵制往 core 加代码"
2. **加日志跑起来** — 在 `turn.rs` 的主循环里加 `tracing::info!`，跑 `codex exec "hello"` 看日志，比干读代码直观
3. **挑个小 bug 修** — 从 `codex-rs/exec/` 或 `codex-rs/config/` 找 issue，边修边理解

## 项目架构速查

```
codex-rs/
├── protocol/          ← 共享类型（起点）
├── config/            ← 配置加载
├── features/          ← 功能标志注册表
├── core/              ← 中心引擎（最大，避免新增代码）
├── core-api/          ← 公共 API facade
├── app-server/        ← JSON-RPC 服务器
├── app-server-protocol/ ← RPC 协议定义
├── tui/               ← 终端 UI（ratatui）
├── exec/              ← 无头执行模式（DAG 集成入口）
├── exec-server/       ← 命令/文件系统执行服务器
├── mcp-server/        ← 独立 MCP 服务器入口
├── codex-mcp/         ← MCP 集成层
├── sandboxing/        ← 跨平台沙箱
├── state/             ← SQLite 持久化
├── hooks/             ← 生命周期钩子
├── plugin/            ← 插件标识符和元数据
├── core-plugins/      ← 插件管理
├── core-skills/       ← 技能管理
├── execpolicy/        ← 执行策略 DSL
└── cli/               ← CLI 入口
```
