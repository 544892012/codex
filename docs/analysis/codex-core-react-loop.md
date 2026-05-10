# Codex 核心 React 循环分析

## 概述

Codex 的核心是一个典型的 ReAct（Reasoning + Acting）循环：模型生成回复（可能包含工具调用）→ 执行工具 → 结果反馈给模型 → 模型继续思考或停止。

## 完整链路

```
exec (CLI入口)
  ↓ turn/start JSON-RPC
app-server (turn_processor.rs)
  ↓ Op::UserTurn
submission_loop (handlers.rs:711)
  ↓ spawn_task
RegularTask::run (regular.rs:40)
  ↓ loop { run_turn() }
run_turn (turn.rs:139) ← 主 ReAct 循环
  ↓ loop {
  │   run_sampling_request → 向模型发请求
  │   handle_output_item_done → 工具分发
  │   drain_in_flight → 结果写回历史
  │   continue 或 break
  │ }
```

## 关键文件与行号

| 环节 | 文件 | 行号 | 说明 |
|------|------|------|------|
| exec 入口 | `codex-rs/exec/src/lib.rs` | 556, 771, 833 | `run_exec_session`, 发送 TurnStart, 事件循环 |
| app-server 处理 | `codex-rs/app-server/src/request_processors/turn_processor.rs` | 315 | `turn_start_inner` 构建 Op |
| 提交循环 | `codex-rs/core/src/session/handlers.rs` | 711 | `submission_loop` 等待用户输入 |
| 任务繁衍 | `codex-rs/core/src/tasks/mod.rs` | 292, 384 | `spawn_task` → `tokio::spawn` |
| RegularTask | `codex-rs/core/src/tasks/regular.rs` | 40, 71 | 包装 `run_turn()` 的外部循环 |
| **主 ReAct 循环** | `codex-rs/core/src/session/turn.rs` | 139, 383 | `run_turn` 内的 `loop`，整个系统的核心 |
| 模型采样 | `codex-rs/core/src/session/turn.rs` | 1004, 1828 | `run_sampling_request` → 调用模型 API |
| 流处理 | `codex-rs/core/src/session/turn.rs` | 1881 | 处理模型返回的事件流 |
| 工具分发 | `codex-rs/core/src/tools/parallel.rs` | 64 | `handle_tool_call` 并行执行工具 |
| 工具路由 | `codex-rs/core/src/tools/registry.rs` | 263 | `dispatch_any` 路由到具体 handler |
| 结果反馈 | `codex-rs/core/src/session/turn.rs` | 1794 | `drain_in_flight` 等待工具完成 |
| 历史写入 | `codex-rs/core/src/session/mod.rs` | 2415 | `record_conversation_items` 写入对话历史 |

## 循环决策逻辑

`run_turn` 主循环（turn.rs:383）的核心逻辑：

```
loop {
    1. 构建上下文（指令、技能、工具列表等）
    2. run_sampling_request → 向模型发送请求
    3. 模型流式返回事件：
       - OutputItemDone(item) → 工具调用则执行，消息则记录
       - Completed { end_turn } → 模型决定是否停止
    4. drain_in_flight → 等待所有工具执行完成，结果写入历史
    5. 判断：
       - needs_follow_up == true → continue（继续循环）
       - needs_follow_up == false → 运行停止钩子 → break
}
```

## 工具执行与反馈

1. 模型返回 `OutputItemDone` 时，`handle_output_item_done`（stream_events_utils.rs:223）将工具调用转为 `ToolCall`
2. `ToolCallRuntime::handle_tool_call`（parallel.rs:64）在 `tokio::spawn` 中执行工具
3. `drain_in_flight`（turn.rs:1794）等待所有工具完成
4. 结果通过 `record_conversation_items`（session/mod.rs:2415）写入内存中的对话历史
5. 下一轮 `run_sampling_request` 构建 prompt 时，历史已包含工具结果，模型据此继续推理

## 三层架构

```
exec (lib.rs)          ← 非交互式，等待事件推送，TurnCompleted 时退出
  ↕ JSON-RPC
app-server             ← 包装 ThreadManager，协议转换
  ↕
core (turn.rs:383)     ← 真正的 ReAct 循环，驱动模型和工具
```
