# 开发命令速查

> 本机环境：macOS 15.7 + Rust 1.95.0，所有命令在 `codex-rs/` 目录下执行。

## 构建与运行

```bash
cargo build                              # 构建（debug）
cargo build --release                    # 构建（release）
cargo run --bin codex -- "prompt"        # 启动交互式 TUI
cargo run --bin codex -- exec "prompt"   # 无头模式执行
just codex                               # 构建并运行 TUI
just exec                                # 构建并运行 exec
```

## 测试

```bash
cargo test -p <crate>                    # 按 crate 测试（最快）
just test                                # 全量测试（需 cargo-nextest）
cargo test -p codex-tui                  # TUI 测试（含快照）
cargo insta pending-snapshots -p codex-tui  # 查看待处理快照
cargo insta accept -p codex-tui          # 接受新快照
```

## Lint 与格式化

```bash
just fmt                                 # cargo fmt
just fix -p <crate>                      # clippy fix
just argument-comment-lint               # 参数注释 lint 检查
```

## 配置与协议

```bash
just write-config-schema                 # 更新 config.schema.json
just write-app-server-schema             # 更新 app-server 协议 fixtures
just bazel-lock-update                   # 更新 Bazel 锁文件
```

## 本地构建注意事项

本机 CLT 的 C++ 标准库头文件不完整，需在 `~/.cargo/config.toml` 中设置：

```toml
[env]
MACOSX_DEPLOYMENT_TARGET = "15.0"
CXXFLAGS = "-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk -cxx-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1"
CFLAGS = "-isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
```

## 二进制路径

编译产物在 `codex-rs/target/debug/codex`（debug）或 `codex-rs/target/release/codex`（release）。
