# codex exec 执行路径分析

## 一、架构总览

`codex exec` 是非交互式 AI 编程代理的入口。用户输入一个 prompt，获得完整输出后退出——没有交互式对话，没有人工审批。

### 三层架构（由顶向下）

```
┌─────────────────────────────────┐
│  用户入口层 (exec crate)         │  CLI 解析 → 事件循环 → 格式化输出
├─────────────────────────────────┤
│  JSON-RPC 桥接层 (app-server)    │  进程内 RPC，包装 ThreadManager
├─────────────────────────────────┤
│  核心引擎层 (core crate)         │  ThreadManager → Session → 模型调用
└─────────────────────────────────┘
```

> **关键设计**：exec 不直接调用 `ThreadManager`。它通过 `InProcessAppServerClient` 发送 JSON-RPC 请求，app-server 持有 `ThreadManager` 实例并执行实际工作。这个分层让 exec 和交互式 TUI 共享同一套核心逻辑。

---

## 二、完整执行路径（按调用时序）

### 阶段 0：入口分发

```
用户命令: codex exec "帮我写一个排序函数"
         │
         ▼
codex-rs/cli/src/main.rs:796  main()
         │
         ├─  arg0_dispatch_or_else()        判断当前二进制身份 (codex / codex-exec / ...)
         │
         ├─  cli_main()                     解析 MultitoolCli 枚举
         │    └─  Subcommand::Exec(exec_cli)
         │
         └─  codex_exec::run_main(exec_cli, arg0_paths)
```

### 阶段 1：初始化 (`run_main`)

文件：`codex-rs/exec/src/lib.rs:233`

```
run_main(cli, arg0_paths)
  │
  ├─ 1. 加载 config.toml，合并 CLI 覆盖项 (--model, --sandbox 等)
  ├─ 2. 检查 OSS provider、login 限制、执行策略
  ├─ 3. 初始化 OpenTelemetry + tracing subscriber
  ├─ 4. 初始化 SQLite 状态数据库 (codex-state)
  ├─ 5. 创建 EnvironmentManager
  ├─ 6. 构建 InProcessClientStartArgs (包含上述所有上下文)
  │
  └─ 调用 run_exec_session(args)
```

### 阶段 2：事件循环 (`run_exec_session`)

文件：`codex-rs/exec/src/lib.rs:556`

```
run_exec_session(args)
  │
  ├─ 1. 创建 EventProcessor
  │     ├─ 默认: EventProcessorWithHumanOutput  → 人类可读输出，写入 stderr/stdout
  │     └─ --json: EventProcessorWithJsonOutput → JSONL 事件流，写入 stdout
  │
  ├─ 2. 确定 InitialOperation
  │     ├─ UserTurn → 正常用户 prompt
  │     └─ Review   → 代码审查
  │
  ├─ 3. InProcessAppServerClient::start()
  │     └─ 在后台线程中启动 app-server 运行时
  │
  ├─ 4. 发送 RPC: thread/start  (或 thread/resume)
  │     └─ 创建或恢复一个 Thread → 获得 ThreadId
  │
  ├─ 5. 发送 RPC: turn/start (或 review/start)
  │     └─ 将用户的 prompt 提交给核心引擎，开始一轮 agent 工作
  │
  ├─ 6. 事件循环 (tokio::select!)
  │     │
  │     ├─ 监听 ctrl+c → 发送 turn/interrupt
  │     │
  │     └─ client.next_event() → InProcessServerEvent
  │          │
  │          ├─ ServerRequest    → handle_server_request()  【全部拒绝】
  │          │   exec 是非交互式的，不会等待用户授权
  │          │
  │          ├─ ServerNotification → event_processor.process()
  │          │   └─ 返回 CodexStatus::Running 或 InitiateShutdown
  │          │
  │          └─ Lagged           → 记录警告日志
  │
  ├─ 7. 当收到 TurnCompleted → CodexStatus::InitiateShutdown → 退出循环
  │
  ├─ 8. client.shutdown()
  └─ 9. event_processor.print_final_output()
```

### 阶段 3：核心引擎处理 (app-server → core)

exec 发送 `turn/start` 后，app-server 内部执行：

```
app-server 收到 turn/start
  │
  └─ ThreadManager::start_thread_with_options()    thread_manager.rs:500
       │
       └─ ThreadManagerState::spawn_thread_with_source()   thread_manager.rs:1076
            │
            └─ Codex::spawn(args)                  session/mod.rs:424
                 │
                 ├─ 创建提交/事件通道 (tx_sub/rx_sub, tx_event/rx_event)
                 ├─ 解析模型信息、指令、协作模式
                 ├─ 构造 Session 对象
                 │
                 └─ 启动 submission_loop (后台 tokio 任务)   session/handlers.rs:711
```

#### submission_loop — 操作分发中心

```
submission_loop
  │
  └─ while let Ok(sub) = rx_sub.recv().await
       │
       └─ match sub.op
            ├─ Op::UserTurn { .. } → user_input_or_turn()
            └─ Op::Shutdown       → 退出循环
```

#### user_input_or_turn — 创建 Turn 上下文

```
user_input_or_turn(&sess, sub_id, op)           session/handlers.rs:99
  │
  ├─ sess.new_turn_with_sub_id(sub_id)          创建 TurnContext
  │   └─ 应用 session 配置覆盖、解析环境、构建上下文
  │
  └─ sess.spawn_task(turn_context, input, RegularTask)   tasks/mod.rs:292
       │
       └─ 创建新的 tokio 任务 → SessionTask::run()
```

### 阶段 4：轮次执行 (RegularTask → run_turn)

文件：`tasks/regular.rs` → `session/turn.rs:139`

```
RegularTask::run()
  │
  ├─ 发射 TurnStarted 事件
  └─ 循环调用 run_turn()   (当需要 follow-up 时重新进入)
       │
       ├─ 1. pre-sampling 压缩检查（如果 token 超限）
       ├─ 2. 将用户消息写入对话历史
       ├─ 3. 运行 hooks (PreToolUse, PostToolUse 等)
       │
       └─ 4. 采样循环: run_sampling_request()     turn.rs:1004
            │
            ├─ 构建 ToolRouter（聚合所有可用工具）
            │   ├─ 内置工具 (shell, file, web_search...)
            │   ├─ MCP 工具
            │   ├─ Plugin 工具
            │   └─ Skill 工具
            │
            ├─ 创建 ToolCallRuntime
            │
            └─ try_run_sampling_request()          turn.rs:1828
                 │
                 ├─ client_session.stream(prompt, model_info)
                 │   │
                 │   └─ 调用模型 API (Responses / WebSocket)
                 │        └─ 将对话历史 + 工具规格发送给模型
                 │
                 └─ 处理流式响应
                      │
                      ├─ OutputItemAdded    → 开始跟踪新项目
                      ├─ OutputTextDelta    → 流式文本增量
                      ├─ OutputItemDone     → 分发工具调用
                      │   └─ handle_output_item_done()
                      │        ├─ 判断是否为工具调用
                      │        ├─ 写入历史
                      │        └─ 创建 Future: tool_runtime.handle_tool_call()
                      │
                      └─ Completed { end_turn }
                           ├─ end_turn=true  → 本轮完成
                           └─ end_turn=false → 需要 follow-up (模型调用了工具)
```

### 阶段 5：工具调用执行

```
handle_tool_call(call)                          tools/parallel.rs:64
  │
  ├─ 检查是否支持并行执行
  │   ├─ 不支持 → 获取写锁，串行执行
  │   └─ 支持   → 直接 spawn tokio 任务
  │
  └─ router.dispatch_tool_call_with_code_mode_result()   tools/router.rs:269
       │
       └─ registry.dispatch_any(invocation)              tools/registry.rs:263
            │
            ├─ 查找工具处理器 (ToolName → handler)
            ├─ 运行 PreToolUse hooks
            ├─ 执行工具:
            │   ├─ FunctionTool → 内置实现 (shell 命令、文件操作等)
            │   └─ MCP 工具     → 转发到 MCP 连接管理器
            ├─ 运行 PostToolUse hooks
            │
            └─ 返回 AnyToolResult → 转为 ResponseInputItem
                 │
                 └─ 注入对话历史 → 采样循环继续
                    (模型看到工具输出，决定下一步)
```

### 阶段 6：ToolOrchestrator — 审批与沙箱

文件：`tools/orchestrator.rs`

每次工具调用都会经过 `ToolOrchestrator`：

```
ToolOrchestrator
  │
  ├─ 1. 审批检查
  │   ├─ 策略引擎 (execpolicy DSL)
  │   └─ 如需人工审批 → 发出 ServerRequest
  │       └─ exec 模式下自动拒绝 (非交互式)
  │
  ├─ 2. 沙箱选择
  │   └─ SandboxManager 根据许可配置决定:
  │       ├─ 只读沙箱
  │       ├─ 工作区写入沙箱
  │       └─ 完整访问
  │
  └─ 3. 重试逻辑 (被拒绝时升级沙箱重试)
```

---

## 三、exec 的交互限制

由于 exec 是非交互式的，它**全部拒绝**以下服务器请求（`lib.rs:1528`）：

| 请求类型 | 处理方式 |
|---------|---------|
| MCP 权限请求 | 自动取消 |
| 命令执行审批 | 拒绝 |
| 文件操作审批 | 拒绝 |
| 动态工具调用 | 拒绝 |
| 用户输入请求 | 拒绝 |

> **这意味着**: 使用 exec 时，必须配置好审批策略和执行策略，或者使用 `--dangerously-bypass-approvals-and-sandbox` 跳过所有审批。

---

## 四、关键文件速查

| 文件 | 职责 |
|------|------|
| `codex-rs/exec/src/lib.rs` | exec 主逻辑：初始化、事件循环、服务器请求处理 |
| `codex-rs/exec/src/cli.rs` | CLI 参数定义 (clap) |
| `codex-rs/exec/src/event_processor.rs` | EventProcessor trait 定义 |
| `codex-rs/exec/src/event_processor_with_human_output.rs` | 人类可读输出处理 |
| `codex-rs/exec/src/event_processor_with_jsonl_output.rs` | JSONL 输出处理 |
| `codex-rs/exec/src/exec_events.rs` | JSONL 事件类型定义 |
| `codex-rs/cli/src/main.rs` | 顶层 CLI 入口，分发子命令 |
| `codex-rs/app-server-client/src/lib.rs` | InProcessAppServerClient，进程内 RPC 门面 |
| `codex-rs/core/src/thread_manager.rs` | ThreadManager，中心编排器 |
| `codex-rs/core/src/session/mod.rs` | Codex::spawn，session 工厂 |
| `codex-rs/core/src/session/handlers.rs` | submission_loop，操作分发 |
| `codex-rs/core/src/session/turn.rs` | run_turn，轮次执行核心 |
| `codex-rs/core/src/tools/router.rs` | 工具路由和调度 |
| `codex-rs/core/src/tools/registry.rs` | 工具注册表和 dispatch_any |
| `codex-rs/core/src/tools/parallel.rs` | 并行工具执行 |
| `codex-rs/core/src/tools/orchestrator.rs` | 审批 + 沙箱编排 |
| `codex-rs/core/src/client.rs` | ModelClient，模型 API 调用 |
| `codex-rs/protocol/src/protocol.rs` | Op/Event/Submission 类型定义 |

---

## 五、调试方法

### 5.1 控制日志输出级别（RUST_LOG）

exec 使用 `RUST_LOG` 环境变量控制日志输出，所有日志写入 **stderr**：

```bash
# 默认仅输出 error 级别
codex exec "your prompt"

# 显示 info 级别日志
RUST_LOG=info codex exec "your prompt"

# 显示 debug 日志（最详细）
RUST_LOG=debug codex exec "your prompt"

# 按模块过滤
RUST_LOG=codex_core=debug,codex_exec=info codex exec "your prompt"

# 关闭 OTEL 自身日志噪音（默认已关闭，如需开启）
RUST_LOG=debug,opentelemetry_sdk=off,opentelemetry_otlp=off codex exec "your prompt"
```

默认过滤规则（`exec/src/lib.rs:158`）：
```
error,opentelemetry_sdk=off,opentelemetry_otlp=off
```

### 5.2 JSON 模式（查看结构化事件）

```bash
codex exec --json "your prompt"
```

输出 JSONL 到 stdout，每行一个事件，包含：
- `ThreadStarted` — 线程创建
- `TurnStarted` — 轮次开始
- `ItemStarted` / `ItemCompleted` — 工具调用、文件修改等
- `TurnCompleted` — 轮次完成
- `Error` — 错误

你可以用 `jq` 过滤：
```bash
codex exec --json "your prompt" | jq 'select(.type == "ItemCompleted")'
```

### 5.3 分布式追踪（OpenTelemetry）

exec 支持 W3C trace context 传播，方便在 LangGraph 层面串联调用链：

```bash
# 设置父 trace context
TRACEPARENT="00-<trace-id>-<span-id>-01" codex exec "your prompt"
```

这样 exec 的 OTEL span 会挂在外部 trace 下，可以在 Jaeger/Tempo 等系统中看到完整的 DAG 执行链路。

### 5.4 输出最后一条消息

```bash
codex exec -o /tmp/last_message.txt "your prompt"
```

将 agent 最后一条消息写入文件，方便后续节点读取。

### 5.5 在 LangGraph 中的调试建议

结合你的 LangGraph + codex exec 架构：

```python
import subprocess
import os
import json

def run_codex_node(prompt: str, debug: bool = False) -> dict:
    env = os.environ.copy()

    if debug:
        env["RUST_LOG"] = "debug"

    cmd = ["codex", "exec", "--json", "-o", "/tmp/codex_output.txt", prompt]

    result = subprocess.run(
        cmd,
        env=env,
        capture_output=True,
        text=True
    )

    # stderr 包含 RUST_LOG 日志
    if debug:
        print("=== CODEX DEBUG LOGS ===")
        print(result.stderr)

    # stdout 包含 JSONL 事件流
    events = []
    for line in result.stdout.strip().split("\n"):
        if line:
            events.append(json.loads(line))

    # 读取最后一条消息
    with open("/tmp/codex_output.txt") as f:
        last_message = f.read()

    return {
        "events": events,
        "last_message": last_message,
        "exit_code": result.returncode
    }
```

这样你就能在 LangGraph 节点中完整捕获 codex exec 的执行过程。
