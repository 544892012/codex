# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

OpenAI Codex CLI — 本地运行的AI编程代理。核心是 Rust 编写的 90-crate 工作区（`codex-rs/`），搭配 TypeScript/Python SDK 和 npm CLI 入口。

## 常用命令

以下命令运行目录均为 `codex-rs/`：

### 构建与运行

```bash
just codex          # 构建并运行 codex（等同于 cargo run --bin codex）
just exec           # 构建并运行无头模式（cargo run --bin codex exec）
cargo build -p <crate>  # 构建单个 crate
```

### 测试

```bash
cargo test -p <crate>           # 运行单个 crate 的测试（推荐先这样跑）
just test                       # 全量测试（cargo nextest run）
cargo test -p codex-tui         # TUI 测试（含快照）
cargo insta pending-snapshots -p codex-tui   # 查看待处理快照
cargo insta accept -p codex-tui             # 接受新快照
```

### Lint 与格式化

```bash
just fix -p <crate>   # 运行 clippy fix（带 --fix 标志）
just fmt              # 运行 cargo fmt（改完代码后总是运行）
just argument-comment-lint  # 参数注释 lint 检查
```

### 配置与协议更新

```bash
just write-config-schema             # 更新 config.schema.json
just write-app-server-schema         # 重新生成 app-server 协议 fixtures
just bazel-lock-update               # 更新 Bazel 锁文件
```

## 核心架构

### 分层依赖（由底向上）

1. **protocol** — 共享类型（ThreadId、SessionId、Op/Event），零内部依赖
2. **config** — 多源 TOML 配置加载、合并、验证
3. **features** — 功能标志注册表（~60个标志，含 Stable/Experimental/Deprecated 生命周期）
4. **core** — 中心编排层（219K行，最大crate）：ThreadManager 管理整个线程生命周期
5. **app-server** — JSON-RPC v2 服务器：包装 core，暴露多客户端（stdio/Unix/WS）接口
6. **tui / exec / mcp-server** — 三个用户入口：交互式终端 / 无头模式 / MCP对外服务

### 关键子系统

- **hooks** (`codex-hooks`)：8 个生命周期钩子事件（PreToolUse、PostToolUse、SessionStart 等）
- **sandboxing** (`codex-sandboxing`)：跨平台沙箱（bubblewrap/landlock/seatbelt/Windows）
- **execpolicy** (`codex-execpolicy`)：自定义 DSL 的执行策略引擎
- **codex-mcp** (`codex-mcp`)：MCP 连接管理、工具发现、OAuth — 优先改 `mcp_connection_manager.rs`
- **state** (`codex-state`)：SQLite 持久化层（sqlx + 迁移）
- **model-provider** (`codex-model-provider`)：模型认证抽象（OpenAI/ChatGPT/Bedrock/Ollama/LM Studio）

### app-server 协议约定

- v2 活跃开发，**v1 不再新增 API**
- 请求：`*Params`，响应：`*Response`，通知：`*Notification`
- 方法格式：`<resource>/<method>`（如 `thread/read`、`app/list`）
- 字段用 `camelCase`，config RPC 除外（`snake_case`）
- 新列表方法必须实现游标分页（`cursor` + `limit` → `data` + `next_cursor`）
- 参数中可选字段：`Option<...>` + `#[ts(optional = nullable)]`
- 时间戳：`i64` Unix 秒，命名为 `*_at`
- 实验性 API：`#[experimental("...")]` + `ExperimentalApi` derive

## Rust 开发约定

### 核心原则

- **抵制往 `codex-core` 加代码** — 优先考虑拆分到已有crate或新建crate。core 已过度膨胀。
- 避免使用 `#[async_trait]` 和 `#[allow(async_fn_in_trait)]`，优先用原生 RPITIT + 显式 `Send` bound
- match 应保持穷尽，避免通配分支
- 优先私有模块和显式导出的公共 API
- 不要创建只引用一次的小辅助方法
- 模块控制在 500 LoC 以内（不含测试），超出需拆分

### 命名与风格

- crate 名以 `codex-` 为前缀（如 `codex-core`、`codex-tui`）
- 折叠 if 语句（per clippy `collapsible_if`）
- 内联 `format!` 参数（per clippy `uninlined_format_args`）
- 优先用方法引用而非闭包（per clippy `redundant_closure_for_method_calls`）

### 位置参数注释

对 `None`、布尔值、数字字面量等不透亮的位置参数，使用 `/*param_name*/` 注释：
```rust
foo(/*allow_network*/ true, /*max_retries*/ 3)
```
参数名必须和签名中的一样。用 `just argument-comment-lint` 检查。

### 测试

- 用 `pretty_assertions::assert_eq` 做深度相等比较
- TUI 变更必须有 `insta` 快照覆盖
- 集成测试优先用 `core_test_support::responses` 工具类
- 用 `wait_for_event` 而非 `wait_for_event_with_timeout`
- 用 `codex_utils_cargo_bin::cargo_bin("...")` 而非 `assert_cmd::Command::cargo_bin(...)`
- 别在测试中修改进程环境

### TUI 约定

- 用 ratatui `Stylize` trait：`"text".dim()`、`"text".red()`、`"text".cyan().underlined()`
- 别用 `.white()`，用默认前景色
- 用 `textwrap::wrap` 换行，用 `tui/src/wrapping.rs` 中的助手包装 Line
- 别往 `chatwidget.rs` 加新独立方法（已有239K行），优先用新模块

### 依赖变更后

- 运行 `just bazel-lock-update` 更新 `MODULE.bazel.lock`
- 运行 `just bazel-lock-check` 检查锁文件一致性
- 如果用到了 `include_str!`/`include_bytes!`/`sqlx::migrate!`，记得更新 crate 的 `BUILD.bazel` 中的 `compile_data`/`build_script_data`

## 目录速查

| 路径 | 内容 |
|------|------|
| `codex-rs/core/` | 中心引擎（最大crate，避免新增代码） |
| `codex-rs/core-api/` | 薄层公共 API facade |
| `codex-rs/tui/` | 终端 UI（ratatui） |
| `codex-rs/app-server/` | JSON-RPC 应用服务器 |
| `codex-rs/app-server-protocol/` | JSON-RPC v1/v2 协议定义 |
| `codex-rs/cli/` | `codex` 二进制入口 |
| `codex-rs/exec/` | 无头执行模式 |
| `codex-rs/exec-server/` | 命令/文件系统执行服务器 |
| `codex-rs/mcp-server/` | 独立 MCP 服务器入口 |
| `codex-rs/codex-mcp/` | MCP 集成层 |
| `codex-rs/sandboxing/` | 跨平台沙箱抽象 |
| `codex-rs/plugin/` | 插件标识符和元数据 |
| `codex-rs/core-plugins/` | 插件管理 |
| `codex-rs/core-skills/` | 技能管理 |
| `codex-rs/state/` | SQLite 持久化 |
| `codex-rs/protocol/` | 共享类型（根级） |
| `codex-rs/config/` | 配置加载 |
| `sdk/typescript/` | TypeScript SDK |
| `sdk/python/` | Python SDK |
| `sdk/python-runtime/` | Python 运行时包 |
| `codex-cli/` | npm 元包入口 |
| `docs/analysis/` | 项目分析报告 |
