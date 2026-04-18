# Hermes Agent 项目文档

> **版本**: 0.10.0 | **许可证**: MIT | **Python**: >= 3.11

Hermes Agent 是由 [Nous Research](https://nousresearch.com/) 开发的**自我进化 AI 智能体框架**。它是一个功能完备的对话式 AI 系统，可在从 $5 VPS 到 GPU 集群再到无服务器基础设施等各种环境中运行。

---

## 目录

- [核心特性](#核心特性)
- [项目架构](#项目架构)
- [核心模块详解](#核心模块详解)
- [支持的 LLM 提供商](#支持的-llm-提供商)
- [支持的消息平台](#支持的消息平台)
- [安装与配置](#安装与配置)
- [常用 CLI 命令](#常用-cli-命令)
- [部署方式](#部署方式)
- [工具系统](#工具系统)
- [技能系统](#技能系统)
- [插件系统](#插件系统)
- [测试](#测试)
- [技术栈](#技术栈)

---

## 核心特性

| 特性 | 说明 |
|------|------|
| **多模型支持** | 通过 OpenRouter 支持 200+ 模型，或直连 Anthropic、OpenAI、Google Gemini、Mistral、Bedrock 等 API |
| **多平台消息接入** | Telegram、Discord、Slack、WhatsApp、Signal、微信、钉钉、飞书等，统一网关管理 |
| **自我学习** | 自动从交互中创建技能，在使用过程中自我优化，主动持久化知识 |
| **工具调用** | 完整的智能体循环，支持自动重试、错误恢复、工具结果流式传输 |
| **上下文压缩** | 当接近 token 上限时，自动压缩消息历史 |
| **提示缓存** | 支持 Anthropic 提示缓存，优化成本 |
| **浏览器自动化** | 通过 Browserbase 和 Camofox 实现无头浏览器操控 |
| **网页研究** | 并行网页抓取 + Firecrawl 内容提取 |
| **终端隔离** | 6 种后端：本地、Docker、SSH、Modal、Daytona、Singularity |
| **记忆系统** | 持久化记忆（MEMORY.md、USER.md）、Honcho 辩证式用户建模 |
| **会话搜索** | 基于 FTS5 全文搜索，配合 LLM 摘要 |
| **费用跟踪** | Token 计数、预估/实际费用、按提供商定价 |
| **定时任务** | 内置 Cron 调度器，支持自然语言定义任务 |
| **编辑器集成** | VS Code、Zed、JetBrains 通过 Agent Client Protocol (ACP) 接入 |
| **批量处理** | 并行轨迹生成，支持 RL 训练 |
| **安全防护** | 命令审批、危险命令检测、路径遍历保护 |

---

## 项目架构

### 顶层目录结构

```
hermes-agent/
├── run_agent.py              # 核心 AIAgent 类 — 对话循环
├── cli.py                    # 交互式 CLI 编排器
├── batch_runner.py           # 并行批量处理
├── model_tools.py            # 工具编排与发现
├── toolsets.py               # 工具集定义系统
├── hermes_state.py           # SQLite 会话存储（FTS5 搜索）
├── hermes_cli/               # CLI 子命令（50 个模块）
├── agent/                    # Agent 内部组件（33 个模块）
├── tools/                    # 72 个工具实现
├── gateway/                  # 消息平台网关（19 个模块）
├── acp_adapter/              # 编辑器集成（VS Code/Zed/JetBrains）
├── cron/                     # 定时任务调度器
├── plugins/                  # 插件系统（记忆、上下文、仪表盘）
├── skills/                   # 28+ 内置技能
├── optional-skills/          # 11 类可选技能
├── environments/             # RL 训练环境（Atropos）
├── tests/                    # ~3000 个 pytest 测试
├── web/                      # Web 界面组件
├── website/                  # 文档站点
├── docker/                   # 容器配置
├── scripts/                  # 安装与工具脚本
├── pyproject.toml            # 项目依赖与构建配置
└── flake.nix                 # NixOS 开发环境
```

### 依赖关系图

```
tools/registry.py（零依赖，最先导入）
    ↑
tools/*.py（每个工具在导入时自注册）
    ↑
model_tools.py（发现所有工具）
    ↑
run_agent.py / cli.py / gateway/run.py / batch_runner.py
```

---

## 核心模块详解

### 1. Agent 核心 — `run_agent.py`

项目的核心组件，实现了 `AIAgent` 类，负责整个对话循环。

**主要类**: `AIAgent`

```python
class AIAgent:
    def __init__(self, model, max_iterations, enabled_toolsets, ...):
        ...
    def chat(self, message) -> str:                    # 简单对话接口
        ...
    def run_conversation(self, user_message, system_message, history, task_id) -> dict:
        ...                                            # 完整对话循环
```

**核心能力**:
- 同步对话循环，支持工具调用
- Token 预算跟踪与上下文压缩
- 多模型提供商支持，自动故障转移
- 轨迹保存，用于 RL 训练
- 记忆上下文构建与会话持久化
- Anthropic 提示缓存成本优化

**Agent 执行流程**:
1. 组装消息，进行上下文压缩
2. 调用 LLM API（支持模型故障转移/路由）
3. 工具调用循环（直到 `stop_reason != "tool_calls"`）
4. 执行工具，缓存结果
5. 费用跟踪与会话持久化

### 2. 工具系统 — `tools/`

包含 72 个工具实现，采用自注册模式。

**注册机制**: 中央注册表 `tools/registry.py`，所有工具在导入时自动注册。

| 类别 | 工具 | 说明 |
|------|------|------|
| **网页** | `web_search`, `web_extract` | 网页搜索与内容提取 |
| **终端** | `terminal`, `process` | 本地/远程命令执行、后台进程管理 |
| **文件** | `read_file`, `write_file`, `patch`, `search_files` | 文件读写与搜索 |
| **浏览器** | `browser_navigate`, `browser_click` 等 | Browserbase 浏览器自动化 |
| **视觉** | `vision_analyze` | 图片分析（Base64） |
| **代码** | `execute_code` | Python 沙箱执行 |
| **委托** | `delegate_task` | 子智能体派生 |
| **技能** | `skills_list`, `skill_view`, `skill_manage` | 技能管理 |
| **记忆** | `memory`, `session_search` | 持久化记忆、FTS5 会话搜索 |
| **MCP** | `mcp_tool` | Model Context Protocol 客户端 |
| **智能家居** | Home Assistant | 家庭自动化集成 |
| **定时任务** | `cronjob` | 定时任务管理 |
| **图片生成** | `image_generate` | 图片合成 |
| **语音** | `text_to_speech` | 文字转语音 |

### 3. 消息网关 — `gateway/`

统一管理所有消息平台的接入，支持跨平台消息同步。

**核心类**:
- `GatewayRunner` — 管理所有平台适配器
- `SessionStore` — 对话持久化
- `BasePlatformAdapter` — 平台适配器抽象基类

**网关执行流程**:
1. `hermes gateway start` 启动网关
2. 加载消息平台配置
3. 实例化各平台适配器
4. `GatewayRunner` 管理适配器生命周期
5. 每个适配器监听传入消息
6. 分发到 `AIAgent.run_conversation()` 处理
7. 结果存储至网关 `SessionStore`

### 4. 命令行界面 — `hermes_cli/`

包含 50 个子模块，提供完整的 CLI 交互体验。

**主要模块**:

| 模块 | 说明 |
|------|------|
| `main.py` | 主命令分发器 |
| `config.py` | 配置系统（YAML 加载、环境变量） |
| `setup.py` | 交互式安装向导 |
| `auth.py` | 认证流程（OAuth2、API Key） |
| `models.py` | 模型注册表与提供商信息 |
| `gateway.py` | 消息网关管理 |
| `plugins_cmd.py` | 插件管理命令 |
| `skills_hub.py` | 技能中心集成 |
| `doctor.py` | 问题诊断工具 |
| `backup.py` | 备份与恢复 |

**UI 特性**:
- **prompt_toolkit** — 自动补全、多行编辑、历史记录
- **Rich** — 格式化输出、旋转动画、面板
- **主题引擎** — 数据驱动的 UI 自定义

### 5. 会话存储 — `hermes_state.py`

基于 SQLite 的持久化存储层。

- **WAL 模式**: 支持并发读取
- **FTS5 全文搜索**: 对话历史全文检索
- **Schema v6**: 包含 `sessions`、`messages`、`messages_fts` 表
- **会话链**: 压缩触发的会话拆分，通过 `parent_session_id` 维护连续性
- **费用追踪**: Token 计数、预估/实际费用、各类 Token 分类统计

### 6. Agent 内部组件 — `agent/`

包含 33 个模块，负责智能体的核心运行逻辑。

| 模块 | 说明 |
|------|------|
| `prompt_builder.py` | 系统提示词组装、记忆/技能指导 |
| `context_compressor.py` | 接近 Token 上限时自动压缩 |
| `auxiliary_client.py` | 辅助 LLM（视觉、摘要等） |
| `model_metadata.py` | 上下文长度、Token 估算、提供商检测 |
| `prompt_caching.py` | Anthropic 提示缓存优化 |
| `memory_manager.py` | 记忆上下文构建 |
| `error_classifier.py` | API 错误分类，用于故障转移 |
| `anthropic_adapter.py` | Anthropic API 集成 |
| `bedrock_adapter.py` | AWS Bedrock 集成 |
| `gemini_cloudcode_adapter.py` | Google Gemini 集成 |
| `credential_pool.py` | 多凭据管理 |
| `usage_pricing.py` | Token 费用估算 |
| `rate_limit_tracker.py` | 速率限制跟踪 |
| `redact.py` | 敏感信息脱敏 |

### 7. 终端后端 — `tools/environments/`

支持 6 种执行环境：

| 后端 | 文件 | 说明 |
|------|------|------|
| **本地** | `local.py` | 宿主机原生 Shell |
| **Docker** | `docker.py` | Docker 容器隔离 |
| **SSH** | `ssh.py` | 远程 SSH 服务器 |
| **Modal** | `modal.py` | Modal 无服务器函数 |
| **Daytona** | `daytona.py` | Daytona 无服务器 IDE |
| **Singularity** | `singularity.py` | Singularity/Apptainer 容器（HPC） |

### 8. 调度系统 — `cron/`

内置 Cron 调度器，支持定时执行任务。

- 支持自然语言定义任务
- 无人值守任务执行
- 支持将结果投递到任意消息平台
- 通过 `croniter` 实现 Cron 表达式解析

### 9. 编辑器集成 — `acp_adapter/`

通过 Agent Client Protocol (ACP) 集成主流编辑器：

- **VS Code** — 完整插件支持
- **Zed** — 编辑器集成
- **JetBrains** — IDE 集成
- 有状态的会话管理
- 命令框架与事件流式传输

---

## 支持的 LLM 提供商

| 提供商 | 模型系列 | 说明 |
|--------|---------|------|
| **Anthropic** | Claude 3/3.5/4/4.5/4.6/4.7 | 原生支持、提示缓存、扩展思维 |
| **OpenAI** | GPT-4.x、GPT-5.x、o 系列 | 完整 API 支持 |
| **OpenRouter** | 200+ 模型 | 统一接入多种模型的路由服务 |
| **Google** | Gemini 2.5/3.x | Cloud Code 认证 |
| **AWS Bedrock** | Claude、Llama 等 | 跨区域模型支持 |
| **Mistral** | Mistral 系列 | 原生 SDK 集成 |
| **阿里巴巴** | 通义千问 Qwen 3.x | 中文大模型 |
| **Moonshot/Kimi** | K2.x | 中文大模型 |
| **MiniMax** | M2.x | 中文大模型 |
| **DeepSeek** | DeepSeek 系列 | 开源大模型 |
| **xAI** | Grok 系列 | X.AI 平台 |
| **自定义端点** | 任意模型 | 任何 OpenAI 兼容 API |

---

## 支持的消息平台

| 平台 | 类型 | 说明 |
|------|------|------|
| **Telegram** | 即时通讯 | 内联键盘、媒体、群组 |
| **Discord** | 即时通讯 | 富文本、嵌入、语音频道 |
| **Slack** | 即时通讯 | 消息、线程、表情回应 |
| **WhatsApp** | 即时通讯 | 消息投递、媒体支持 |
| **Signal** | 即时通讯 | 端到端加密 |
| **微信** | 即时通讯 | 中国市场 |
| **企业微信** | 企业通讯 | 团队/企业消息 |
| **飞书** | 企业通讯 | 字节跳动生态 |
| **钉钉** | 企业通讯 | 阿里巴巴生态 |
| **Matrix** | 即时通讯 | 开放联邦协议 |
| **Mattermost** | 即时通讯 | 自托管 Slack 替代品 |
| **QQ Bot** | 即时通讯 | QQ 平台集成 |
| **邮件** | 邮件 | SMTP/IMAP 集成 |
| **短信** | 短信 | 通过 Twilio 发送 |
| **BlueBubbles** | macOS | iMessage 桥接 |
| **Home Assistant** | IoT | 智能家居事件 |
| **Webhook** | 通用 | 通用 HTTP 回调 |
| **API Server** | REST API | OpenAI 兼容的 `/v1` 端点 |

---

## 安装与配置

### 环境要求

- Python >= 3.11
- Node.js 22（用于浏览器工具）
- Git

### 方式一：脚本安装

```bash
# 克隆仓库
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 创建虚拟环境
uv venv venv --python 3.11
source venv/bin/activate

# 安装依赖（全部功能）
uv pip install -e ".[all,dev]"

# 初始化配置目录
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml

# 配置 API 密钥
echo 'OPENROUTER_API_KEY=sk-or-v1-...' >> ~/.hermes/.env

# 运行安装向导
hermes setup
```

或使用一键安装脚本：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

### 方式二：Docker 安装

```bash
docker pull nousresearch/hermes-agent
docker run -it -v ~/.hermes:/opt/data nousresearch/hermes-agent
```

Docker 镜像特性：
- 基于 Debian 13.4，预装 Playwright、ffmpeg、git
- 使用 `uv` 管理 Python 依赖
- 非 root 用户 `hermes`（UID 10000）
- 持久化数据挂载于 `/opt/data`

### 方式三：NixOS

```bash
nix develop  # 进入开发环境
```

支持平台：`x86_64-linux`、`aarch64-linux`、`aarch64-darwin`

### 配置文件说明

| 文件 | 路径 | 说明 |
|------|------|------|
| `config.yaml` | `~/.hermes/config.yaml` | 主配置（模型、工具、显示、记忆等） |
| `.env` | `~/.hermes/.env` | API 密钥（独立存放保障安全） |
| `SOUL.md` | `~/.hermes/SOUL.md` | 智能体人设/性格指令 |
| `MEMORY.md` | `~/.hermes/MEMORY.md` | 内置持久化记忆 |
| `USER.md` | `~/.hermes/USER.md` | 用户画像记忆 |
| `state.db` | `~/.hermes/state.db` | SQLite 会话数据库 |

**配置优先级**（从高到低）：

1. 命令行参数
2. 环境变量
3. `~/.hermes/config.yaml`
4. `~/.hermes/.env`
5. 硬编码默认值

---

## 常用 CLI 命令

### 基础命令

```bash
hermes                  # 启动交互式对话
hermes chat             # 同上
hermes model            # 切换 LLM 模型/提供商
hermes config           # 管理配置
hermes setup            # 运行交互式配置向导
hermes doctor           # 诊断问题
```

### 技能管理

```bash
hermes skills           # 浏览、安装、管理技能
hermes skills list      # 列出已安装技能
hermes skills reset     # 重置内置技能
```

### 消息网关

```bash
hermes gateway start    # 启动消息网关
hermes platforms        # 查看平台配置
```

### 会话管理

```bash
hermes session          # 浏览会话历史
hermes memory           # 管理记忆
hermes logs             # 查看日志
```

### 定时任务

```bash
hermes cron             # 管理定时任务
```

### 编辑器集成

```bash
hermes acp              # 启动 ACP 服务（VS Code/Zed/JetBrains）
```

### 其他

```bash
hermes update           # 更新 Hermes
hermes backup           # 备份数据
hermes login            # 登录提供商
hermes provider         # 提供商管理
```

---

## 部署方式

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| **本地 CLI** | 直接运行 `hermes` 命令 | 个人开发、日常使用 |
| **Docker** | 容器化部署 | 服务器部署、环境隔离 |
| **SSH** | 远程终端后端 | 远程服务器执行 |
| **Modal** | 无服务器函数（空闲时休眠） | 按需计算、成本优化 |
| **Daytona** | 无服务器 IDE 平台 | 云端开发环境 |
| **Singularity** | HPC 容器运行时 | 高性能计算集群 |
| **Systemd** | 后台服务运行网关 | 长期运行消息网关 |
| **NixOS** | 声明式配置 | NixOS 用户 |

### API 服务器

网关内置 OpenAI 兼容的 REST API：

| 端点 | 说明 |
|------|------|
| `POST /v1/chat/completions` | 无状态聊天补全 |
| `POST /v1/responses` | 有状态 Responses API |
| `GET /v1/responses/{id}` | 获取响应 |
| `GET /v1/models` | 列出可用模型 |
| `POST /v1/runs` | 启动异步运行 |
| `GET /v1/runs/{id}/events` | SSE 事件流 |
| `GET /health` | 健康检查 |

---

## 工具系统

### 工具集（Toolsets）

工具通过灵活的工具集系统进行分组管理：

- `_HERMES_CORE_TOOLS` — CLI 和所有消息平台共享的核心工具
- 可组合的工具集：`web`、`search`、`vision`、`terminal`、`moa`（混合智能体）等
- 支持按平台配置工具启用/禁用
- 每个工具支持 `check_fn` 门控（环境检查）

### 工具注册

所有工具采用自注册模式，每个工具文件在导入时调用 `registry.register()` 自动注册。`model_tools.py` 通过 `discover_builtin_tools()` 动态发现所有工具。

---

## 技能系统

### 内置技能（28 类）

位于 `skills/` 目录：

| 类别 | 说明 |
|------|------|
| `research` | 研究与信息收集 |
| `software-development` | 软件开发辅助 |
| `devops` | DevOps 运维 |
| `mlops` | ML 运维 |
| `data-science` | 数据科学 |
| `github` | GitHub 集成 |
| `feeds` | 信息流订阅 |
| `note-taking` | 笔记 |
| `gaming` | 游戏 |
| `creative` | 创意内容 |
| `diagramming` | 图表绘制 |
| `apple` | Apple 生态集成 |
| `mcp` | Model Context Protocol |
| `red-teaming` | 红队测试 |
| `social-media` | 社交媒体 |
| `smart-home` | 智能家居 |

### 可选技能（11 类）

位于 `optional-skills/` 目录：

autonomous-ai-agents、blockchain、communication、creative、devops、email、health、mlops、mcp、migration、productivity、research、security

### 技能格式

每个技能包含 `SKILL.md` 清单文件，使用 frontmatter 定义：
- 名称、版本、作者
- 适用平台
- 前置条件
- 使用/不使用场景
- 安装配置步骤
- 快速参考示例

### 技能中心

集成 [agentskills.io](https://agentskills.io) 技能中心，支持搜索、浏览和安装社区技能。

---

## 插件系统

### 架构

- 支持从三个来源发现插件：内置（`plugins/`）、用户（`~/.hermes/plugins/`）、pip 入口点
- 每个插件需要 `plugin.yaml` 清单 + `__init__.py`（含 `register(ctx)` 函数）
- 通过 `PluginContext` 注册工具和 CLI 命令

### 钩子系统

支持以下生命周期钩子：
- `pre_tool_call` / `post_tool_call` — 工具调用前后
- `pre_llm_call` / `post_llm_call` — LLM 调用前后
- `on_session_start` / `on_session_end` — 会话开始/结束

### 记忆插件

内置 8 个记忆后端提供商：

| 插件 | 说明 |
|------|------|
| `honcho` | Honcho AI 辩证式用户建模 |
| `mem0` | Mem0 云记忆 |
| `retaindb` | RetainDB 持久化记忆 |
| `hindsight` | Hindsight 情景记忆 |
| `holographic` | 基于向量嵌入的全息记忆 |
| `openviking` | OpenViking 记忆引擎 |
| `supermemory` | Supermemory 向量记忆 |
| `byterover` | ByteRover 记忆后端 |

同一时间只能激活一个外部记忆提供商（内置记忆始终可用）。

---

## 测试

### 测试框架

- **pytest** — 主测试框架
- **pytest-asyncio** — 异步测试支持
- **pytest-xdist** — 并行执行（`-n auto`）
- 约 **3000** 个测试用例

### 测试目录结构

```
tests/
├── conftest.py           # 共享 fixtures（HERMES_HOME 隔离）
├── agent/                # Agent 核心测试
├── cli/                  # CLI 命令测试
├── tools/                # 工具实现测试
├── gateway/              # 网关/平台测试
├── hermes_cli/           # CLI 模块测试
├── skills/               # 技能加载测试
├── plugins/              # 插件测试
├── cron/                 # 调度器测试
├── environments/         # 终端后端测试
├── acp/                  # ACP 协议测试
├── run_agent/            # Agent 运行测试
├── integration/          # 集成测试（需要 API 密钥）
├── e2e/                  # 端到端测试
└── fakes/                # Mock 实现
```

### 运行测试

```bash
# 运行所有非集成测试（并行）
python -m pytest tests/ -q

# 运行特定目录的测试
python -m pytest tests/agent/ -q

# 运行集成测试（需要配置 API 密钥）
python -m pytest tests/integration/ -q
```

---

## 技术栈

### 核心依赖

| 库 | 版本 | 用途 |
|----|------|------|
| `openai` | >= 2.21.0 | OpenAI API 客户端 |
| `anthropic` | >= 0.39.0 | Anthropic API 客户端 |
| `httpx` | >= 0.28.1 | 异步 HTTP 客户端 |
| `rich` | >= 14.3.3 | 终端格式化输出 |
| `prompt_toolkit` | >= 3.0.52 | CLI 交互（补全、多行编辑） |
| `pydantic` | >= 2.12.5 | 数据验证 |
| `pyyaml` | >= 6.0.2 | YAML 配置解析 |
| `tenacity` | >= 9.1.4 | 重试机制 |
| `fire` | >= 0.7.1 | CLI 命令生成 |
| `firecrawl-py` | >= 4.16.0 | 网页内容提取 |
| `exa-py` | >= 2.9.0 | 搜索 API |
| `fal-client` | >= 0.13.1 | 图片生成 |
| `edge-tts` | >= 7.2.7 | 文字转语音 |
| `jinja2` | >= 3.1.5 | 模板引擎 |

### 可选依赖组

| 组 | 内容 |
|----|------|
| `messaging` | Telegram、Discord、Slack 客户端库 |
| `cron` | croniter（定时任务） |
| `voice` | faster-whisper、sounddevice（语音识别/播放） |
| `mcp` | Model Context Protocol 支持 |
| `rl` | atroposlib、tinker、wandb（强化学习训练） |
| `modal` | Modal 无服务器执行 |
| `daytona` | Daytona 环境支持 |
| `bedrock` | AWS Bedrock 集成 |
| `mistral` | Mistral AI SDK |
| `web` | FastAPI、Uvicorn（Web 服务） |
| `all` | 以上全部 |

### 构建与 CI/CD

- **包管理器**: uv（快速 Python 包管理）
- **构建系统**: setuptools（PEP 517）
- **CI**: GitHub Actions
  - `tests.yml` — pytest 测试套件
  - `docker-publish.yml` — Docker 镜像构建与发布
  - `supply-chain-audit.yml` — 依赖安全审计
  - `skills-index.yml` — 技能中心索引
  - `nix.yml` — NixOS 构建验证

---

## 安全特性

| 特性 | 说明 |
|------|------|
| **命令审批** | 可配置的命令执行审批模式 |
| **路径遍历保护** | `path_security.py` 验证文件操作 |
| **危险命令检测** | `approval.py` 检测 Shell 注入 |
| **凭据管理** | 独立 `.env` 文件、凭据池 |
| **提示注入防护** | 上下文脱敏、系统提示固定 |
| **供应链审计** | GitHub Actions 依赖安全检查 |
| **进程隔离** | 子进程 HOME 目录隔离 |

---

## 设计亮点

1. **无 LangChain/LlamaIndex** — 轻量化，直接调用 API
2. **SQLite 而非 JSONL** — 更好的查询能力、FTS5 搜索、并发访问
3. **提示缓存优先** — Anthropic 缓存标记内建于系统提示
4. **同步主循环** — 简化工具处理逻辑，同时支持异步工具
5. **灵活终端后端** — 适应多种部署场景
6. **开放技能标准** — 兼容 agentskills.io 注册表
7. **网关独立于 CLI** — 可在不同机器上独立运行
8. **研究就绪** — 一等公民的轨迹保存，支持 RL 训练
