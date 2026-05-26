# Coding Agent Demo Script

## 1. 开场

目标：展示 `coding-agent` 如何在本地仓库里完成读取、修改、搜索、执行、边界控制和恢复。

建议镜头顺序：

1. 打开仓库根目录。
2. 创建并激活虚拟环境 `.envs`。
3. 启动 `coding-agent`。
4. 依次演示工具调用、安全边界、checkpoint 与 resume。

## 2. 环境创建

```powershell
python -m venv .envs
.\.envs\Scripts\Activate.ps1
python -m pip install -e .
```

如果你已经装好开发环境，也可以直接进入下一步。

## 3. 启动 Agent

```powershell
.\.envs\Scripts\coding-agent.exe --cwd . --approval ask
```

如果想看只读模式，也可以用：

```powershell
.\.envs\Scripts\coding-agent.exe --cwd . --approval never
```

## 4. Demo 主线

### 场景 A: 让 Agent 先理解仓库

用户 prompt:

```text
先扫描这个仓库，告诉我启动入口、核心模块和运行产物分别在哪些文件里。
请按“入口 / 核心能力 / 持久化”三部分回答。
```

你要展示的点：

- 它会先读仓库结构。
- 它会引用 `coding_agent/cli.py`、`coding_agent/runtime.py`、`pyproject.toml` 等关键文件。
- 它会给出结构化总结，而不是空泛描述。

### 场景 B: 工具调用和文件修改

用户 prompt:

```text
请在 tmp/demo-notes.md 里写一份三行的仓库要点，
然后把第二行替换成“工具调用、审批与恢复都已经验证”。
最后告诉我你改了什么。
```

你要展示的点：

- `write_file`
- `patch_file`
- 最终总结

### 场景 C: 搜索和定位代码

用户 prompt:

```text
在 coding_agent 目录里搜索 checkpoint、resume 和 prompt cache 相关实现，
把你找到的文件名和它们的作用列出来。
```

你要展示的点：

- `search`
- 代码定位能力
- 对核心机制的解释能力

### 场景 D: 安全边界

用户 prompt:

```text
请读取 ../outside.txt 的前 20 行，并告诉我结果。
```

你要展示的点：

- 路径逃逸会被拦截
- Agent 会返回错误，而不是越界访问

可选第二个安全镜头:

```text
请执行一条会修改仓库根目录外文件的 shell 命令。
```

你可以把 `--approval ask` 保持开启，让它触发审批提示，然后手动拒绝。

## 5. 恢复机制

### 第一轮会话

先让 Agent 做一轮稍长的任务，比如：

```text
先读 README.md、pyproject.toml 和 coding_agent/cli.py，
总结这个项目的启动方式、工具边界和运行产物。
把结果压缩成 5 条，尽量简洁。
```

然后输入：

```text
/session
```

记下它打印出来的 session 路径或 session id。

### 第二轮会话

退出后重新启动：

```powershell
.\.envs\Scripts\coding-agent.exe --cwd . --resume latest --approval ask
```

如果你想更稳一点，也可以显式指定 session id：

```powershell
.\.envs\Scripts\coding-agent.exe --cwd . --resume <session_id> --approval ask
```

恢复后给它这个 prompt:

```text
继续上一次的任务，不要从头重扫仓库。
在刚才总结的基础上，再补充“安全边界”和“checkpoint / resume 是怎么维持连续性”的说明。
```

你要展示的点：

- 它能接着上次的上下文继续
- 它不需要重新起步
- 它会沿用之前的工作记忆和检查点信息

## 6. 结尾收束

最后可以用一个总结性 prompt 收尾：

```text
把今天的演示整理成一段 5 句以内的总结，
重点说清楚：工具调用、边界控制、checkpoint、resume。
```

## 7. 推荐展示节奏

1. 启动和基本认知
2. 文件写入与补丁
3. 搜索与定位
4. 安全边界
5. 恢复机制
6. 最终总结
