# LangGraph + codex exec 节点设计

## 一、总体方案

两个核心问题用 LangGraph 内置机制解决：

| 问题 | 方案 | 依赖 |
|------|------|------|
| 服务重启 | `SqliteSaver` checkpointer，状态持久化到 SQLite | LangGraph 内置 |
| 节点重试 | 失败节点标注 + `update_state` 重置 + 重新 `invoke` | LangGraph 内置 |

不需要自己写状态持久化逻辑，LangGraph 的 checkpointer 已经做好了。

---

## 二、State 定义

```python
from typing import TypedDict, Annotated, Optional
from langgraph.graph.message import add_messages
import operator


class NodeResult(TypedDict):
    """单个 codex 节点的执行结果"""
    node_name: str
    status: str              # pending | running | completed | failed | interrupted
    thread_id: str           # codex 的 thread_id，用于 resume
    items: list[dict]        # 所有 item 最终状态
    last_message: str        # AI 最终回复
    file_changes: list[dict]
    exit_code: int
    duration_ms: float
    error: str               # 失败时的错误信息
    retry_count: int         # 重试次数


class PipelineState(TypedDict):
    """整个 DAG 流水线的状态"""
    # 持久化，由 checkpointer 管理
    prompt: str                          # 用户原始需求
    nodes: Annotated[dict[str, NodeResult], operator.or_]  # node_name → 结果，用 or_ 合并

    # 仅用于节点间通信，不持久化
    messages: Annotated[list, add_messages]
```

---

## 三、codex 节点装饰器

```python
import asyncio
import json
import time
import signal
import os
from pathlib import Path


DATA_DIR = Path("/tmp/codex-pipeline")  # JSONL 落盘目录
DATA_DIR.mkdir(exist_ok=True)


async def codex_node(
    node_name: str,
    prompt: str,
    state: PipelineState,
) -> dict:
    """包装 codex exec 的 LangGraph 节点，支持中断和失败处理"""

    node_start = time.time()
    jsonl_path = DATA_DIR / f"{node_name}_{int(time.time())}.jsonl"
    proc = None

    # 准备上下文：把上游节点的关键信息注入 prompt
    ctx = _build_context(state)
    full_prompt = ctx + prompt if ctx else prompt

    try:
        # 写入 JSONL 文件
        with open(jsonl_path, "w") as out:
            proc = await asyncio.create_subprocess_exec(
                "codex", "exec", "--json", full_prompt,
                stdout=out,
                stderr=asyncio.subprocess.DEVNULL,
            )
            exit_code = await proc.wait()

        # 解析 JSONL
        items = _parse_jsonl(jsonl_path)
        messages = [v["details"]["text"] for v in items.values()
                    if v["details"]["type"] == "agent_message"]
        changes = []
        for v in items.values():
            if v["details"]["type"] == "file_change":
                changes.extend(v["details"].get("changes", []))

        thread_id = ""
        for item in items.values():
            if item["id"].startswith("thread"):
                thread_id = item["details"].get("thread_id", "")

        return {"nodes": {node_name: {
            "node_name": node_name,
            "status": "completed" if exit_code == 0 else "failed",
            "thread_id": thread_id,
            "items": list(items.values()),
            "last_message": messages[-1] if messages else "",
            "file_changes": changes,
            "exit_code": exit_code,
            "duration_ms": (time.time() - node_start) * 1000,
            "error": "" if exit_code == 0 else f"exit_code={exit_code}",
            "retry_count": state["nodes"].get(node_name, {}).get("retry_count", 0),
        }}}

    except asyncio.CancelledError:
        # 用户中断
        if proc and proc.returncode is None:
            proc.send_signal(signal.SIGTERM)
            try:
                await asyncio.wait_for(proc.wait(), timeout=5)
            except asyncio.TimeoutError:
                proc.kill()

        return {"nodes": {node_name: {
            "node_name": node_name,
            "status": "interrupted",
            "thread_id": "",
            "items": _parse_jsonl_safe(jsonl_path),
            "last_message": "",
            "file_changes": [],
            "exit_code": -1,
            "duration_ms": (time.time() - node_start) * 1000,
            "error": "user interrupted",
            "retry_count": state["nodes"].get(node_name, {}).get("retry_count", 0),
        }}}

    except Exception as e:
        return {"nodes": {node_name: {
            "node_name": node_name,
            "status": "failed",
            "thread_id": "",
            "items": _parse_jsonl_safe(jsonl_path),
            "last_message": "",
            "file_changes": [],
            "exit_code": -1,
            "duration_ms": (time.time() - node_start) * 1000,
            "error": str(e),
            "retry_count": state["nodes"].get(node_name, {}).get("retry_count", 0),
        }}}


def _build_context(state: PipelineState) -> str:
    """把上游节点的输出拼接成上下文，注入下游节点的 prompt"""
    parts = []
    for name, result in state.get("nodes", {}).items():
        if result["status"] == "completed":
            parts.append(f"[上游节点 '{name}' 的输出]\n{result['last_message']}")
    return "\n\n".join(parts)


def _parse_jsonl(path: Path) -> dict[str, dict]:
    items = {}
    if path.exists():
        with open(path) as f:
            for line in f:
                event = json.loads(line)
                if event["type"] in ("item.started", "item.updated", "item.completed"):
                    items[event["item"]["id"]] = event["item"]
    return items


def _parse_jsonl_safe(path: Path) -> list[dict]:
    """出错时尽量挽救已输出的 JSONL"""
    try:
        return list(_parse_jsonl(path).values())
    except Exception:
        return []
```

---

## 四、DAG 定义（以三节点流水线为例）

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3


# === DAG 定义 ===

builder = StateGraph(PipelineState)

builder.add_node("analyze", lambda s: codex_node("analyze", "分析需求: " + s["prompt"], s))
builder.add_node("implement", lambda s: codex_node("implement", "实现代码", s))
builder.add_node("test", lambda s: codex_node("test", "编写并运行测试", s))

builder.set_entry_point("analyze")
builder.add_edge("analyze", "implement")

# 条件边：implement 成功 → test，失败 → END
def after_implement(state: PipelineState) -> str:
    node = state["nodes"].get("implement", {})
    if node.get("status") == "failed":
        return END  # 等用户手动重试
    return "test"

builder.add_conditional_edges("implement", after_implement, {"test": "test", END: END})
builder.add_edge("test", END)


# === 持久化 checkpointer（服务重启不丢状态）===

DB_PATH = DATA_DIR / "pipeline.db"
conn = sqlite3.connect(str(DB_PATH), check_same_thread=False)
checkpointer = SqliteSaver(conn)
app = builder.compile(checkpointer=checkpointer)
```

---

## 五、运行、中断、重试

### 5.1 启动运行

```python
import asyncio

thread_id = "pipeline-run-001"  # 全局唯一，重启后靠它恢复
config = {"configurable": {"thread_id": thread_id}}

initial_state: PipelineState = {
    "prompt": "实现一个 HTTP 客户端库，支持重试和超时",
    "nodes": {},
}

async def run():
    result = await app.ainvoke(initial_state, config)
    return result
```

### 5.2 中断

```python
# 方案1：LangGraph 原生 interrupt — 在节点执行前/后设断点
# 对 codex exec 这种长进程不太实用，因为它在 await proc.wait() 期间无法响应

# 方案2：asyncio task cancel — 实用
task = asyncio.create_task(run())

# 用户点停止
task.cancel()
try:
    await task
except asyncio.CancelledError:
    print("流水线已停止，状态已通过 checkpointer 持久化")
```

### 5.3 失败节点重试

```python
async def retry_node(thread_id: str, node_name: str):
    """用户触发重试某个失败节点"""

    # 1. 读取当前 state
    current = app.get_state({"configurable": {"thread_id": thread_id}})
    state = current.values.copy()

    # 2. 重置目标节点状态，retry_count +1
    old = state["nodes"].get(node_name, {})
    state["nodes"][node_name] = {
        **old,
        "status": "pending",
        "retry_count": old.get("retry_count", 0) + 1,
    }

    # 3. 更新 state 并从中断点继续
    app.update_state({"configurable": {"thread_id": thread_id}}, state)
    result = await app.ainvoke(None, {"configurable": {"thread_id": thread_id}})
    return result
```

### 5.4 服务重启后恢复

```python
def resume_after_restart(thread_id: str):
    """服务重启后，复用同一个 SqliteSaver 和 thread_id"""

    # 1. 重新打开 SQLite
    conn = sqlite3.connect(str(DB_PATH), check_same_thread=False)
    checkpointer = SqliteSaver(conn)

    # 2. 重建 graph（必须和之前的结构完全一致）
    builder = StateGraph(PipelineState)
    # ... 重新 add_node ...
    app = builder.compile(checkpointer=checkpointer)

    # 3. 获取上次的状态
    state_snapshot = app.get_state({"configurable": {"thread_id": thread_id}})

    if state_snapshot.next:
        # 还有未执行的节点 → 继续
        return app.ainvoke(None, {"configurable": {"thread_id": thread_id}})
    else:
        # 已经全部完成或者已失败
        return state_snapshot.values
```

---

## 六、Web 管理台 API

```python
from fastapi import FastAPI, WebSocket
from pydantic import BaseModel

api = FastAPI()


# === REST API ===

class StartRequest(BaseModel):
    prompt: str
    thread_id: str


@api.post("/pipeline/start")
async def start_pipeline(req: StartRequest):
    initial_state: PipelineState = {"prompt": req.prompt, "nodes": {}}
    task = asyncio.create_task(
        app.ainvoke(initial_state, {"configurable": {"thread_id": req.thread_id}})
    )
    return {"thread_id": req.thread_id, "status": "started"}


@api.post("/pipeline/{thread_id}/stop")
async def stop_pipeline(thread_id: str):
    # 从全局 task map 里找到 task 并 cancel
    task = _task_registry.get(thread_id)
    if task:
        task.cancel()
    return {"thread_id": thread_id, "status": "stopped"}


@api.post("/pipeline/{thread_id}/retry/{node_name}")
async def retry_node_api(thread_id: str, node_name: str):
    result = await retry_node(thread_id, node_name)
    return {"thread_id": thread_id, "node": node_name, "result": result}


@api.get("/pipeline/{thread_id}/state")
async def get_state(thread_id: str):
    snapshot = app.get_state({"configurable": {"thread_id": thread_id}})
    return snapshot.values if snapshot else None


# === WebSocket：推送节点执行过程到前端 xterm.js ===

@api.websocket("/pipeline/{thread_id}/nodes/{node_name}/stream")
async def stream_node_output(ws: WebSocket, thread_id: str, node_name: str):
    await ws.accept()

    # 找到该节点的 JSONL 文件，逐行推送 ANSI 文本
    state = app.get_state({"configurable": {"thread_id": thread_id}})
    if state:
        items = state.values.get("nodes", {}).get(node_name, {}).get("items", [])
        for item in items:
            ansi_text = format_event_for_terminal(item)
            if ansi_text:
                await ws.send_text(ansi_text)
```


## 七、中断/重试行为表

| 场景 | 行为 | 数据是否安全 |
|------|------|-------------|
| 用户在 A 节点执行中点停止 | 当前节点标记 `interrupted`，已执行节点保留 | ✅ JSONL + State 均已落盘 |
| 用户在 B 节点成功、C 节点未启动时停止 | B 完成，C 保持 pending | ✅ SqliteSaver 持久化 |
| 服务在 C 节点执行中崩溃 | C 标记 `failed`（进程被 OS 杀掉） | ✅ checkpoint 在崩溃前已写 |
| 用户重试失败的 B 节点 | B 的 retry_count+1，codex exec 全新执行 | ✅ JSONL 覆盖为新文件 |
| 服务重启后恢复 | 重建 graph + SqliteSaver，读到 checkpoint 继续 | ✅ 本地 SQLite |
| 管理员查看历史 run | `get_state` 返回完整 State | ✅ JSONL 文件在磁盘上 |
