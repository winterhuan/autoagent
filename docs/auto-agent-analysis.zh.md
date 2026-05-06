# AutoAgent 深入分析

本文面向需要接手、启动或继续改造本仓库 AutoAgent 的工程师，基于当前仓库源码梳理设计思想、总体架构、业务逻辑、核心模块、运行链路、启动方案与常见风险。

## 1. 结论速览

- AutoAgent 不是单一脚本，而是一套“可被自动改造的 agent harness + Harbor 评测适配层 + 外部 meta-agent 实验流程”。
- 当前主实现是 `agent.py`：Harbor 通过 `--agent-import-path agent:AutoAgent` 加载 `AutoAgent`，再由它调用 OpenAI Agents SDK 运行模型，并通过 `run_shell` 工具把 shell 命令代理到 Harbor 任务容器。
- `program.md` 定义外部 meta-agent 的工作方法：先跑未修改基线，再按分数驱动迭代修改 `agent.py` 的可编辑区，保留更好或更简单的方案。
- `Dockerfile.base` 构建任务容器基础镜像；真正的 benchmark 任务目录 `tasks/` 在当前基线仓库中不存在，需要按 Harbor 格式另行添加。
- `agent-claude.py` 是一份 Claude SDK 风格的备选/历史 harness；由于文件名包含连字符，不能直接作为常规 Python import path 使用，除非重命名、复制为 `agent.py` 或调整加载方式。

## 2. 仓库阅读地图

| 文件/目录 | 角色 | 关键内容 |
| --- | --- | --- |
| `agent.py` | 当前主 agent harness | OpenAI Agents SDK agent 配置、`run_shell` 工具、Harbor `AutoAgent` 适配器、ATIF 轨迹序列化。 |
| `program.md` | 外部 meta-agent 操作手册 | 指令、可改区域、实验循环、保留/丢弃规则、结果记录格式。 |
| `README.md` | 项目总览与快速启动 | 项目定位、启动命令、任务格式、设计选择、清理命令。 |
| `Dockerfile.base` | 任务容器基础镜像 | Python 3.12 slim + `uv`，安装 `pyproject.toml` 依赖，复制 `agent.py`。 |
| `pyproject.toml` | Python 项目元数据 | 包名 `autoagent`、版本 `0.1.0`、Python `>=3.12`、运行依赖。 |
| `agent-claude.py` | Claude SDK 备选实现 | Claude Code 预设工具、容器内执行入口、ATIF v1.2 序列化。 |
| `.dockerignore` | Docker 构建上下文控制 | 排除 `.venv`、`.env`、`docs/`、`tasks/`、`jobs/`、`uv.lock` 等。 |
| `.gitignore` | Git 工作区忽略规则 | 忽略虚拟环境、密钥、运行产物与任务数据目录。 |
| `progress.png` | README 展示图片 | 项目 teaser 图。 |

当前仓库没有 `tasks/`、`.agent/`、`jobs/`、`results.tsv`、`run.log`。这意味着“运行完整实验循环”前必须先准备 benchmark 任务，或者切换到包含任务数据的分支。

## 3. 业务目标与边界

### 3.1 核心业务目标

AutoAgent 的目标是把“agent harness 工程”做成可自动迭代的实验系统：

1. 人类编写或更新 `program.md`，描述要构建什么类型的 agent、怎么评测、什么变化可以保留。
2. 外部 meta-agent 读取 `program.md`、`README.md`、`agent.py` 和任务样本。
3. meta-agent 先运行未修改基线，得到分数和失败轨迹。
4. meta-agent 修改 `agent.py` 的可编辑 harness 区域，例如 prompt、工具、编排或子 agent。
5. Harbor 运行任务集，生成 task 级分数、日志和 ATIF 轨迹。
6. meta-agent 根据 `program.md` 的 keep/discard 规则决定保留或回滚。
7. 每次实验记录到 `results.tsv`，形成可审计的优化历史。

这个业务流程强调“评测驱动”和“简单性优先”：只有通过分数或等分时更简单的结构证明价值，改动才应保留。

### 3.2 三类 agent/执行者

| 执行者 | 是否在仓库代码中直接实现 | 责任 |
| --- | --- | --- |
| 人类工程师 | 否 | 设定目标、准备任务、提供密钥、审查最终方案。 |
| 外部 meta-agent | 否，主要由调用方提供 | 按 `program.md` 读代码、跑基线、改 `agent.py`、评估、记录结果。 |
| AutoAgent 被测 agent | 是，位于 `agent.py` | 在 Harbor 任务环境中执行自然语言任务，产出任务要求的文件或状态。 |

因此，仓库内的 `agent.py` 是“被优化对象”，而不是完整的自动优化调度器；自动优化循环由外部编码 agent 依据 `program.md` 驱动。

### 3.3 固定边界与可改边界

`agent.py` 把代码分成两层：

- 可编辑 harness：`SYSTEM_PROMPT`、`MODEL`、`MAX_TURNS`、`create_tools()`、`create_agent()`、`run_task()`。
- 固定适配边界：`to_atif()` 和 `AutoAgent` 的 Harbor 集成逻辑。

`program.md` 明确要求普通实验只改固定边界上方，除非人类明确要求，否则不要改 Harbor 适配和轨迹序列化部分。

## 4. 设计思想

### 4.1 Program the meta-agent, not the harness directly

项目希望工程师主要修改 `program.md` 来“编程”外部 meta-agent，而不是每次都手写 harness 改动。这样做的好处是：

- 实验目标、评测策略、保留规则集中在 Markdown 中，可读性强。
- meta-agent 能根据统一规则自主尝试多轮 harness 改造。
- `agent.py` 保持为清晰的被测对象，便于比较不同实验版本。

### 4.2 单文件 harness，降低修改面

主实现把 prompt、工具注册、agent 构造、运行函数、Harbor 适配集中在 `agent.py`，降低 meta-agent 找文件和跨模块理解的成本。对自动改造场景来说，单文件有两个优势：

- 搜索空间小，模型更容易定位可编辑区。
- commit/diff 更集中，实验结果更容易关联到具体变化。

代价是随着工具和子 agent 增多，文件会变长，需要通过清晰分区和注册函数控制复杂度。

### 4.3 Harbor 评测兼容

项目使用 Harbor 的 `BaseAgent` / `BaseEnvironment` 抽象：

- Harbor 负责加载 agent、创建任务容器、执行 verifier、收集日志。
- AutoAgent 只需要实现 `setup()` 和 `run()`。
- `BaseEnvironment.exec()` 成为模型操作任务容器的统一通道。

这样同一个 harness 可以复用 Harbor 的任务格式、并发执行、缓存、日志和分数统计。

### 4.4 容器隔离与 host-side brain

当前 `agent.py` 的模型运行在 Harbor agent 进程一侧；shell 工具通过 `environment.exec()` 进入任务容器。也就是说：

- 模型和 OpenAI Agents SDK 在 host/runner 侧运行。
- 任务文件、测试依赖和命令执行在 Harbor 任务容器内发生。
- `run_shell` 是 host-side agent 与 container-side workspace 的桥。

这种设计避免模型直接在容器中长期驻留，同时保留容器隔离和可重建的任务环境。

### 4.5 分数驱动与简单性标准

`program.md` 的核心约束是：

- `passed` 增加则保留。
- `passed` 不变但 harness 更简单则保留。
- 其他情况丢弃。

这使优化循环尽量避免过拟合和复杂化。对 agent harness 来说，复杂工具链可能提升个别任务，却降低泛化和可维护性，因此“简单性”被列为同级决策标准。

## 5. 总体架构

### 5.1 组件视图

```text
Human / External Meta-Agent
        |
        | reads and follows
        v
program.md  ---- explains rules ----> README.md
        |
        | edits editable harness region
        v
agent.py
  |-- create_tools() -> run_shell()
  |-- create_agent() -> OpenAI Agents SDK Agent
  |-- run_task()     -> Runner.run(...)
  |-- to_atif()      -> trajectory.json
  `-- AutoAgent     -> Harbor BaseAgent adapter
        |
        | loaded by
        v
Harbor runner
        |
        | BaseEnvironment.exec/upload_file
        v
Task container built from Dockerfile.base + task/environment/Dockerfile
        |
        | verifier writes score/logs
        v
jobs/ + run.log + results.tsv
```

### 5.2 运行时调用链

```text
uv run harbor run ... --agent-import-path agent:AutoAgent
  -> Harbor imports agent.py:AutoAgent
  -> AutoAgent.run(instruction, environment, context)
  -> mkdir -p /task in task container
  -> upload instruction.md to /task/instruction.md
  -> run_task(environment, instruction)
  -> create_agent(environment)
  -> Runner.run(agent, input=instruction, max_turns=30)
  -> model may call run_shell(command)
  -> run_shell delegates to environment.exec(command, timeout_sec=120)
  -> task container returns stdout/stderr
  -> Runner returns RunResult
  -> to_atif(result, model, duration_ms)
  -> logs_dir/trajectory.json
  -> Harbor context token metrics updated
```

### 5.3 数据流

| 数据 | 来源 | 去向 | 说明 |
| --- | --- | --- | --- |
| `instruction` | Harbor task `instruction.md` | `AutoAgent.run()` / `Runner.run()` | 模型收到的自然语言任务。 |
| `/task/instruction.md` | `AutoAgent.run()` 上传 | 任务容器 | 兼容需要从固定路径读取任务说明的工具或脚本。 |
| shell command | 模型工具调用 | `BaseEnvironment.exec()` | 在任务容器内执行，不直接在 host 执行。 |
| stdout/stderr | 任务容器命令 | `run_shell` 返回给模型 | stderr 会被拼接到输出文本中，便于模型诊断。 |
| `RunResult.new_items` | OpenAI Agents SDK | `to_atif()` | 转换成 ATIF step。 |
| token usage | SDK raw responses | Harbor `AgentContext` | 记录输入、输出、缓存 token。 |
| `trajectory.json` | `to_atif()` | Harbor logs | 供失败分析、复盘、打分审计使用。 |
| `reward.txt` | task verifier | Harbor job logs | 任务分数来源，通常由 test 脚本写入。 |

## 6. `agent.py` 主实现详解

### 6.1 配置区

```python
SYSTEM_PROMPT = "You are an agent that executes tasks"
MODEL = "gpt-5"
MAX_TURNS = 30
```

- `SYSTEM_PROMPT` 是当前最小化 prompt，只描述“执行任务”。这保留了很大的后续优化空间。
- `MODEL` 固定为 `gpt-5`；`program.md` 明确要求除非人类改变约束，不要改模型。
- `MAX_TURNS` 限制 `Runner.run()` 最多 30 轮，避免无限循环或异常成本。

### 6.2 工具创建：`create_tools(environment)`

当前只有一个工具：`run_shell(command: str) -> str`。

行为细节：

1. 用 `@function_tool` 暴露给 OpenAI Agents SDK。
2. 调用 `await environment.exec(command=command, timeout_sec=120)`。
3. 如果有 stdout，直接拼接。
4. 如果有 stderr，以 `STDERR:` 标签拼接到返回文本。
5. 如果 stdout/stderr 都为空，返回 `(no output)`。
6. 如果执行抛异常，捕获并返回 `ERROR: ...`，让模型看到可恢复错误。

设计含义：

- 工具粒度非常通用，适合 coding/terminal 类任务。
- 120 秒超时保护单次命令，避免长时间卡死。
- 返回文本而非结构化对象，简单但会消耗更多 token。
- `program.md` 已指出后续可添加更专业工具，例如表格检查、定向读写、验证工具。

### 6.3 Agent 构造：`create_agent(environment)`

`create_agent()` 负责把工具列表注入 `Agent`：

```python
return Agent(
    name="autoagent",
    instructions=SYSTEM_PROMPT,
    tools=tools,
    model=MODEL,
)
```

这里是未来扩展 sub-agent、handoff、`agent.as_tool()` 的主要位置。当前实现没有路由层、没有专用 reviewer agent，也没有动态工具选择。

### 6.4 执行函数：`run_task(environment, instruction)`

`run_task()` 是 SDK 层的最小编排：

1. 调用 `create_agent(environment)`。
2. 记录开始时间。
3. 调用 `Runner.run(agent, input=instruction, max_turns=MAX_TURNS)`。
4. 计算 `duration_ms`。
5. 返回 `(result, duration_ms)`。

这个函数是“多阶段执行”和“自检循环”的自然扩展点。例如后续可在这里先让主 agent 执行，再让 verification sub-agent 检查产物，必要时回到主 agent 修正。

### 6.5 ATIF 转换：`to_atif(result, model, duration_ms)`

`to_atif()` 把 OpenAI Agents SDK 的 `RunResult` 转成 ATIF v1.6 轨迹，便于 Harbor 统一消费。

它处理的 item 类型包括：

- `MessageOutputItem`：提取模型输出文本，记录为 `source=agent` 的 step。
- `ReasoningItem`：如果 raw item 有 summary，则写入 `reasoning_content`。
- `ToolCallItem`：暂存工具调用。
- `ToolCallOutputItem`：把暂存的工具调用和工具输出组合成带 `tool_calls` 与 `observation` 的 step。

指标处理：

- 遍历 `result.raw_responses`。
- 用 `Usage().add(response.usage)` 聚合 token。
- 写入 `total_prompt_tokens`、`total_completion_tokens`、`total_cached_tokens`、`duration_ms`、`num_turns`。

需要注意的实现边界：当前 `pending_tool_call` 是单个变量，适合当前单工具、串行工具调用假设；如果未来启用并行工具调用，应改成按 `call_id` 管理的字典，避免并发调用覆盖。

### 6.6 Harbor 适配器：`AutoAgent`

`AutoAgent` 继承 Harbor 的 `BaseAgent`，暴露给命令行 `--agent-import-path agent:AutoAgent`。

关键行为：

- `SUPPORTS_ATIF = True`：声明会输出 ATIF 轨迹。
- `name()` 返回 `autoagent`。
- `version()` 返回 `0.1.0`。
- `setup()` 为空，表示当前没有预启动环境准备。
- `run()` 执行完整任务生命周期。

`run()` 详细步骤：

1. 在任务容器中创建 `/task`。
2. 把 instruction 写入 agent 日志目录下的 `instruction.md`。
3. 上传该文件到任务容器 `/task/instruction.md`。
4. 调用 `run_task()`，由模型通过工具操作容器。
5. 调用 `to_atif()` 生成轨迹字典。
6. 写入 `logs_dir/trajectory.json`。
7. 从 `final_metrics` 回填 Harbor `AgentContext` 的 token 字段。
8. 聚合 usage 并打印一行 summary：turns、duration、input、output。

这层代码是 Harbor 与 SDK 的边界，通常不应由实验 meta-agent 修改。

## 7. `agent-claude.py` 备选实现

`agent-claude.py` 展示了另一种架构：Claude SDK/Claude Code 在任务容器内运行，Harbor 适配器在 host 侧只负责启动容器内脚本。

### 7.1 配置特点

- `TOOLS_PRESET = {"type": "preset", "preset": "claude_code"}`：使用 Claude Code 预设工具。
- `AGENT_CWD = <repo>/.agent`：把 `.agent/` 作为 agent 工作目录。
- `THINKING = {"type": "enabled", "budget_tokens": 10000}`：开启 thinking budget。
- `MODEL = "haiku"`：使用 Claude Haiku 系列模型名简写。
- `permission_mode="bypassPermissions"`：跳过交互式权限确认。

### 7.2 运行链路

```text
Harbor imports AutoAgent from agent-claude.py-compatible module
  -> AutoAgent.run(...)
  -> upload /task/instruction.md
  -> collect .env + extra_env
  -> environment.exec("cd /app && python agent.py", timeout_sec=600)
  -> container-side _run_in_container()
  -> ClaudeSDKClient.query(instruction)
  -> receive_response() collects messages
  -> _trajectory_to_atif(...)
  -> /logs/agent/trajectory.json
```

与主实现的差异：

- 主 `agent.py` 是 host-side OpenAI SDK brain，工具调用进容器。
- `agent-claude.py` 是 container-side Claude Code brain，SDK 客户端在容器内执行。
- 主实现输出 ATIF v1.6；Claude 备选实现输出 ATIF v1.2。
- 主实现的运行命令直接使用 `agent:AutoAgent`；Claude 文件名包含 `-`，不能直接作为普通 import path，需要重命名或包装。

## 8. 外部 meta-agent 实验流程

`program.md` 是外部 meta-agent 的作业说明，它要求“不要直接解决 benchmark 任务”，而是提高 `agent.py` 这个 harness 的泛化能力。

### 8.1 启动前检查

meta-agent 开始实验前应：

1. 读取 `README.md`、`program.md`、`agent.py`。
2. 读取本地补充文档和 OpenAI Agents SDK 工具模式文档；当前仓库的 `program.md` 提到 `docs/good-harness.md` 和 `docs/openai-agents-sdk/tools.md`，但基线中这两个文件不存在。
3. 如果当前分支包含 `tasks/`，抽样阅读 task instruction 和 verifier。
4. 检查运行依赖是否缺失。
5. 只在必要时修改 `pyproject.toml` 或 `Dockerfile.base`。
6. 构建基础镜像，并验证 agent 可导入。
7. 初始化 `results.tsv`。
8. 第一次运行必须是未修改 baseline。

### 8.2 可修改范围

允许修改 `agent.py` 固定边界上方：

- `SYSTEM_PROMPT`、`MODEL`、`MAX_TURNS`。
- `create_tools(environment)`。
- `create_agent(environment)`。
- `run_task(environment, instruction)`。

不应修改固定边界下方的 Harbor adapter 和 ATIF 序列化，除非人类明确要求。

### 8.3 实验循环

标准循环是：

1. 记录当前分支和 commit。
2. 阅读最近的 `run.log` 和 task 级结果。
3. 从 trajectory 与 verifier 日志诊断失败任务。
4. 按根因归类失败。
5. 选择一个通用 harness 改进。
6. 修改 harness。
7. commit。
8. 重建并重跑任务集。
9. 写入 `results.tsv`。
10. 按 keep/discard 规则决定保留或丢弃。

`results.tsv` 预期列为：

```text
commit	avg_score	passed	task_scores	cost_usd	status	description
```

### 8.4 失败诊断维度

`program.md` 建议重点看：

- 任务理解错误。
- 缺能力或缺工具。
- 信息收集不足。
- 执行策略差。
- 缺验证。
- 环境或依赖问题。
- agent 以为完成但产物错误的 silent failure。

这些维度也解释了为什么项目鼓励添加专业工具和 verification sub-agent，而不是只调 prompt。

## 9. 启动方案

### 9.1 本地前置条件

需要准备：

- Docker / Docker Desktop。
- Python 3.12+；`pyproject.toml` 和 README 均要求 3.12+，当前 `.python-version` 是 `3.13`。
- `uv`。
- OpenAI Agents SDK 所需凭证；当前主实现使用 `MODEL = "gpt-5"`，通常需要 `OPENAI_API_KEY`。
- Harbor 任务目录 `tasks/`；当前基线仓库未包含任务。

### 9.2 安装依赖

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync
```

`pyproject.toml` 依赖包括：

- `openai-agents`
- `harbor`
- `pandas`
- `openpyxl`
- `numpy`

### 9.3 配置环境变量

主 `agent.py` 未显式读取 `.env`，但 OpenAI SDK 通常会从进程环境读取凭证。可用如下方式让 Harbor 运行命令继承变量：

```bash
cat > .env <<'EOF'
OPENAI_API_KEY=...
EOF

set -a && source .env && set +a
```

如果改用 `agent-claude.py` 的架构，需要按 Claude SDK 要求提供相应 Anthropic/Claude 凭证，并确保容器内也能读到这些环境变量。

### 9.4 构建基础镜像

```bash
docker build -f Dockerfile.base -t autoagent-base .
```

`Dockerfile.base` 做了以下事情：

1. 基于 `ghcr.io/astral-sh/uv:python3.12-bookworm-slim`。
2. 安装 `ca-certificates` 和 `git`。
3. 设置 `/app` 为工作目录。
4. 复制 `pyproject.toml` 并执行 `uv pip install --system .`。
5. 复制 `agent.py` 到镜像。
6. 创建 `/logs` 和 `/app/output`。

注意：`.dockerignore` 排除了 `docs/`、`tasks/`、`jobs/` 和运行日志，因此文档和任务数据不会被直接复制进基础镜像。任务内容应通过 Harbor task 格式提供。

### 9.5 准备 Harbor 任务

当前仓库没有 `tasks/`。需要按 Harbor 格式添加，例如：

```text
tasks/my-task/
  task.toml
  instruction.md
  tests/
    test.sh
    test.py
  environment/
    Dockerfile
  files/
```

约定上，task verifier 将分数写入 `/logs/reward.txt`，Harbor 再汇总到 job 输出中。

### 9.6 运行单个任务

```bash
rm -rf jobs; mkdir -p jobs && \
uv run harbor run -p tasks/ \
  --task-name "<task-name>" \
  -l 1 \
  -n 1 \
  --agent-import-path agent:AutoAgent \
  -o jobs \
  --job-name latest > run.log 2>&1
```

参数含义：

- `-p tasks/`：任务根目录。
- `--task-name`：只运行指定任务。
- `-l 1`：限制运行数量。
- `-n 1`：并发数为 1，便于调试。
- `--agent-import-path agent:AutoAgent`：加载 `agent.py` 中的 `AutoAgent`。
- `-o jobs`：输出目录。
- `--job-name latest`：job 名称。
- `> run.log 2>&1`：保存 stdout/stderr。

### 9.7 运行全量任务

```bash
rm -rf jobs; mkdir -p jobs && \
uv run harbor run -p tasks/ \
  -n 100 \
  --agent-import-path agent:AutoAgent \
  -o jobs \
  --job-name latest > run.log 2>&1
```

`-n 100` 表示较高并发，适合任务数较多且 Docker 资源充足的机器。调试阶段建议降低并发，先稳定复现单任务失败。

### 9.8 运行 meta-agent 实验

外部编码 agent 的启动提示可参考 README：

```text
Read program.md and let's kick off a new experiment!
```

它应该遵循：

1. 先跑未修改 baseline。
2. 读取失败 trajectory 和 verifier 日志。
3. 选择一个通用改进。
4. 修改 `agent.py` 可编辑区域。
5. 重跑 benchmark。
6. 记录 `results.tsv`。
7. 根据 keep/discard 规则保留或回滚。

## 10. 关键扩展点

### 10.1 Prompt 与模型控制

位置：`agent.py` 的 `SYSTEM_PROMPT`、`MODEL`、`MAX_TURNS`。

适合改进：

- 增加任务理解、文件检查、验证、交付格式的系统规则。
- 调整最大轮数以适应复杂任务。
- 在人类允许时切换模型。

风险：prompt 过长会消耗 token，且容易引入僵硬流程；模型切换会破坏结果可比性。

### 10.2 工具层

位置：`create_tools(environment)`。

适合添加：

- 结构化文件查看工具。
- 表格/CSV/Excel inspection 和 validated write 工具。
- 测试运行工具。
- 产物存在性检查工具。
- 日志摘要工具。

工具设计原则：

- 工具名要贴近模型先验。
- 返回结构化结果，避免超长 stdout。
- 错误消息要可操作。
- 避免为单个 benchmark 写硬编码工具。

### 10.3 Agent 编排层

位置：`create_agent(environment)` 和 `run_task(environment, instruction)`。

适合添加：

- verification sub-agent。
- plan/execute/review 多阶段流程。
- `agent.as_tool()` 形式的专用子 agent。
- 失败后自动读取日志并二次修复的 loop。

风险：编排越复杂，调试越难；必须用 benchmark 分数和 simplicity criterion 证明价值。

### 10.4 Harbor 适配层

位置：`AutoAgent.run()` 和 `to_atif()`。

通常不改。只有在以下情况才考虑：

- ATIF schema 需要升级。
- Harbor context 指标字段变化。
- 需要支持并行工具调用的准确轨迹记录。
- 任务系统要求额外上传/下载文件。

## 11. 观测、日志与排障

### 11.1 主要产物

| 产物 | 位置 | 用途 |
| --- | --- | --- |
| `run.log` | 仓库根目录 | Harbor 运行日志，便于快速看失败。 |
| `jobs/` | 仓库根目录 | Harbor job 输出，包含 task 级 logs。 |
| `trajectory.json` | Harbor agent logs | ATIF 轨迹，用于分析模型消息、工具调用、token。 |
| `results.tsv` | 仓库根目录 | meta-agent 实验记录。 |
| `/task/instruction.md` | 任务容器内 | agent 可通过 shell 查看任务说明。 |
| `/logs/reward.txt` | 任务容器 verifier logs | verifier 分数输出。 |

### 11.2 常见问题

| 问题 | 现象 | 排查/处理 |
| --- | --- | --- |
| 未提供 `tasks/` | Harbor 找不到任务或没有可运行项 | 添加 Harbor 格式任务，或切换到 benchmark 分支。 |
| 缺 API key | SDK 初始化或模型调用失败 | 确认 shell 环境中有 `OPENAI_API_KEY`，必要时 `set -a && source .env && set +a`。 |
| Docker 未启动 | build/run 失败 | 启动 Docker Desktop，再重试构建和 Harbor run。 |
| 单命令超时 | `run_shell` 返回错误或长时间等待 | 拆分命令，或在 harness 中调整 timeout。 |
| ATIF 缺工具输出 | trajectory 难以复盘工具调用 | 检查 `pending_tool_call` 串行假设；并行工具需改为按 call id 匹配。 |
| `program.md` 引用文档不存在 | meta-agent 启动时找不到 `docs/good-harness.md` 等 | 当前基线缺这些文件；可补充文档或让 meta-agent 跳过不存在的本地上下文。 |

### 11.3 清理命令

```bash
uv run harbor cache clean -f
docker container prune -f
docker system prune -a -f
```

如果 Docker Desktop 在大量并发任务后无响应：

```bash
killall Docker && open -a Docker
```

## 12. 当前状态评估

### 12.1 优点

- 主链路很短，容易理解和改造。
- Harbor 适配边界清晰。
- `run_shell` 通用，适合多数 coding/terminal 任务。
- ATIF 输出覆盖消息、reasoning summary、工具调用、工具观察和 token 指标。
- `program.md` 对实验纪律、反过拟合和 keep/discard 规则写得明确。

### 12.2 风险与限制

- 当前 prompt 极简，复杂任务可能缺少计划、验证和交付纪律。
- 只有通用 shell 工具，结构化任务会浪费 token。
- `pending_tool_call` 只支持单个挂起工具调用的记录方式。
- 当前基线无 `tasks/`，无法直接跑 benchmark。
- `program.md` 引用的部分 `docs/` 文件不存在。

### 12.3 后续改造优先级建议

这些不是本次文档任务的实现范围，但可作为后续实验候选：

1. 加强 `SYSTEM_PROMPT`，要求先读任务、列计划、执行、验证、交付。
2. 增加结构化文件/目录查看工具，减少 shell 输出噪声。
3. 增加产物验证工具或 reviewer sub-agent。
4. 把 ATIF 工具调用匹配改为按 `call_id` 的字典，支持并行工具。
5. 补齐 `program.md` 引用的本地 docs，降低 meta-agent 启动失败概率。

## 13. 验证清单

接手者可用下面的顺序验证仓库和文档描述是否仍然成立：

```bash
# 文件存在性
test -f agent.py
test -f program.md
test -f README.md
test -f pyproject.toml
test -f Dockerfile.base

# 关键符号
rg -n "class AutoAgent|def create_tools|def create_agent|async def run_task|def to_atif" agent.py

# 依赖与 Python 版本
rg -n "requires-python|openai-agents|harbor" pyproject.toml

# 启动命令来源
rg -n "docker build|uv run harbor run|--agent-import-path agent:AutoAgent" README.md program.md

# 文档差异检查
git diff --check
```

如果要验证真实运行链路，还需要先准备 `tasks/` 和模型凭证，然后运行单任务命令，检查 `run.log`、`jobs/`、`trajectory.json` 和 verifier 分数。
