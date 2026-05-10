# codex exec 在 LangGraph DAG 中的使用指南

## 一、基础设施

```
/data
├── repos/{project}/           ← -C 指向的工作目录
├── knowledge/                  ← --add-dir，业务知识库（人工维护）
│   ├── domain-model.md         领域模型、业务概念
│   ├── architecture.md         架构决策、技术选型
│   └── coding-conventions.md   编码规范
├── codebase-index/             ← --add-dir，代码库索引（定时任务自动生成）
│   └── {project}-index.md
├── rules/                      ← 执行规则（符号链接到工作目录或全局 CODEX_HOME）
│   └── {project}.rules
└── output/                     ← --add-dir，DAG 产出汇总
    ├── tech-design.md
    └── ...
```

- **knowledge**：人工维护，长期沉淀，讲"业务是什么"、"为什么这么设计"
- **codebase-index**：定时任务自动生成，讲"代码在哪"、"模块间怎么依赖"
- **rules**：`.rules` 文件放在工作目录下，codex 自动递归加载，不需要 CLI 传参
- **output**：DAG 节点产出文件，下游节点通过 `--add-dir` 读取

## 二、定时任务：代码库索引生成

作为一个独立的全局定时任务，不是 DAG 节点。每次代码更新后刷新。

```bash
# crontab: 每天凌晨 2 点
0 2 * * * /usr/local/bin/codex exec \
  --json \
  --dangerously-bypass-approvals-and-sandbox \
  -C /data/repos/my-project \
  --add-dir /data/knowledge \
  "分析当前项目的完整代码结构（目录、模块、依赖关系、关键入口文件），
   参考 /data/knowledge/ 理解业务背景，
   输出一份给 AI 阅读的代码导航文档到 /data/codebase-index/my-project-index.md。
   需要按模块列出：职责、关键文件路径、对外接口、依赖的模块。"
```

## 三、单个 DAG 节点的标准调用

```bash
codex exec \
  --json \
  --dangerously-bypass-approvals-and-sandbox \
  -C /data/repos/my-project \
  --add-dir /data/knowledge \
  --add-dir /data/codebase-index \
  --add-dir /data/output \
  --output-last-message /data/output/last-message.txt \
  "prompt"
```

参数固定，只有 prompt 因节点不同而变化。以下是每个参数的说明。

## 四、参数速查

### 核心

| 参数 | 说明 | 必选 |
|------|------|:----:|
| `[PROMPT]` | AI 指令，写 `-` 则从 stdin 读 | ✅ |
| `--json` | 输出 JSONL 到 stdout | ✅ |

### 上下文注入

| 参数 | 说明 |
|------|------|
| `-C, --cd <DIR>` | 工作目录。codex 在此目录内读取/写入文件，同时也是 rules 的加载根目录 |
| `-i, --image <FILE>` | 附加图片到 prompt，可多次指定（逗号分隔或重复 `-i`） |
| `--add-dir <DIR>` | 额外可写目录，可多次指定。用于挂载 knowledge、codebase-index、output 等 |
| `-p, --profile <NAME>` | 使用 `config.toml` 中指定 profile，按节点类型切换 skills 配置 |

### 模型

| 参数 | 说明 |
|------|------|
| `-m, --model <MODEL>` | 指定模型 |
| `--oss` | 使用开源供应商 |
| `--local-provider <NAME>` | `lmstudio` 或 `ollama` |

### 输出控制

| 参数 | 说明 |
|------|------|
| `-o, --output-last-message <FILE>` | AI 最终回复写入文件 |
| `--output-schema <FILE>` | JSON Schema，约束 AI 输出为结构化 JSON |
| `--color <always\|never\|auto>` | ANSI 颜色开关，默认 auto（管道下自动关闭） |

### 沙箱

| 参数 | 说明 |
|------|------|
| `-s, --sandbox <MODE>` | `read-only` / `workspace-write` / `danger-full-access` |
| `--dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`） | 跳过所有审批和沙箱 |

> DAG 自动流水线需要 `--dangerously-bypass-approvals-and-sandbox`，否则 exec 因为在非交互模式下无法获得审批而中断。

### 环境

| 参数 | 说明 |
|------|------|
| `--skip-git-repo-check` | 允许在非 git 目录运行 |
| `--ephemeral` | 不持久化 session 到磁盘 |
| `--ignore-user-config` | 不加载 `~/.codex/config.toml` |
| `--ignore-rules` | 禁止加载 `.rules` 文件 |

## 五、Rules

Rules 是 execpolicy DSL 格式的执行策略文件，控制 codex 可以执行哪些命令、不能执行哪些。

**加载方式**：codex 从工作目录 `-C` 向上递归查找 `.rules` 文件，自动加载。不需 CLI 传参。`$CODEX_HOME/.rules` 全局生效。

```
# /data/repos/my-project/.rules
allow: cargo build, cargo test, cargo fmt
allow: git status, git diff, git add, git commit
deny: rm -rf
deny: git push --force
```

## 六、Knowledge

**加载方式**：通过 `--add-dir` 挂载。AI 在 turn 中自己 `cat` / `rg` 查阅。

**内容**：
- `domain-model.md` — 业务领域概念、实体定义、业务流程
- `architecture.md` — 系统架构、技术选型理由、关键决策记录
- `coding-conventions.md` — 命名规范、目录结构约定、代码风格

## 七、Skills

**加载方式**：通过 `config.toml` + 文件系统，不是 CLI 参数。

```
$CODEX_HOME/skills/              ← 全局
$PROJECT/.codex/skills/          ← 项目级，随代码库一起
```

按节点类型切换 profile，每个 profile 可配置不同的 skills 组合：

```toml
# ~/.codex/config.toml
[profile.design]
# 设计类节点，加载架构评审、API 设计等 skills

[profile.implement]
# 编码类节点，加载代码生成、重构等 skills

[profile.review]
# 审查类节点
```

调用时按需切换：

```bash
codex exec --profile design "设计订单模块 API"     # 设计节点
codex exec --profile implement "实现订单模块 API"  # 编码节点
```

## 八、不同节点类型的调用差异

```bash
# 需求分析/文档编写：读为主，不写代码
codex exec --json --dangerously-bypass-approvals-and-sandbox \
  -C /data/repos/my-project \
  --add-dir /data/knowledge \
  --add-dir /data/codebase-index \
  -o /data/output/last-message.txt \
  --profile design \
  "分析需求，编写技术方案文档"

# 编码实现：写代码，-C 指到真实代码库
codex exec --json --dangerously-bypass-approvals-and-sandbox \
  -C /data/repos/my-project \
  --add-dir /data/knowledge \
  --add-dir /data/output \
  --profile implement \
  "实现代码"

# 测试验证：跑真实测试框架
codex exec --json --dangerously-bypass-approvals-and-sandbox \
  -C /data/repos/my-project \
  --profile implement \
  "运行测试套件，修复失败的测试"
```

## 九、环境变量

| 变量 | 说明 |
|------|------|
| `RUST_LOG` | 控制日志级别，默认 `error`。debug 时设 `RUST_LOG=debug` |
| `TRACEPARENT` | W3C trace context，挂到外部 trace 系统 |
| `CODEX_HOME` | codex 配置/状态/缓存的根目录 |

## 十、与其他文档的关系

| 文档 | 内容 |
|------|------|
| `codex-exec-execution-path.md` | 从 CLI 到核心引擎的完整执行路径 + 调试方法 |
| `codex-exec-web-console-integration.md` | JSONL 消费 + xterm.js 渲染方案 |
| `langgraph-codex-node-design.md` | DAG 节点状态定义、中断/重试实现 |
| 本文档 | 每个节点的实际调用命令、基础设施布局、配置方法 |
