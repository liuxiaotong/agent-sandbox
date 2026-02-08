<div align="center">

# AgentSandbox

**Code Agent 执行沙箱 - 可复现的 Docker 隔离执行环境**
**Reproducible Docker sandbox for Code Agent task execution and trajectory replay**

[![PyPI](https://img.shields.io/pypi/v/knowlyr-sandbox?color=blue)](https://pypi.org/project/knowlyr-sandbox/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![MCP](https://img.shields.io/badge/MCP-4_Tools-purple.svg)](#mcp-server)

[快速开始](#快速开始) · [CLI 命令](#cli-命令参考) · [MCP Server](#mcp-server) · [Knowlyr 生态](#knowlyr-生态)

</div>

---

**GitHub Topics**: `sandbox`, `code-agent`, `docker`, `execution-environment`, `trajectory-replay`, `mcp`

为 Code Agent 提供标准化的 Docker 沙箱执行环境，支持代码任务的隔离执行、状态快照与轨迹重放。

## 核心能力 / Core Capabilities

```
TaskConfig (repo + commit) → Docker 沙箱 → Agent 工具调用 → 轨迹记录 → 可复现重放
```

### 标准工具接口 / Standard Tool Interface

| 工具 | 功能 | 说明 |
|------|------|------|
| `file_read` | 读取文件 | 支持行号范围 |
| `file_write` | 写入文件 | 自动创建目录 |
| `shell` | 执行命令 | 超时控制 |
| `search` | 搜索代码 | 正则匹配 |
| `git` | Git 操作 | diff, log, status |

### 解决的问题 / Problems Solved

| 痛点 | 传统方案 | AgentSandbox |
|------|----------|--------------|
| **隔离性** | 在宿主机执行，有安全风险 | Docker 容器隔离 |
| **可复现** | 环境差异导致结果不一致 | 固定镜像 + commit |
| **可追踪** | 操作过程难以记录 | 完整轨迹记录与重放 |
| **资源控制** | 无限制的资源使用 | CPU/内存/超时限制 |

## 安装 / Installation

```bash
pip install knowlyr-sandbox
```

可选依赖:

```bash
pip install knowlyr-sandbox[mcp]   # MCP 服务器
pip install knowlyr-sandbox[dev]   # 开发工具
pip install knowlyr-sandbox[all]   # 全部功能
```

## 快速开始 / Quick Start

### CLI 使用 / CLI Usage

```bash
# 创建沙箱
knowlyr-sandbox create --repo https://github.com/user/repo --commit abc123

# 在沙箱中执行工具
knowlyr-sandbox exec <sandbox_id> --tool shell --params '{"command": "python -m pytest"}'

# 重置沙箱到初始状态
knowlyr-sandbox reset <sandbox_id>

# 重放执行轨迹
knowlyr-sandbox replay <sandbox_id> trajectory.json

# 列出活跃沙箱
knowlyr-sandbox list
```

### API 使用 / API Usage

```python
from agentsandbox import Sandbox, SandboxConfig
from agentsandbox.config import TaskConfig

# 配置
config = SandboxConfig(
    image="python:3.11-slim",
    timeout=300,
    memory_limit="512m",
)

task = TaskConfig(
    repo_url="https://github.com/user/repo",
    base_commit="abc123",
    test_command="pytest tests/",
)

# 创建沙箱
sandbox = Sandbox.create(config, task)

# 执行工具
result = sandbox.execute_tool("shell", {"command": "python -m pytest"})
print(f"Exit code: {result.exit_code}")
print(f"Output: {result.output}")

# 快照和重置
snapshot_id = sandbox.snapshot()
sandbox.reset()

# 清理
sandbox.close()
```

### 轨迹重放 / Trajectory Replay

```python
from agentsandbox.replay import replay_trajectory, Trajectory

# 从文件加载轨迹
trajectory = Trajectory.from_dict({
    "steps": [
        {"tool_name": "file_read", "params": {"path": "src/main.py"}},
        {"tool_name": "file_write", "params": {"path": "src/main.py", "content": "..."}},
        {"tool_name": "shell", "params": {"command": "pytest"}},
    ],
    "metadata": {"agent": "claude", "model": "claude-opus-4-20250514"}
})

# 重放
result = replay_trajectory(sandbox, trajectory)
print(f"成功: {result.success}")
print(f"偏离步骤: {result.divergence_step}")
```

---

## CLI 命令参考

| 命令 | 功能 |
|------|------|
| `knowlyr-sandbox create` | 创建沙箱环境 |
| `knowlyr-sandbox exec <id>` | 在沙箱中执行工具 |
| `knowlyr-sandbox reset <id>` | 重置沙箱到初始状态 |
| `knowlyr-sandbox replay <id> <file>` | 重放 Agent 执行轨迹 |
| `knowlyr-sandbox list` | 列出活跃沙箱 |

### create 选项

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--repo` | Git 仓库 URL | (必填) |
| `--commit` | 起始 commit SHA | (必填) |
| `--language` | 编程语言 | python |
| `--image` | Docker 镜像 | python:3.11-slim |
| `--timeout` | 超时 (秒) | 300 |
| `--memory` | 内存限制 | 512m |
| `--cpu` | CPU 限制 | 1.0 |

---

## MCP Server / Claude Integration

在 Claude Desktop / Claude Code 中直接使用。

### 配置 / Config

添加到 `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "knowlyr-sandbox": {
      "command": "uv",
      "args": ["--directory", "/path/to/agent-sandbox", "run", "python", "-m", "agentsandbox.mcp_server"]
    }
  }
}
```

### 可用工具 / Tools

| 工具 | 功能 |
|------|------|
| `create_sandbox` | 创建 Docker 沙箱执行环境 |
| `execute_tool` | 在沙箱中执行工具 (5 种标准工具) |
| `reset_sandbox` | 重置沙箱到初始状态 |
| `replay_trajectory` | 重放 Agent 执行轨迹 |

### 使用示例 / Usage Example

```
用户: 帮我在 https://github.com/user/repo 的 abc123 上创建沙箱并运行测试

Claude: [调用 create_sandbox]
        沙箱已创建: sandbox-xyz

        [调用 execute_tool: shell "pytest tests/"]
        测试结果:
        - 通过: 42
        - 失败: 3
        - 错误: 0
```

---

## 架构 / Architecture

```
src/agentsandbox/
├── config.py       # 沙箱和任务配置
├── sandbox.py      # 核心沙箱 (Docker 管理)
├── tools.py        # 标准工具接口 (5 种工具)
├── replay.py       # 轨迹重放
├── cli.py          # CLI 命令行
└── mcp_server.py   # MCP Server (4 工具)
```

---

## Knowlyr 生态 / Ecosystem

AgentSandbox 是 Knowlyr 生态的执行环境组件:

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                    Knowlyr 生态                                          │
├──────────────────┬──────────────────┬──────────────────┬──────────────────┬───────────────┤
│   DataRecipe     │    DataSynth     │    DataCheck     │  AgentSandbox    │   更多...      │
│     数据分析      │      数据合成     │      数据质检     │    Agent 沙箱     │               │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┼───────────────┤
│  · 逆向工程分析   │  · LLM批量生成    │  · 规则验证       │ · Docker 隔离    │               │
│  · Schema提取    │  · 种子数据扩充   │  · 重复检测       │ · 标准工具接口    │               │
│  · 成本估算      │  · 成本追踪       │  · 分布分析       │ · 轨迹重放       │               │
│  · 样例生成      │  · 交互/API模式   │  · 质量报告       │ · 状态快照       │               │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┴───────────────┘
```

### 生态项目

| 项目 | 功能 | 仓库 |
|------|------|------|
| **DataRecipe** | 数据集逆向分析 | [data-recipe](https://github.com/liuxiaotong/data-recipe) |
| **DataSynth** | 数据合成扩充 | [data-synth](https://github.com/liuxiaotong/data-synth) |
| **DataCheck** | 数据质量检查 | [data-check](https://github.com/liuxiaotong/data-check) |
| **AgentSandbox** | Agent 执行沙箱 | You are here |

### AgentSandbox 与数据流水线的关系

```
DataRecipe → DataSynth → DataCheck
                              ↓
                    AgentSandbox (执行环境)
                    ┌─────────────────────┐
                    │  Code Agent 在沙箱中  │
                    │  执行代码修改任务      │
                    │  生成可复现的轨迹      │
                    └─────────────────────┘
```

AgentSandbox 可与数据流水线协同使用: 用 DataSynth 生成代码任务，在 AgentSandbox 中执行，用 DataCheck 验证结果质量。

---

## API 参考

### SandboxConfig

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `image` | str | python:3.11-slim | Docker 镜像 |
| `timeout` | int | 300 | 超时 (秒) |
| `memory_limit` | str | 512m | 内存限制 |
| `cpu_limit` | float | 1.0 | CPU 限制 |
| `work_dir` | str | /workspace | 工作目录 |
| `network_enabled` | bool | False | 网络访问 |

### TaskConfig

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `repo_url` | str | "" | Git 仓库 URL |
| `base_commit` | str | "" | 起始 commit |
| `test_command` | str | "" | 测试命令 |
| `language` | str | python | 编程语言 |
| `setup_commands` | list | [] | 初始化命令 |

### ToolResult

| 属性 | 类型 | 说明 |
|------|------|------|
| `output` | str | 标准输出 |
| `exit_code` | int | 退出码 |
| `error` | str \| None | 错误信息 |
| `success` | bool | 是否成功 (属性) |

---

## License

[MIT](LICENSE)

---

<div align="center">
<sub>为 Code Agent 提供安全、可复现的执行环境</sub>
</div>
