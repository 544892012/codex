## 安装与构建

### 系统要求

| 要求 | 详情 |
| --------------------------- | --------------------------------------------------------------- |
| 操作系统 | macOS 12+、Ubuntu 20.04+/Debian 10+，或通过 **WSL2** 运行 Windows 11 |
| Git（可选，推荐） | 2.23+，用于内置 PR 辅助功能 |
| 内存 | 最低 4 GB（推荐 8 GB） |

### DotSlash

GitHub Release 中包含了 Codex CLI 的 [DotSlash](https://dotslash-cli.com/) 文件，名为 `codex`。使用 DotSlash 文件可以将一个轻量级的提交纳入版本控制，确保所有贡献者使用相同版本的可执行文件，无论他们使用什么平台进行开发。

### 从源码构建

```bash
# 克隆仓库并进入 Cargo workspace 根目录
git clone https://github.com/openai/codex.git
cd codex/codex-rs

# 安装 Rust 工具链（如尚未安装）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustup component add rustfmt
rustup component add clippy
# 安装 workspace justfile 使用的辅助工具：
cargo install --locked just
# 可选：安装 nextest 以使用 `just test` 辅助命令
cargo install --locked cargo-nextest

# 构建 Codex
cargo build

# 使用示例提示词启动 TUI
cargo run --bin codex -- "explain this codebase to me"

# 修改代码后，使用根目录 justfile 辅助命令（默认在 codex-rs 目录下执行）：
just fmt
just fix -p <你修改的-crate>

# 运行相关测试（按 crate 指定最快），例如：
cargo test -p codex-tui
# 如果安装了 cargo-nextest，`just test` 会通过 nextest 运行测试套件：
just test
# 日常本地运行避免使用 `--all-features`，因为它会增加构建时间和 target/ 磁盘占用。
# 如果需要完整的功能覆盖，使用：
cargo test --all-features
```

## 追踪与详细日志

Codex 使用 Rust 编写，因此支持通过 `RUST_LOG` 环境变量配置日志行为。

TUI 模式默认 `RUST_LOG=codex_core=info,codex_tui=info,codex_rmcp_client=info`，日志默认写入 `~/.codex/log/codex-tui.log`。单次运行时，可通过 `-c log_dir=...` 覆盖日志目录（例如 `-c log_dir=./.codex-log`）。

```bash
tail -F ~/.codex/log/codex-tui.log
```

相比之下，非交互模式（`codex exec`）默认 `RUST_LOG=error`，日志直接打印到终端，和正常输出混在一起，不需要像 TUI 那样另外监控日志文件。

更多配置选项请参阅 Rust 文档中关于 [`RUST_LOG`](https://docs.rs/env_logger/latest/env_logger/#enabling-logging) 的说明。
