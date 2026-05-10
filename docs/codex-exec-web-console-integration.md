# codex exec 输出在 Web 管理台的渲染方案

## 最终方案

```
codex exec --json "prompt" > /tmp/codex_output.jsonl
       │
       └─ stdout → JSONL 文件，所有模型输出都在这一条流里

         ↓ 逐行解析这同一个文件

Python 层
       ├─ 格式化为 ANSI 彩色文本 → WebSocket 推送 → 前端 xterm.js 渲染
       └─ 提取结构化数据 → diff 视图、状态卡片等

前端 (浏览器)
       └─ xterm.js (npm 包 @xterm/xterm)，与 codex 零耦合
```

**就一个文件**：`--json` 模式下，模型回复、推理过程、命令执行、文件修改全部混在 stdout 这一个 JSONL 流里，逐行解析即可。stderr 默认 empty，可以不管。

---

## 二、实现代码

### Python 层：JSONL → ANSI 格式化

```python
import json

def format_event_for_terminal(event: dict) -> str:
    """单个 JSONL event → ANSI 彩色终端文本"""
    G, C, Y, R, D, B, Z = "\033[32m", "\033[36m", "\033[33m", "\033[31m", "\033[2m", "\033[1m", "\033[0m"
    t = event["type"]

    if t == "turn.started":
        return f"\n{C}━━━ Turn Started ━━━{Z}\n"
    if t == "turn.completed":
        u = event.get("usage", {})
        return f"\n{G}✓ 完成{Z}  {D}输入: {u.get('input_tokens', '?')} | 输出: {u.get('output_tokens', '?')} tokens{Z}\n"
    if t == "turn.failed":
        return f"\n{R}✗ 执行失败: {event['error']['message']}{Z}\n"
    if t in ("item.started", "item.completed"):
        item = event["item"]
        detail = item["details"]
        dtype = detail["type"]
        icon = {"command_execution": "⚡", "file_change": "📄",
                "web_search": "🔍", "todo_list": "📋", "mcp_tool_call": "🔌"}.get(dtype, "•")

        if t == "item.started":
            return f"  {C}{icon} {dtype}{Z} ...\n"

        # item.completed
        if dtype == "command_execution":
            rc = detail.get("exit_code")
            s = f"{G}✓{Z}" if rc == 0 else f"{R}✗{Z}"
            return f"  {s} $ {detail['command']}\n{detail.get('aggregated_output', '')}\n{D}  exit_code={rc}{Z}\n"
        if dtype == "agent_message":
            return f"\n{G}{detail['text']}{Z}\n"
        if dtype == "reasoning":
            return f"{D}{detail['text']}{Z}\n"
        if dtype == "file_change":
            lines = [f"  {'+' if c['kind']=='add' else '-' if c['kind']=='delete' else '~'} {c['path']}"
                     for c in detail.get("changes", [])]
            return "\n".join(lines) + "\n"
    return ""
```

### LangGraph 节点集成

```python
import asyncio, json

async def codex_node(prompt: str, node_id: str, ws):
    """DAG 节点：执行 codex exec，JSONL 落盘 + 推前端终端"""

    jsonl_path = f"/tmp/codex_{node_id}.jsonl"

    # 执行 codex，JSONL 落盘
    proc = await asyncio.create_subprocess_exec(
        "codex", "exec", "--json", prompt,
        stdout=open(jsonl_path, "w"),
    )
    await proc.wait()

    # 2. 逐行解析 → 推前端 xterm.js
    items: dict[str, dict] = {}
    with open(jsonl_path) as f:
        for line in f:
            event = json.loads(line)
            # 推送 ANSI 文本到终端
            ansi = format_event_for_terminal(event)
            if ansi:
                await ws.send_text(ansi)
            # 维护 item 状态
            if event["type"] in ("item.started", "item.updated", "item.completed"):
                items[event["item"]["id"]] = event["item"]

    # 3. 最终 AI 回复（最后一个 agent_message）
    messages = [v["details"]["text"] for v in items.values()
                if v["details"]["type"] == "agent_message"]
    last_message = messages[-1] if messages else ""

    return {"items": list(items.values()), "last_message": last_message,
            "exit_code": proc.returncode}
```

### 前端：xterm.js 接收

```typescript
import { Terminal } from "@xterm/xterm";
import "@xterm/xterm/css/xterm.css";

const term = new Terminal({ convertEol: true, fontSize: 13 });
term.open(document.getElementById("terminal"));

const ws = new WebSocket("ws://localhost:8000/node/exec-stream");
ws.onmessage = (e) => term.write(e.data);
```

---

## 三、为什么不能复用 codex TUI

TUI (`codex-rs/tui/`) 和 exec (`codex-rs/exec/`) 是**对等的消费者**，共享同一个 `InProcessAppServerClient`：

```
                    ┌──────────────────────────┐
                    │ InProcessAppServerClient   │
                    └──────────┬───────────────┘
               ┌───────────────┴───────────────┐
       ┌───────▼──────┐               ┌───────▼──────┐
       │   codex tui   │               │  codex exec  │
       │  交互式终端     │               │  非交互式     │
       │  渲染: ratatui │               │  输出: JSONL  │
       │  目标: 终端     │               │  目标: stdout │
       └───────────────┘               └───────────────┘
```

| 阻隔 | 原因 |
|------|------|
| **技术栈** | TUI 是 Rust + ratatui（终端 UI），Web 是 JS + DOM，无法互操作 |
| **交互模式** | TUI 处理键盘输入、审批弹窗；exec 下 ServerRequest 全部自动拒绝 |
| **耦合度** | `chatwidget.rs`（239K 行）直接依赖 `ThreadManager`，无独立渲染接口 |
| **强制方案** | 跑交互版 `codex` + PTY + xterm.js = 浏览器里开远程终端，不适用于 DAG 自动流水线 |

---

## 四、JSONL 事件速查

### 顶层事件

| type | 含义 | 关键字段 |
|------|------|---------|
| `thread.started` | 线程创建 | `thread_id` |
| `turn.started` | prompt 已提交 | — |
| `turn.completed` | 本轮完成 | `usage: {input_tokens, output_tokens}` |
| `turn.failed` | 本轮失败 | `error.message` |
| `item.started` | 子任务开始 | `item: {id, details}` |
| `item.updated` | 子任务更新 | `item: {id, details}` |
| `item.completed` | 子任务完成 | `item: {id, details}` |
| `error` | 致命错误 | `message` |

### item 详情类型 (`details.type`)

| type | 关键字段 |
|------|---------|
| `agent_message` | `text` — AI 最终文本回复 |
| `reasoning` | `text` — AI 推理过程 |
| `command_execution` | `command`, `aggregated_output`, `exit_code`, `status` |
| `file_change` | `changes: [{path, kind: add/delete/update}]`, `status` |
| `web_search` | `query`, `action: search/fetch` |
| `todo_list` | `items: [{text, completed}]` — 唯一会多次更新的类型 |
| `mcp_tool_call` | `server`, `tool`, `arguments`, `result`, `error`, `status` |
| `error` | `message` |

---

## 五、前端组件映射

| item type | 前端组件 | 推荐库 |
|-----------|---------|--------|
| `agent_message` | Markdown 渲染 | `react-markdown` |
| `reasoning` | 可折叠面板（默认收起） | `<details>` |
| `command_execution` | 终端块（黑底绿字） | xterm.js 或纯 CSS |
| `file_change` | Diff 视图 | `react-diff-viewer` |
| `todo_list` | 任务进度条 | `CheckList` |
| `web_search` | 搜索结果卡片 | 自定义 Card |
| `error` | 红色告警 | `Alert variant="error"` |

```typescript
const statusColor = {
  in_progress: "#1890ff",
  completed:   "#52c41a",
  failed:      "#ff4d4f",
  declined:    "#faad14",
};
```

---

## 六、LangGraph 完整数据流

```
LangGraph DAG
  │
  ├─ Node A: codex exec --json "需求分析"  →  JSONL  →  items  →  State
  ├─ Node B: codex exec --json "编码实现"   →  JSONL  →  items  →  State
  └─ Node C: codex exec --json "编写测试"   →  JSONL  →  items  →  State
       │
       └─ 每个 node 完成时 → WebSocket → Web 管理台
                                   ├─ xterm.js 终端视图 (ANSI 文本)
                                   └─ 结构化视图 (diff, 状态卡片)
```
