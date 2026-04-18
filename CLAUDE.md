# CLAUDE.md — Hermes Agent 项目指南

本文件为 AI 助手（如 Claude Code）提供项目上下文，以便更好地协助开发工作。

## 项目概述

Hermes Agent 是由 Nous Research 开发的自我进化 AI 智能体框架（v0.10.0），支持 200+ LLM 模型和 18+ 消息平台，可在本地、Docker、SSH、Modal 等多种环境中运行。

## 常用命令

```bash
# 安装依赖
uv pip install -e ".[all,dev]"

# 运行全部单元测试（排除集成和 e2e 测试，并行执行）
python -m pytest tests/ -q --ignore=tests/integration --ignore=tests/e2e -n auto

# 运行特定目录的测试
python -m pytest tests/agent/ -q
python -m pytest tests/tools/ -q
python -m pytest tests/gateway/ -q

# 运行集成测试（需要 API 密钥）
python -m pytest tests/integration/ -q

# 运行 e2e 测试
python -m pytest tests/e2e/ -q

# 启动交互式对话
hermes

# 启动消息网关
hermes gateway start

# 问题诊断
hermes doctor
```

## 项目结构

```
run_agent.py          # 核心 AIAgent 类，对话循环主逻辑
cli.py                # 交互式 CLI 编排器
model_tools.py        # 工具编排与发现（导入工具模块触发自注册）
toolsets.py           # 工具集定义与组合
hermes_state.py       # SQLite 会话存储（WAL 模式 + FTS5 全文搜索）
agent/                # Agent 内部组件（提示构建、上下文压缩、模型适配器等 33 模块）
tools/                # 72 个工具实现（自注册模式）
tools/environments/   # 6 种终端后端（local, docker, ssh, modal, daytona, singularity）
gateway/              # 消息网关（18 平台适配器）
hermes_cli/           # CLI 子命令（50 模块）
skills/               # 28 类内置技能
optional-skills/      # 11 类可选技能
plugins/              # 插件系统（记忆后端、上下文引擎、仪表盘）
cron/                 # 定时任务调度
acp_adapter/          # 编辑器集成（VS Code/Zed/JetBrains）
tests/                # ~3000 个 pytest 测试
```

## 架构要点

- **工具自注册**: 每个 `tools/*.py` 在导入时调用 `registry.register()` 自动注册。`model_tools.py` 负责导入所有工具模块触发发现。
- **配置目录**: 始终使用 `hermes_constants` 中的 `get_hermes_home()`，不要硬编码 `~/.hermes`。
- **会话持久化**: 所有对话存储在 SQLite（`hermes_state.py`），系统提示为临时注入，不持久化到数据库。
- **技能 vs 工具**: 能通过指令 + shell 命令 + 现有工具表达的做成技能（Skill）；需要 API 密钥管理、二进制数据处理或流式事件的做成工具（Tool）。

## 代码风格

- 遵循 PEP 8，行长度不严格限制
- 注释仅用于非显而易见的意图、权衡或 API 细节，不要重述代码逻辑
- 捕获具体异常类型；对意外错误使用 `logger.warning()`/`logger.error()` 并带 `exc_info=True`
- 跨平台兼容：不假设 Unix 环境，对 `termios`/`fcntl` 用 `ImportError` 防护；使用 `pathlib.Path` 处理文件路径
- 安全：对 shell 插值使用 `shlex.quote()`；路径检查前用 `os.path.realpath()` 解析符号链接；绝不记录密钥

## 提交规范

使用 Conventional Commits 格式：

```
<type>(<scope>): <description>
```

- **type**: `fix`, `feat`, `docs`, `test`, `refactor`, `chore`
- **scope**: `cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `security` 等

示例：
```
fix(cli): prevent crash in save_config_value when model is a string
feat(gateway): add WhatsApp multi-user session isolation
fix(security): prevent shell injection in sudo password piping
```

## 测试约定

- 测试框架: pytest + pytest-xdist（并行） + pytest-asyncio（异步）
- `@pytest.mark.integration` 标记需要外部服务的测试
- `tests/conftest.py` 中的 `_isolate_hermes_home` fixture 自动将 `~/.hermes` 重定向到临时目录
- CI 中 API 密钥设为空字符串，防止意外真实调用

## 分支与 PR

- 分支命名: `fix/`, `feat/`, `docs/`, `test/`, `refactor/`
- PR 提交前: 运行测试、手动验证、检查跨平台影响、保持 PR 聚焦
- PR 描述: 包含改动内容/原因、测试方法、已测平台、关联 issue
