# Coding Agent

Coding Agent 是一个面向本地代码仓库的智能协作运行时。它把大模型、受控工具、工作区快照、上下文压缩、会话恢复和运行审计组合成一条完整的命令行工作流，让开发者可以在终端里发起代码分析、文件修改、测试排查和仓库级任务。

项目强调“可控”和“可追踪”：模型不能直接接触文件系统或 shell，而是通过显式注册的工具完成读取、搜索、写入、补丁和命令执行；每次请求都会生成独立运行记录，便于复盘工具调用、停止原因、上下文预算、检查点和最终报告。

## 项目简介

Coding Agent 适合用作轻量级仓库智能执行器原型、模型工具调用实验平台，以及代码仓库自动化任务的研究基座。

核心能力包括：

- 仓库感知：启动时采集当前工作区、Git 信息、项目文档和基础状态。
- 多后端模型接入：支持 Ollama、OpenAI-compatible Responses API、Anthropic-compatible Messages API。
- 受控工具执行：提供文件列表、文件读取、全文搜索、shell 执行、文件写入、精确补丁和只读委派。
- 持久会话：会话历史、工作记忆和检查点保存在本地 `.coding-agent/` 目录。
- 上下文管理：按预算组装 prompt，并对历史、记忆和相关笔记进行压缩。
- 运行审计：每次任务落盘 `task_state.json`、`trace.jsonl`、`report.json`。
- 安全边界：路径锁定在 workspace 内，敏感环境变量会被识别并在 trace/report 中脱敏。
- 实验评估：内置 benchmark、resume metrics、ablation 和 provider experiment 脚本。

## 系统架构
<img width="2730" height="1536" alt="harness_architecture" src="https://github.com/user-attachments/assets/0834bf03-4698-48b6-a234-50ade52df6b3" />


运行链路：

1. `cli.py` 解析命令行参数，选择模型后端，构建 `WorkspaceContext`、`SessionStore` 和 `CodingAgent` 实例。
2. `WorkspaceContext` 收集仓库路径、Git 分支、状态、近期提交和关键文档片段，形成稳定前缀上下文。
3. `CodingAgent.ask()` 创建一次任务运行，并通过 `ContextManager` 拼装本轮 prompt。
4. 模型返回 `<tool>...</tool>` 或 `<final>...</final>`；runtime 负责解析、校验和调度。
5. 工具调用先经过参数校验、路径边界检查、审批策略和重复调用保护，再执行真实动作。
6. 每轮工具结果写入会话历史，关键过程写入 trace，任务状态持续写入 `task_state.json`。
7. 任务结束后生成最终 `report.json`，并更新会话记忆与检查点。

## 运行方式

### 环境要求

- Python 3.10+
- 可选：`uv`
- 可选：Ollama 本地服务
- 可选：OpenAI-compatible 或 Anthropic-compatible API Key

### 安装依赖

使用 `uv`：

```bash
uv sync
```

使用 pip editable 安装：

```bash
pip install -e .
```

### 查看帮助

```bash
uv run coding-agent --help
```

也可以通过模块方式启动：

```bash
python -m coding_agent --help
```

### 交互模式

```bash
uv run coding-agent
```

指定工作目录：

```bash
uv run coding-agent --cwd /path/to/repository
```

### 一次性任务模式

```bash
uv run coding-agent "inspect the failing tests and suggest a fix"
```

### 选择模型后端

Ollama：

```bash
ollama serve
ollama pull qwen3.5:4b
uv run coding-agent --provider ollama --model qwen3.5:4b
```

OpenAI-compatible：

```bash
export OPENAI_API_BASE="https://your-api.example/v1"
export OPENAI_API_KEY="your-api-key"
export OPENAI_MODEL="gpt-5.4"
uv run coding-agent --provider openai
```

Anthropic-compatible：

```bash
export ANTHROPIC_API_BASE="https://your-api.example/v1"
export ANTHROPIC_API_KEY="your-api-key"
export ANTHROPIC_MODEL="claude-sonnet-4-6"
uv run coding-agent --provider anthropic
```

Windows PowerShell 示例：

```powershell
$env:OPENAI_API_BASE = "https://your-api.example/v1"
$env:OPENAI_API_KEY = "your-api-key"
$env:OPENAI_MODEL = "gpt-5.4"
uv run coding-agent --provider openai
```

### 会话恢复

恢复指定 session：

```bash
uv run coding-agent --resume 20260416-110300-a1b2c3
```

恢复最近一次 session：

```bash
uv run coding-agent --resume latest
```

### 审批策略

涉及 shell、写文件、打补丁等高风险工具时，可通过 `--approval` 控制行为：

```bash
uv run coding-agent --approval ask
uv run coding-agent --approval auto
uv run coding-agent --approval never
```

- `ask`：执行风险工具前询问确认。
- `auto`：自动允许风险工具。
- `never`：拒绝风险工具，适合只读分析。

### 常用 REPL 命令

- `/help`：查看内置命令。
- `/memory`：查看当前工作记忆。
- `/session`：查看当前 session 文件路径。
- `/reset`：清空当前会话历史和记忆。
- `/exit` 或 `/quit`：退出交互模式。

### 测试与检查

```bash
uv run pytest
```

```bash
uv run ruff check .
```

## 项目结构详解

```text
.
├── coding_agent/                         # 核心 Python 包
│   ├── __init__.py                # 公共 API 导出
│   ├── __main__.py                # python -m coding_agent 入口
│   ├── cli.py                     # CLI 参数、模型选择、Agent 装配、REPL
│   ├── runtime.py                 # CodingAgent 控制循环、工具调度、会话恢复、审计事件
│   ├── models.py                  # Ollama / OpenAI-compatible / Anthropic-compatible 客户端
│   ├── tools.py                   # 工具注册、参数校验、文件与 shell 工具实现
│   ├── workspace.py               # 工作区快照、Git 信息、文本裁剪与路径展示
│   ├── context_manager.py         # prompt 拼装、上下文预算、历史压缩
│   ├── memory.py                  # 分层记忆、文件摘要、持久笔记检索
│   ├── task_state.py              # 单次 ask() 的状态机与停止原因
│   ├── run_store.py               # 运行产物目录、trace、report、原子写 JSON
│   ├── evaluator.py               # benchmark 加载、固定任务执行与结果汇总
│   └── metrics.py                 # resume、memory、context、安全与 provider 指标实验
├── tests/                         # 单元测试与行为回归测试
│   ├── fixtures/                  # benchmark 测试夹具
│   ├── test_coding_agent.py               # runtime、模型适配、恢复、审计主测试集
│   ├── test_safety_invariants.py  # 路径逃逸、审批、脱敏、只读委派等安全测试
│   ├── test_context_manager.py    # prompt 预算与上下文压缩测试
│   ├── test_memory.py             # 工作记忆和持久记忆测试
│   ├── test_evaluator.py          # benchmark harness 测试
│   ├── test_metrics.py            # 指标实验产物测试
│   ├── test_run_store.py          # 运行产物写入测试
│   └── test_task_state.py         # 任务状态机测试
├── benchmarks/
│   └── coding_tasks.json          # 固定编码任务 benchmark 数据
├── scripts/
│   ├── collect_resume_metrics.py  # 收集恢复能力指标
│   ├── run_large_scale_experiments.py
│   └── run_provider_experiments.py
├── assets/
│   └── screenshots/               # CLI 截图素材
├── pyproject.toml                 # 包元数据、入口点、dev 依赖
├── uv.lock                        # uv 锁文件
└── README.md
```

### 核心模块说明

- `coding_agent.cli`：面向用户的入口层，负责解析 `--provider`、`--model`、`--cwd`、`--resume`、`--approval` 等参数。
- `coding_agent.runtime`：系统中枢，维护模型调用循环、工具执行生命周期、检查点、运行报告、敏感信息脱敏和任务停止逻辑。
- `coding_agent.models`：模型适配层，将不同 provider 的 HTTP API 统一为 `complete(prompt, max_new_tokens, ...)`。
- `coding_agent.tools`：能力白名单，所有外部动作都在这里注册、校验和执行。
- `coding_agent.context_manager`：上下文预算控制器，按 prefix、memory、relevant memory、history、current request 的顺序组装 prompt。
- `coding_agent.memory`：记忆系统，支持任务摘要、近期文件、文件摘要、过程笔记和 durable notes。
- `coding_agent.run_store`：审计落盘层，把任务状态、事件流和报告稳定写入 `.coding-agent/runs/<run_id>/`。
- `coding_agent.evaluator` 与 `coding_agent.metrics`：评估实验层，服务 benchmark、恢复能力、上下文压缩、安全约束和 provider 对比。

## 技术栈全览

| 分类 | 技术 / 模块 | 说明 |
| --- | --- | --- |
| 语言 | Python 3.10+ | 使用标准库实现核心能力，保持轻量 |
| 包管理 | uv / pip | `uv.lock` 固定环境，`pip install -e .` 支持开发安装 |
| CLI | argparse | 命令行参数解析与帮助信息生成 |
| 模型后端 | Ollama | 本地模型调用，走 `/api/generate` |
| 模型后端 | OpenAI-compatible Responses API | 支持 `/responses`、普通 JSON、SSE 文本抽取和 prompt cache 元数据 |
| 模型后端 | Anthropic-compatible Messages API | 支持 `/messages` 协议 |
| HTTP | urllib.request | 无第三方运行时依赖的 API 调用实现 |
| 工具系统 | list/read/search/run/write/patch/delegate | 显式白名单工具，按风险等级进入审批流程 |
| 搜索 | ripgrep 优先，Python fallback | `rg` 可用时优先使用，缺失时回退到内置遍历 |
| 持久化 | JSON / JSONL | session、task_state、trace、report 均为本地文本产物 |
| 记忆 | LayeredMemory / DurableMemoryStore | 工作记忆、相关记忆检索、文件 freshness 校验 |
| 测试 | pytest | 覆盖 runtime、工具、安全、上下文、记忆、评估与指标 |
| 代码质量 | Ruff | 可选静态检查 |

## 运行产物

Coding Agent 会在目标仓库下创建 `.coding-agent/`：

```text
.coding-agent/
├── sessions/
│   └── <session_id>.json
└── runs/
    └── <run_id>/
        ├── task_state.json
        ├── trace.jsonl
        └── report.json
```

- `sessions` 保存可恢复的会话历史、记忆和检查点。
- `task_state.json` 描述当前任务状态、工具步数、停止原因和最终答案。
- `trace.jsonl` 记录模型调用、工具调用、检查点、上下文预算等事件。
- `report.json` 汇总最终结果、prompt 元数据、记忆提升和脱敏环境信息。

这些产物默认只用于本地调试和复盘，通常不需要提交到业务仓库。
