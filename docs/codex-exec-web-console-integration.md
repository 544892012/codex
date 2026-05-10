# codex exec 输出在 Web 管理台的渲染方案

## 一、数据来源：stdout 与 stderr 的分工

### 1.1 --json 模式（推荐）

结构化事件写入 **stdout**，日志写入 **stderr**，互不污染。

```
codex exec --json "实现一个排序函数"
  ├─ stdout (JSONL, 逐行流式输出)
  │   {"type":"thread.started","thread_id":"018f..."}
  │   {"type":"turn.started"}
  │   {"type":"item.started","item":{"id":"item_1","type":"todo_list",...}}
  │   {"type":"item.completed","item":{"id":"item_2","type":"command_execution",...}}
  │   {"type":"item.started","item":{"id":"item_3","type":"file_change",...}}
  │   {"type":"item.completed","item":{"id":"item_3","type":"file_change",...}}
  │   {"type":"item.completed","item":{"id":"item_4","type":"agent_message",...}}
  │   {"type":"turn.completed","usage":{"input_tokens":1200,...}}
  │
  └─ stderr (RUST_LOG 日志，默认仅 error 级别，几乎无输出)
```

### 1.2 不带 --json（人类可读模式）

**完整执行流程写入 stderr**，最终 AI 回复写入 stdout（管道场景用）。所有输出均来自 `EventProcessorWithHumanOutput` 中的 `eprintln!` 调用（`codex-rs/exec/src/event_processor_with_human_output.rs`）。

```
codex exec "实现一个排序函数"
  ├─ stderr（完整执行过程）
  │   OpenAI Codex v2.1.138
  │   ────────
  │   model: claude-sonnet-4-6
  │   ────────
  │
  │   user
  │   实现一个排序函数
  │
  │   exec                                     ← item started
  │   cargo init --lib    in /tmp/project
  │   succeeded:                               ← item completed
  │
  │   apply patch                              ← item started
  │   completed                                ← item completed
  │   src/lib.rs
  │
  │   codex                                    ← agent_message
  │   已实现快速排序...
  │
  │   tokens used  1,200 + 800
  │
  └─ stdout（仅最终 AI 回复文本，管道场景用）
      已实现快速排序...
```

> **重要澄清**：stderr 不只是"日志"。在不带 `--json` 的模式下，事件处理器将所有格式化后的执行过程（turn 开始/完成、命令执行、文件修改、AI 回复等）都写到 stderr。RUST_LOG 开启后会混入同一流，但默认 error 级别几乎无输出。

---

## 二、终端嵌入方案对比

在 Web 管理台的节点详情页中嵌入一个"终端"窗口，实时查看执行过程。以下是两种方案：

### 方案 A：捕获 stderr 直连 xterm.js（简单，快速跑通）

不传 `--json`，直接捕获 stderr，通过 WebSocket 推到前端 xterm.js。

```python
import asyncio

async def codex_to_xterm(prompt: str, ws):
    """方案 A：stderr 直连，完整执行流程已在其中"""
    proc = await asyncio.create_subprocess_exec(
        "codex", "exec", prompt,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    async for line in proc.stderr:
        await ws.send_text(line.decode())

    await proc.wait()
```

```typescript
// 前端：xterm.js 原生支持 ANSI 转义码
import { Terminal } from "@xterm/xterm";
const term = new Terminal({ convertEol: true });
term.open(document.getElementById("terminal"));
ws.onmessage = (e) => term.write(e.data);
```

**优点**：零额外开发，输出效果和本地终端一致。  
**缺点**：
- stderr 上执行过程和 RUST_LOG 混在同一流，调高日志级别会产生噪音
- 输出是纯文本，无法提取结构化数据做 diff 视图、状态卡片
- md/修改的文件/搜索结果等，无法获取数据做额外展示，只能展示执行过程

### 方案 B：--json 事件 → ANSI 格式化 → xterm.js（推荐，最终方案）

用 `--json` 拿到结构化事件，在 Python 层转成带 ANSI 颜色的终端文本，前端仍用 xterm.js 渲染。

```
codex exec --json "prompt"
  → stdout: JSONL 结构化事件
  → Python: 逐行解析 → 转 ANSI 彩色文本
  → WebSocket → 前端 xterm.js 渲染
```

```python
def format_event_for_terminal(event: dict) -> str:
    """将单个 JSONL 事件格式化为带 ANSI 颜色的终端文本"""
    t = event["type"]
    GREEN  = "\033[32m"; CYAN = "\033[36m"; YELLOW = "\033[33m"
    RED    = "\033[31m"; DIM = "\033[2m"; BOLD = "\033[1m"; RESET = "\033[0m"

    if t == "turn.started":
        return f"\n{CYAN}━━━ Turn Started ━━━{RESET}\n"
    elif t == "turn.completed":
        u = event.get("usage", {})
        return (
            f"\n{GREEN}✓ 完成{RESET}"
            f"  {DIM}输入: {u.get('input_tokens', '?')}"
            f" | 输出: {u.get('output_tokens', '?')} tokens{RESET}\n"
        )
    elif t == "turn.failed":
        return f"\n{RED}✗ 执行失败: {event['error']['message']}{RESET}\n"

    elif t in ("item.started", "item.completed"):
        return _format_item(event["item"], completed=(t == "item.completed"))
    return ""

def _format_item(item: dict, completed: bool) -> str:
    detail = item["details"]
    dtype = detail["type"]
    icons = {"command_execution": "⚡", "file_change": "📄",
             "web_search": "🔍", "todo_list": "📋", "mcp_tool_call": "🔌"}
    icon = icons.get(dtype, "•")

    if not completed:
        return f"  {CYAN}{icon} {dtype}{RESET} ...\n"

    if dtype == "command_execution":
        rc = detail.get("exit_code")
        status = f"{GREEN}✓{RESET}" if rc == 0 else f"{RED}✗{RESET}"
        return (
            f"  {status} $ {detail['command']}\n"
            f"{detail.get('aggregated_output', '')}\n"
            f"{DIM}  exit_code={rc}{RESET}\n"
        )
    elif dtype == "agent_message":
        return f"\n{GREEN}{detail['text']}{RESET}\n"
    elif dtype == "file_change":
        lines = []
        for ch in detail.get("changes", []):
            kind = {"add": "+", "delete": "-", "update": "~"}[ch["kind"]]
            lines.append(f"  {kind} {ch['path']}")
        return "\n".join(lines) + "\n"
    return ""
```

**优点**：
- stdout = 数据通道，stderr = 日志通道，完美分离
- 结构化数据可同时用于 diff 视图、状态卡片（第 5 节的组件方案）
- ANSI 颜色完全可控

**缺点**：需要在 Python 侧写格式化逻辑。

### 方案对比总结

| | 方案 A (stderr 直连) | 方案 B (--json + 格式化) |
|---|---|---|
| 开发量 | 极少 | 中等 |
| 数据流分离 | 差（过程和日志混在 stderr） | 好（stdout=数据, stderr=日志） |
| 终端效果 | 原生格式，不可控 | 可控，可加图标/颜色 |
| 数据复用 | 纯文本，无法提取 | 结构保留，可同时做 diff 视图等 |
| 推荐场景 | 快速 debug / 原型验证 | 最终产品 |

建议：先用方案 A 跑通整个链路，后续切换方案 B — 两者前端组件都是 xterm.js，切换不需要改前端。

---

## 三、为什么不能复用 TUI（codex-rs/tui/）

### 3.1 TUI 和 exec 是同一层的两个对等实现

```
                    ┌──────────────────────┐
                    │ InProcessAppServerClient │  ← 进程内 JSON-RPC 桥
                    └──────────┬───────────┘
                               │
               ┌───────────────┴───────────────┐
               │                               │
       ┌───────▼──────┐               ┌───────▼──────┐
       │   codex tui   │               │  codex exec  │
       │  (交互式终端)   │               │  (非交互式)   │
       ├───────────────┤               ├───────────────┤
       │ 渲染: ratatui   │               │ 输出: JSONL    │
       │ 目标: ANSI 终端 │               │ 目标: stdout   │
       │ 交互: 键盘输入   │               │ 交互: 全部拒绝  │
       └───────────────┘               └───────────────┘
```

TUI 和 exec 是**对等的消费者**，都调用同一个 `InProcessAppServerClient`，只是渲染和交互方式不同：
- **TUI**：收到 `ServerNotification` → 用 ratatui 的 `Frame`/`Paragraph`/`Span` 拼成终端界面
- **exec**：收到 `ServerNotification` → 序列化为 JSONL 写到 stdout

TUI 不消费 exec 的 JSONL 输出，exec 也不调用 TUI 的 ratatui 渲染组件。

### 3.2 技术阻隔

| 维度 | TUI (codex-rs/tui/) | Web 管理台需求 | 结论 |
|------|---------------------|---------------|------|
| 渲染库 | ratatui (Rust) | React/Vue (JS/TS) | 无法互操作 |
| 输出目标 | ANSI 终端字节流 | 浏览器 DOM | 完全不同 |
| 交互模式 | 实时键盘事件、审批弹窗 | 只读展示（exec 无交互） | exec 模式下不适用 |
| 耦合度 | `chatwidget.rs` 直接依赖 `ThreadManager` | 只通过 JSONL 消费 | TUI 无法独立运行 |

关键事实：
- `chatwidget.rs` 有 239K 行，深度耦合 `ThreadManager` 的内部状态，没有独立的"渲染器"接口
- ratatui 有 `Paragraph`、`Line`、`Span` 等终端布局原语，没有 `render_to_html()` 方法
- TUI 是交互式的：它处理键盘快捷键、弹审批弹窗、滚动聊天记录 — exec 模式下这些 ServerRequest 全部被自动拒绝

### 3.3 那 xterm.js 和 TUI 是什么关系？

**没有任何关系。** xterm.js 是独立的开源前端终端模拟器库（npm 包 `@xterm/xterm`），VS Code 内置终端用的就是它。

两者只是"目标场景相似"——都在各自的平台上模拟终端体验：

| | codex TUI (ratatui) | xterm.js |
|---|---|---|
| 作者 | Anthropic | 独立开源社区 |
| 语言 | Rust | TypeScript |
| 运行位置 | 用户本地终端程序 | 浏览器 DOM |
| npm 包 | 不适用 | `npm install @xterm/xterm` |
| 用法 | 直接使用 codex 命令 | 嵌入 Web 页面 |

```html
<!-- xterm.js 的最小化使用 -->
<div id="terminal"></div>
<script type="module">
  import { Terminal } from "@xterm/xterm";
  const term = new Terminal({ convertEol: true });
  term.open(document.getElementById("terminal"));
  term.write("Hello from \x1B[32mcodex exec\x1B[0m\n");
</script>
```

### 3.4 如果强行要"用 TUI"怎么办？

唯一的办法是在服务器上跑完整交互版 `codex`，挂到 PTY 伪终端上，再把 PTY 流通过 WebSocket 推到前端 xterm.js。这等同于在浏览器里开一个远程终端窗口。

**但这不适用于 LangGraph DAG 场景**——交互版需要人工插手（审批命令、回答问题），和自动流水线矛盾。exec 之所以存在，就是为了非交互场景。

---

## 四、顶层事件类型

| 事件 | type | 含义 | 关键字段 |
|------|------|------|---------|
| 线程创建 | `thread.started` | 新线程启动，始终为第一个事件 | `thread_id` |
| 轮次开始 | `turn.started` | 用户 prompt 已提交 | — |
| 项目开始 | `item.started` | 一个子任务开始执行 | `item: ThreadItem` |
| 项目更新 | `item.updated` | 子任务状态更新 | `item: ThreadItem` |
| 项目完成 | `item.completed` | 子任务到达终态 | `item: ThreadItem` |
| 轮次完成 | `turn.completed` | 本轮工作结束 | `usage: Usage` |
| 轮次失败 | `turn.failed` | 本轮执行出错 | `error: { message }` |
| 致命错误 | `error` | 流级错误 | `message` |

## 五、项目详情类型 (ThreadItemDetails)

每个 item 通过 `details.type` 区分负载：

```typescript
type ThreadItemDetails =
  | { type: "agent_message"; text: string }              // AI 文本回复
  | { type: "reasoning"; text: string }                   // AI 推理过程
  | { type: "command_execution"; command: string; ... }   // Shell 命令执行
  | { type: "file_change"; changes: FileUpdate[]; ... }   // 文件修改
  | { type: "mcp_tool_call"; server: string; ... }        // MCP 工具调用
  | { type: "collab_tool_call"; tool: CollabTool; ... }   // 协作 agent 调用
  | { type: "web_search"; query: string; ... }            // 网络搜索
  | { type: "todo_list"; items: TodoItem[] }              // 任务清单
  | { type: "error"; message: string }                    // 非致命错误
```

### 各类型的详细字段

**command_execution** — Shell 命令执行
```typescript
{
  type: "command_execution";
  command: string;           // 执行的命令文本
  aggregated_output: string; // 聚合输出 (stdout + stderr)
  exit_code: number | null;  // 退出码
  status: "in_progress" | "completed" | "failed" | "declined";
}
```

**file_change** — 文件修改
```typescript
{
  type: "file_change";
  changes: Array<{
    path: string;                    // 文件路径
    kind: "add" | "delete" | "update"; // 变更类型
  }>;
  status: "in_progress" | "completed" | "failed";
}
```

**mcp_tool_call** — MCP 工具调用
```typescript
{
  type: "mcp_tool_call";
  server: string;            // MCP 服务器名
  tool: string;              // 工具名
  arguments: object;         // 调用参数
  result: { content: object[]; structured_content?: object } | null;
  error: { message: string } | null;
  status: "in_progress" | "completed" | "failed";
}
```

**web_search** — 网络搜索
```typescript
{
  type: "web_search";
  id: string;
  query: string;
  action: "search" | "fetch";  // 搜索或抓取
}
```

**todo_list** — 任务清单（唯一会多次更新的类型）
```typescript
{
  type: "todo_list";
  items: Array<{
    text: string;       // 任务描述
    completed: boolean; // 是否完成
  }>;
}
```

**collab_tool_call** — 协作 agent 调用
```typescript
{
  type: "collab_tool_call";
  tool: "spawn_agent" | "send_input" | "wait" | "close_agent";
  sender_thread_id: string;
  receiver_thread_ids: string[];
  prompt?: string;
  agents_states: Record<string, { status: string; message?: string }>;
  status: "in_progress" | "completed" | "failed";
}
```

---

## 六、LangGraph 节点中的消费方案

### 6.1 批量模式（推荐起步）

节点等待 codex 执行完毕，一次性解析所有事件：

```python
import subprocess
import json
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class CodexNodeResult:
    """codex exec 节点的完整输出"""
    thread_id: str
    events: list[dict]                    # 全部 JSONL 事件
    items: dict[str, dict]                # id → item 最终状态 Map
    last_message: str                     # AI 最终回复文本
    usage: Optional[dict] = None          # token 用量
    exit_code: int = 0
    stderr: str = ""

    @property
    def agent_messages(self) -> list[str]:
        """提取所有 AI 文本回复"""
        return [
            item["details"]["text"]
            for item in self.items.values()
            if item["details"]["type"] == "agent_message"
        ]

    @property
    def file_changes(self) -> list[dict]:
        """提取所有文件变更"""
        result = []
        for item in self.items.values():
            if item["details"]["type"] == "file_change":
                result.extend(item["details"]["changes"])
        return result

    @property
    def command_executions(self) -> list[dict]:
        """提取所有命令执行记录"""
        return [
            item["details"]
            for item in self.items.values()
            if item["details"]["type"] == "command_execution"
        ]


def run_codex_node(prompt: str, debug: bool = False) -> CodexNodeResult:
    """在 LangGraph 节点中调用 codex exec"""
    cmd = [
        "codex", "exec",
        "--json",
        "-o", "/tmp/codex_last_msg.txt",
        prompt,
    ]

    env = os.environ.copy()
    if debug:
        env["RUST_LOG"] = "debug"

    proc = subprocess.run(cmd, capture_output=True, text=True, env=env)

    events = []
    items: dict[str, dict] = {}

    for line in proc.stdout.strip().split("\n"):
        if not line:
            continue
        event = json.loads(line)
        events.append(event)

        # 维护 item 状态 Map
        if event["type"] in ("item.started", "item.updated", "item.completed"):
            item = event["item"]
            items[item["id"]] = item

    # 读取 AI 最终回复
    last_message = ""
    try:
        with open("/tmp/codex_last_msg.txt") as f:
            last_message = f.read()
    except FileNotFoundError:
        pass

    # 提取用量
    usage = None
    for event in events:
        if event["type"] == "turn.completed":
            usage = event.get("usage")

    # 提取线程 ID
    thread_id = events[0]["thread_id"] if events else ""

    return CodexNodeResult(
        thread_id=thread_id,
        events=events,
        items=items,
        last_message=last_message,
        usage=usage,
        exit_code=proc.returncode,
        stderr=proc.stderr,
    )
```

### 6.2 流式模式（实时推送）

通过 SSE 将事件实时推到前端：

```python
import asyncio
import json

async def run_codex_node_streaming(prompt: str, sse_queue):
    """流式执行 codex exec，事件实时入队"""
    proc = await asyncio.create_subprocess_exec(
        "codex", "exec", "--json", prompt,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )

    items: dict[str, dict] = {}

    async for line in proc.stdout:
        event = json.loads(line.decode())
        event_type = event["type"]

        if event_type in ("item.started", "item.updated", "item.completed"):
            item = event["item"]
            items[item["id"]] = item

        # 推送事件到前端
        await sse_queue.put(json.dumps({
            "event": event,
            "items_snapshot": list(items.values()),
        }))

    await proc.wait()
```

---

## 七、前端渲染映射

### 7.1 整体布局

```
┌──────────────────────────────────────────────────────┐
│  Node: "实现排序算法"                        [完成]    │
│  ├─ 模型: claude-sonnet-4-6                          │
│  ├─ 输入 tokens: 1,200 | 输出 tokens: 800            │
│  └─ 耗时: 12.3s                                      │
├──────────────────────────────────────────────────────┤
│  ▼ 任务计划 (todo_list)                               │
│  [✓] 分析需求                                         │
│  [✓] 实现快速排序                                     │
│  [✓] 编写测试                                         │
│  [ ] 运行测试验证                  ← 当前执行中       │
├──────────────────────────────────────────────────────┤
│  ▶ 推理过程 (reasoning)                    [展开]      │
│  ┌──────────────────────────────────┐                │
│  │ 用户需要实现排序算法，我先...       │                │
│  └──────────────────────────────────┘                │
├──────────────────────────────────────────────────────┤
│  $ 命令执行 (command_execution)                      │
│  ┌──────────────────────────────────┐                │
│  │ $ cargo test --lib sort          │                │
│  │ running 3 tests                  │                │
│  │ test test_quick_sort ... ok      │                │
│  │ test test_edge_cases ... ok      │                │
│  │ test test_empty_input ... ok     │                │
│  │                                  │                │
│  │ Exit code: 0                     │                │
│  └──────────────────────────────────┘                │
├──────────────────────────────────────────────────────┤
│  📄 文件修改 (file_change)                            │
│  ┌──────────────────────────────────┐                │
│  │ src/sort.rs         +45 -0  ✨   │                │
│  │ tests/sort_test.rs  +32 -0  ✨   │                │
│  └──────────────────────────────────┘                │
├──────────────────────────────────────────────────────┤
│  💬 AI 回复 (agent_message)                           │
│  ┌──────────────────────────────────┐                │
│  │ 已实现快速排序算法，包含以下文件：   │                │
│  │ ...markdown 内容...              │                │
│  └──────────────────────────────────┘                │
└──────────────────────────────────────────────────────┘
```

### 7.2 组件映射表

| Item Type | 前端组件 | 库建议 |
|-----------|---------|--------|
| `agent_message` | Markdown 渲染器 | `react-markdown` + `react-syntax-highlighter` |
| `reasoning` | 可折叠面板，默认收起 | `<details>` / `Collapse` |
| `command_execution` | 终端模拟器风格 | `@xterm/xterm` 或纯 CSS 黑底绿字 |
| `file_change` | Diff 视图 | `react-diff-viewer` 或自定义色块 |
| `mcp_tool_call` | 工具调用卡片 | 自定义 Card 组件 |
| `web_search` | 搜索结果列表 | 链接卡片 |
| `todo_list` | 复选框进度列表 | `CheckList` / `Progress` |
| `error` | 红色告警条 | `Alert` variant="error" |

### 7.3 从 item 到组件的路由

```typescript
function renderItem(item: ThreadItem): ReactNode {
  const { type } = item.details;

  switch (type) {
    case "agent_message":
      return <AgentMessageCard text={item.details.text} />;
    case "reasoning":
      return <ReasoningPanel text={item.details.text} />;
    case "command_execution":
      return <TerminalBlock
        command={item.details.command}
        output={item.details.aggregated_output}
        exitCode={item.details.exit_code}
        status={item.details.status}
      />;
    case "file_change":
      return <FileChangeList changes={item.details.changes} status={item.details.status} />;
    case "mcp_tool_call":
      return <McpToolCard
        server={item.details.server}
        tool={item.details.tool}
        status={item.details.status}
      />;
    case "web_search":
      return <WebSearchCard query={item.details.query} />;
    case "todo_list":
      return <TodoProgress items={item.details.items} />;
    case "error":
      return <ErrorAlert message={item.details.message} />;
  }
}
```

### 7.4 状态颜色约定

```typescript
const statusColor = {
  in_progress: "#1890ff", // 蓝色 — 执行中
  completed:   "#52c41a", // 绿色 — 成功
  failed:      "#ff4d4f", // 红色 — 失败
  declined:    "#faad14", // 黄色 — 被拒绝
} as const;
```

---

## 八、完整数据流架构

```
LangGraph DAG
  │
  ├─ Node A: codex exec --json "需求分析"
  │   ├─ stdout → JSONL events → parse → items Map
  │   └─ 结果写入 LangGraph State
  │
  ├─ Node B: codex exec --json "编码实现"
  │   ├─ 读取 State 中的上下文
  │   └─ ...
  │
  └─ Node C: codex exec --json "编写测试"
      └─ ...

                    ↓ (每个节点完成后)

Web 管理台 ← SSE/WebSocket ← LangGraph callback → NodeResult
  │
  ├─ DAG 拓扑视图 (展示节点依赖、执行状态)
  └─ 节点详情面板 (展示 items 渲染)
```

### State 中的节点结果结构

```python
from typing import TypedDict

class CodexNodeState(TypedDict):
    """存储在 LangGraph State 中的 codex 节点输出"""
    node_name: str
    thread_id: str
    items: list[dict]               # 所有 item 最终状态
    agent_messages: list[str]       # AI 文本回复
    file_changes: list[dict]        # 文件变更列表
    usage: dict | None              # token 用量
    exit_code: int
    duration_ms: float
```

这样下游节点可以引用上游节点的输出，管理台也能按节点索引查看详情。

---

## 九、TypeScript 类型生成

`exec_events.rs` 中所有结构体都标注了 `#[derive(TS)]`，可在 codex-rs 构建时导出 TypeScript 类型定义，保证前后端类型一致。

```
cargo test -p codex-exec --test ts_export
```

生成的类型文件可直接导入前端项目：

```typescript
import type { ThreadEvent, ThreadItem, ThreadItemDetails } from "@/types/codex-events";
```
