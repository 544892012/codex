# Codex 代码工程全量分析报告

> 生成日期：2026-05-09 | 分支：main | 提交：fca81eeb5b

## 一、项目概要

| 维度 | 详情 |
|------|------|
| **项目名称** | OpenAI Codex CLI |
| **代码仓库** | `openai/codex` |
| **总提交数** | 29,354 次 |
| **核心语言** | Rust（1,838 个 .rs 文件） |
| **Rust 代码量** | 约 68.4 万行 |
| **Rust 子crate数** | 90 个 |
| **许可证** | Apache-2.0 |
| **活跃贡献者** | 20+ 名核心开发者 |
| **首位贡献者** | Michael Bolin（9,963 次提交） |
| **2026年提交** | 3,734 次（日均约30次） |

## 二、顶层目录结构

```
codex/
├── codex-rs/          # 核心代码：90个 Rust crate 工作区
├── codex-cli/          # npm 元包，通过 bin/codex.js 启动原生二进制
├── sdk/                # SDK 层
│   ├── python/         #   Python SDK（JSON-RPC v2 over stdio）
│   ├── python-runtime/ #   Python 运行时包（捆绑原生二进制）
│   └── typescript/     #   TypeScript SDK（子进程+JSONL协议）
├── docs/               # 文档
├── scripts/            # 构建/开发/CI 辅助脚本
├── patches/            # 29个跨平台兼容性补丁
├── .github/            # CI/CD（19个工作流）
├── .codex/             # 本地技能和环境配置
├── third_party/        # V8、wezterm 等第三方集成
└── .devcontainer/      # 开发容器配置
```

## 三、Rust 核心架构

### 3.1 分层架构图

```
                      codex-cli (顶层二进制入口)
                     /       |         \
                    /        |          \
              codex-tui   codex-exec   codex-mcp-server
                 |            |              |
                 +------+-----+              |
                        |                    |
                codex-app-server             |
                   |         |               |
        codex-app-server-client             |
                   |                        |
                codex-core  ←──────────────+
               /    |     \
              /     |      \
     codex-config  |  codex-protocol
          |        |        |
     codex-mcp     |   codex-hooks
     codex-execpolicy  codex-sandboxing
     codex-model-provider  codex-features
     codex-exec-server  codex-core-skills
     codex-core-plugins  codex-api
     codex-rmcp-client  codex-state
```

### 3.2 各层职责

#### 第1层：基础层（零内部依赖）
- **protocol**（17,546行）：共享类型定义（ThreadId、SessionId、权限模型、Op/EventMsg事件系统）
- **features**（2,004行）：约60个功能标志的中央注册表
- **utils/***（24个子crate）：absolute-path、cargo-bin、cache、image、pty 等基础工具

#### 第2层：配置与基础设施层
- **config**（13,046行）：多源配置加载与合并（全局/项目/个人/云端/CLI覆盖）
- **hooks**（9,006行）：生命周期钩子引擎，支持8种事件
- **sandboxing**（4,926行）：跨平台沙箱抽象（Linux bubblewrap/landlock、macOS seatbelt、Windows）
- **execpolicy**（2,753行）：执行策略DSL解析引擎
- **model-provider**（1,737行）：模型提供商抽象（OpenAI API Key、ChatGPT、Bedrock、Ollama、LM Studio）
- **codex-api**（10,526行）：后端API客户端（Responses API、compaction、SSE/WebSocket）
- **codex-mcp**（5,337行）：MCP集成层（连接管理、工具发现、OAuth权限获取）
- **rmcp-client**（7,602行）：Rust MCP 协议客户端库
- **state**（14,336行）：SQLite 持久化存储（线程元数据、回填、迁移）
- **login**（9,193行）：多方式认证

#### 第3层：核心引擎
- **core**（219,986行，最大crate）：线程生命周期管理、模型交互、工具执行、补丁应用、钩子调度、MCP管理、技能加载、沙箱协调、消息压实、Web搜索。核心抽象：`ThreadManager`、`CodexThread`、`McpManager`、`ModelClient`

#### 第4层：服务器层
- **app-server**（86,703行，第二大crate）：JSON-RPC应用服务器，三种传输（stdio/Unix Socket/WebSocket）
- **app-server-protocol**（23,621行）：JSON-RPC v1/v2 协议类型定义，自动生成 TypeScript 类型
- **app-server-transport**（8,791行）：传输层
- **app-server-client**（3,079行）：客户端抽象
- **exec-server**（16,287行）：命令执行和文件系统操作服务器

#### 第5层：用户界面/入口层
- **tui**（174,175行，第三大crate）：ratatui 终端UI，事件循环、Markdown渲染、协作、键位映射
- **exec**（8,457行）：无头非交互模式
- **mcp-server**（3,540行）：独立 MCP 服务器，通过 stdio 暴露 codex 能力
- **cli**（7,868行）：顶层子命令调度器

#### 第6层：功能扩展
- **core-plugins**（19,652行）：插件管理
- **core-skills**（7,088行）：技能管理
- **tools**（4,122行）：模型可用工具定义
- **shell-command**（5,968行）：Shell 执行
- **apply-patch**（5,065行）：AI 生成补丁的应用算法
- **git-utils**（2,941行）：Git 仓库检测
- **network-proxy**（9,352行）：沙箱 HTTP 代理
- **analytics**（7,791行）：遥测
- **builtin-mcps**（101行）：内置 MCP 服务器目录

## 四、SDK 层

### TypeScript SDK（`@openai/codex-sdk`）
- 通过子进程 `codex exec --experimental-json` 通信
- 核心类：`Codex`（线程管理）、`Thread`（turn执行）
- 支持流式输出、结构化输出、图片输入

### Python SDK（`openai-codex-app-server-sdk`）
- 基于 `codex app-server --listen stdio://` 的 JSON-RPC v2 协议
- 同步和异步客户端（`Codex`/`AsyncCodex`）
- 高级 API：`Thread.run()` 和 `Thread.turn()` + `TurnHandle`

### Python Runtime（`openai-codex-cli-bin`）
- 纯 wheel 包，捆绑原生二进制

## 五、构建与 CI/CD

### 双构建系统
| 系统 | 用途 |
|------|------|
| **Bazel 9.0.0** | 主构建系统：CI多平台测试、发布构建、远程执行（BuildBuddy） |
| **Cargo** | 开发者日常工作流 |

### 多平台支持
- 10个目标三元组（Linux/macOS/Windows × x86_64/aarch64 × glibc/musl/MSVC/gnullvm）
- 29个跨平台兼容性补丁

### CI/CD 流水线（19个工作流）
- `ci.yml`：核心CI
- `bazel.yml`：Bazel主CI（多平台测试+发布构建）
- `rust-ci.yml`：快速PR检查
- `rust-release.yml`：完整发布管道
- `sdk.yml`：SDK 构建和测试

## 六、关键技术和依赖

| 技术 | 说明 |
|------|------|
| 异步运行时 | Tokio（多线程） |
| 数据库 | SQLite（通过 sqlx，含迁移） |
| 终端UI | ratatui + crossterm |
| Web框架 | axum |
| 序列化 | serde + serde_json + toml + ts-rs |
| MCP协议 | rmcp |
| 沙箱技术 | bubblewrap/landlock（Linux）、seatbelt（macOS）、Windows Restricted Tokens |
| V8引擎 | v8-146.4.0（JavaScript沙箱） |
| 测试框架 | insta（快照）、wiremock（HTTP mock）、pretty_assertions |
| 遥测 | OpenTelemetry、Sentry |
| 包管理 | pnpm 10.33.0（Node）、Cargo（Rust） |

## 七、代码规模分布

```
总 Rust 代码: ~684,300 行

  core          219,986行 (32.1%)
  tui           174,175行 (25.4%)
  app-server     86,703行 (12.7%)
  app-protocol   23,621行  (3.5%)
  core-plugins   19,652行  (2.9%)
  protocol       17,546行  (2.6%)
  exec-server    16,287行  (2.4%)
  state          14,336行  (2.1%)
  config         13,046行  (1.9%)
  其余80个crate  98,948行 (14.5%)
```

前3个crate占代码总量的 69%。

## 八、架构特点总结

1. **分层清晰**：从 protocol → config → core → app-server → tui/exec 依赖方向一致
2. **crate 粒度细**：90个crate，大部分小而专注
3. **core 过大**：项目明确承认 `codex-core` 膨胀，正在引导拆分
4. **多平台投入大**：29个补丁+10个目标+三种沙箱实现
5. **协议驱动**：app-server JSON-RPC v2，支持多客户端和远程控制
6. **MCP 深度集成**：既是 MCP 服务端也可连接外部 MCP 服务器
7. **可扩展**：插件+技能两套体系
8. **沙箱优先**：默认启用沙箱执行
