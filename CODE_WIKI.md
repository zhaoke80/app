# Trae Agent — Code Wiki

> **项目地址**: https://github.com/bytedance/trae-agent  
> **版本**: 0.1.0  
> **许可证**: MIT  
> **Python 版本**: >= 3.12  
> **技术报告**: [arXiv:2507.23370](https://arxiv.org/abs/2507.23370)

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 整体架构](#2-整体架构)
- [3. 项目目录结构](#3-项目目录结构)
- [4. 核心模块职责](#4-核心模块职责)
  - [4.1 CLI 入口层 (cli.py)](#41-cli-入口层-clipy)
  - [4.2 Agent 层 (agent/)](#42-agent-层-agent)
  - [4.3 工具层 (tools/)](#43-工具层-tools)
  - [4.4 LLM 客户端层 (utils/llm_clients/)](#44-llm-客户端层-utilsllm_clients)
  - [4.5 配置管理 (utils/config.py)](#45-配置管理-utilsconfigpy)
  - [4.6 轨迹记录 (utils/trajectory_recorder.py)](#46-轨迹记录-utilstrajectory_recorderpy)
  - [4.7 Lakeview 摘要 (utils/lake_view.py)](#47-lakeview-摘要-utilslake_viewpy)
  - [4.8 MCP 客户端 (utils/mcp_client.py)](#48-mcp-客户端-utilsmcp_clientpy)
  - [4.9 CLI 控制台 (utils/cli/)](#49-cli-控制台-utilscli)
  - [4.10 Prompt 模块 (prompt/)](#410-prompt-模块-prompt)
  - [4.11 Docker 管理 (agent/docker_manager.py)](#411-docker-管理-agentdocker_managerpy)
  - [4.12 代码知识图谱 (tools/ckg/)](#412-代码知识图谱-toolsckg)
- [5. 关键类与函数说明](#5-关键类与函数说明)
- [6. 依赖关系](#6-依赖关系)
- [7. 评估系统 (evaluation/)](#7-评估系统-evaluation)
- [8. 项目运行方式](#8-项目运行方式)
- [9. 开发与测试](#9-开发与测试)
- [10. 路线图](#10-路线图)

---

## 1. 项目概述

**Trae Agent** 是一个基于 LLM（大语言模型）的通用软件工程智能体。它提供强大的 CLI 接口，能够理解自然语言指令并使用多种工具和 LLM 提供商执行复杂的软件工程工作流。

### 核心特性

| 特性 | 说明 |
|------|------|
| **Lakeview** | 为 Agent 每一步提供简短精炼的摘要总结 |
| **多 LLM 支持** | 兼容 OpenAI、Anthropic、Doubao、Azure、OpenRouter、Ollama、Google Gemini |
| **丰富工具生态** | 文件编辑、Bash 执行、顺序思考、JSON 编辑、代码知识图谱等 |
| **交互模式** | 支持对话式迭代开发界面 |
| **轨迹记录** | 详细记录所有 Agent 动作，用于调试和分析 |
| **灵活配置** | 基于 YAML 的配置，支持环境变量覆盖 |
| **Docker 模式** | 支持在 Docker 容器中隔离执行任务 |
| **MCP 集成** | 支持 Model Context Protocol 扩展工具能力 |

### 设计理念

Trae Agent 采用**透明、模块化**的架构，研究人员和开发者可以轻松修改、扩展和分析，是研究 AI Agent 架构、进行消融实验和开发新型 Agent 能力的理想平台。

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI 入口 (cli.py)                      │
│              trae-cli run / interactive / show-config         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent 工厂 (Agent)                       │
│         根据 agent_type 实例化对应的 Agent 实现               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    TraeAgent (继承 BaseAgent)                │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  LLM Client  │  │ ToolExecutor │  │ TrajectoryRecorder│ │
│  │  (多 Provider)│  │ (本地/Docker) │  │   (轨迹记录)       │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────────────┘ │
│         │                 │                                  │
│         ▼                 ▼                                  │
│  ┌─────────────┐  ┌──────────────────────────────────────┐  │
│  │  LLM API     │  │           工具注册表                  │  │
│  │  (7种Provider)│  │  bash / edit / json_edit /           │  │
│  │              │  │  sequentialthinking / task_done /    │  │
│  │              │  │  ckg / mcp_tool                       │  │
│  └─────────────┘  └──────────────────────────────────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              CLI Console (Simple / Rich)                │ │
│  │              + Lakeview 摘要生成                         │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 架构设计要点

1. **工厂模式**: `Agent` 类根据 `AgentType` 枚举创建对应的 Agent 实例
2. **模板方法模式**: `BaseAgent` 定义执行骨架，`TraeAgent` 覆写关键方法
3. **策略模式**: `LLMClient` 根据配置动态选择具体的 LLM 客户端实现
4. **注册表模式**: `tools_registry` 字典统一管理所有可用工具
5. **代理模式**: `DockerToolExecutor` 代理工具调用，按需在本地或 Docker 中执行

---

## 3. 项目目录结构

```
trae-agent/
├── trae_agent/                    # 主包
│   ├── __init__.py
│   ├── cli.py                     # CLI 入口 (click 命令组)
│   ├── agent/                     # Agent 核心逻辑
│   │   ├── __init__.py            # 导出 BaseAgent, TraeAgent, Agent
│   │   ├── agent.py               # Agent 工厂类 & AgentType 枚举
│   │   ├── agent_basics.py        # Agent 执行数据结构 & 状态枚举
│   │   ├── base_agent.py          # 抽象基类 BaseAgent (执行骨架)
│   │   ├── trae_agent.py          # TraeAgent 具体实现
│   │   └── docker_manager.py      # Docker 容器生命周期管理
│   ├── tools/                     # 工具生态
│   │   ├── __init__.py            # tools_registry 注册表
│   │   ├── base.py                # Tool 抽象基类 & ToolExecutor
│   │   ├── bash_tool.py           # Bash 命令执行工具
│   │   ├── edit_tool.py           # 文本编辑工具 (view/create/str_replace/insert)
│   │   ├── edit_tool_cli.py       # 编辑工具的 CLI 入口 (Docker 模式)
│   │   ├── json_edit_tool.py      # JSONPath 编辑工具
│   │   ├── json_edit_tool_cli.py  # JSON 编辑工具的 CLI 入口
│   │   ├── sequential_thinking_tool.py  # 顺序思考工具
│   │   ├── task_done_tool.py      # 任务完成信号工具
│   │   ├── mcp_tool.py            # MCP 协议工具适配器
│   │   ├── ckg_tool.py            # 代码知识图谱查询工具
│   │   ├── docker_tool_executor.py # Docker 工具执行代理
│   │   ├── run.py                 # 通用异步 Shell 执行
│   │   └── ckg/                   # 代码知识图谱
│   │       ├── base.py            # CKG 数据模型
│   │       └── ckg_database.py    # CKG SQLite 数据库
│   ├── utils/                    # 工具与基础设施
│   │   ├── config.py             # 配置模型 (YAML/JSON)
│   │   ├── constants.py         # 常量定义
│   │   ├── legacy_config.py     # 旧版 JSON 配置兼容
│   │   ├── lake_view.py         # Lakeview 摘要生成
│   │   ├── mcp_client.py        # MCP 客户端
│   │   ├── trajectory_recorder.py # 轨迹记录器
│   │   ├── cli/                 # CLI 控制台
│   │   │   ├── cli_console.py   # 控制台抽象基类
│   │   │   ├── console_factory.py # 控制台工厂
│   │   │   ├── simple_console.py # 简单文本控制台
│   │   │   ├── rich_console.py   # Rich TUI 控制台
│   │   │   └── rich_console.tcss # Rich 控制台样式
│   │   └── llm_clients/         # LLM 客户端
│   │       ├── base_client.py    # 抽象基类 BaseLLMClient
│   │       ├── llm_client.py    # 客户端工厂 LLMClient
│   │       ├── llm_basics.py    # 消息/响应数据结构
│   │       ├── retry_utils.py   # API 重试装饰器
│   │       ├── anthropic_client.py   # Anthropic Claude
│   │       ├── openai_client.py      # OpenAI GPT
│   │       ├── openai_compatible_base.py # OpenAI 兼容基类
│   │       ├── google_client.py      # Google Gemini
│   │       ├── doubao_client.py      # 字节豆包
│   │       ├── azure_client.py       # Azure OpenAI
│   │       ├── openrouter_client.py  # OpenRouter
│   │       └── ollama_client.py      # Ollama 本地模型
│   └── prompt/
│       └── agent_prompt.py      # Agent 系统提示词
├── evaluation/                   # 评估框架
│   ├── run_evaluation.py         # Benchmark 评估入口
│   ├── utils.py                  # 评估配置 & 工具
│   ├── setup.sh                  # 评估环境安装
│   └── patch_selection/          # Patch 选择评估
│       ├── analysis.py           # 结果分析
│       ├── selector.py           # 选择器入口
│       └── trae_selector/        # Trae 选择器实现
│           ├── selector_agent.py  # 选择 Agent
│           ├── selector_evaluation.py # 评估逻辑
│           ├── sandbox.py         # Docker 沙箱
│           ├── utils.py           # 工具函数
│           └── tools/             # 选择器专用工具
├── tests/                        # 测试套件
│   ├── agent/
│   ├── tools/
│   └── utils/
├── docs/                         # 文档
│   ├── tools.md
│   ├── roadmap.md
│   ├── legacy_config.md
│   └── TRAJECTORY_RECORDING.md
├── server/                       # 服务端 (预留)
├── pyproject.toml               # 项目配置 & 依赖
├── Makefile                     # 构建/测试命令
├── trae_config.yaml.example     # YAML 配置示例
├── trae_config.json.example     # JSON 配置示例 (旧版)
└── uv.lock                      # UV 锁文件
```

---

## 4. 核心模块职责

### 4.1 CLI 入口层 (cli.py)

**文件**: `trae_agent/cli.py`

CLI 入口基于 [Click](https://click.palletsprojects.com/) 框架，提供三个核心子命令：

| 命令 | 说明 |
|------|------|
| `trae-cli run` | 执行单次任务 |
| `trae-cli interactive` | 启动交互式会话 |
| `trae-cli show-config` | 显示当前配置 |
| `trae-cli tools` | 列出可用工具 |

**关键函数**:

```python
def resolve_config_file(config_file: str) -> str
    # 配置文件解析，YAML 优先，回退到 JSON

def check_docker(timeout=3) -> dict
    # 检查 Docker CLI 和 daemon 是否可用

def build_with_pyinstaller()
    # 使用 PyInstaller 构建 Docker 模式所需的工具二进制

def run(task, file_path, provider, model, ...) 
    # 主执行函数：加载配置 → 创建 Agent → 执行任务 → 保存轨迹

def interactive(provider, model, ...)
    # 交互模式入口

async def _run_simple_interactive_loop(agent, cli_console, ...)
    # Simple 控制台的交互循环

async def _run_rich_interactive_loop(agent, cli_console, ...)
    # Rich 控制台的交互循环 (Textual TUI)
```

**执行流程**:
1. 解析命令行参数与配置文件
2. 若启用 Docker 模式，检查 Docker 环境并构建工具二进制
3. 通过 `Config.create()` 加载配置
4. 通过 `ConsoleFactory.create_console()` 创建控制台
5. 实例化 `Agent`，设置工作目录
6. `asyncio.run(agent.run(task, task_args))` 异步执行
7. 输出轨迹文件路径

---

### 4.2 Agent 层 (agent/)

#### 4.2.1 Agent (agent.py) — 工厂入口

```python
class AgentType(Enum):
    TraeAgent = "trae_agent"

class Agent:
    def __init__(self, agent_type, config, trajectory_file, cli_console, 
                 docker_config, docker_keep)
    async def run(self, task, extra_args, tool_names) -> AgentExecution
```

**职责**: 根据传入的 `AgentType` 枚举实例化对应的 Agent 实现（目前仅支持 `TraeAgent`），设置轨迹记录器和 CLI 控制台，然后委托执行。

**`run()` 流程**:
1. 调用 `agent.new_task()` 创建新任务
2. 初始化 MCP 工具（如果配置了 MCP 服务）
3. 打印任务详情
4. 启动 CLI 控制台（异步任务）
5. `await agent.execute_task()` 执行任务
6. 清理 MCP 客户端
7. 返回 `AgentExecution` 结果

#### 4.2.2 BaseAgent (base_agent.py) — 执行骨架

```python
class BaseAgent(ABC):
    _tool_caller: Union[ToolExecutor, DockerToolExecutor]
    
    def __init__(self, agent_config, docker_config, docker_keep)
    # 初始化 LLM 客户端、加载工具、配置 Docker/本地执行器、清理旧 CKG
    
    @abstractmethod
    def new_task(self, task, extra_args, tool_names)
    # 抽象方法：创建新任务
    
    async def execute_task(self) -> AgentExecution
    # 核心执行循环
    
    async def _run_llm_step(self, step, messages, execution) -> list[LLMMessage]
    # 单步 LLM 交互：调用 LLM → 判断完成 → 处理工具调用
    
    async def _tool_call_handler(self, tool_calls, step) -> list[LLMMessage]
    # 工具调用处理：顺序/并行执行 → 生成结果消息 → 反思
    
    async def _finalize_step(self, step, messages, execution)
    # 完成步骤：记录轨迹 → 更新控制台
    
    def llm_indicates_task_completed(self, llm_response) -> bool
    # 判断 LLM 是否表示任务完成（默认通过关键词检测）
    
    def reflect_on_result(self, tool_results) -> str | None
    # 对工具执行结果进行反思
    
    @abstractmethod
    async def cleanup_mcp_clients(self)
    # 抽象方法：清理 MCP 客户端
```

**核心执行循环 (`execute_task`)**:
```
while step_number <= max_steps:
    1. 创建 AgentStep
    2. _run_llm_step(): 调用 LLM，获取响应
       - 若 LLM 表示完成 → 检查是否真正完成 → 标记 COMPLETED
       - 否则 → _tool_call_handler(): 执行工具调用
    3. _finalize_step(): 记录轨迹、更新控制台
    4. 若 COMPLETED 则退出循环
```

#### 4.2.3 TraeAgent (trae_agent.py) — 具体实现

```python
TraeAgentToolNames = [
    "str_replace_based_edit_tool",
    "sequentialthinking",
    "json_edit_tool",
    "task_done",
    "bash",
]

class TraeAgent(BaseAgent):
    def __init__(self, trae_agent_config, docker_config, docker_keep)
    # 初始化项目路径、MCP 配置等特有状态
    
    async def initialise_mcp(self)
    # 发现并加载 MCP 工具
    
    async def discover_mcp_tools(self)
    # 遍历 MCP 服务器配置，连接并发现工具
    
    @override
    def new_task(self, task, extra_args, tool_names)
    # 设置任务、系统提示词、用户消息、轨迹记录
    
    @override
    async def execute_task(self) -> AgentExecution
    # 执行后完成轨迹记录，可选写入 patch 文件
    
    def get_git_diff(self) -> str
    # 获取项目 git diff
    
    def remove_patches_to_tests(self, model_patch) -> str
    # 从 patch 中移除测试目录的修改（源自 Aider，Apache-2.0）
    
    @override
    def llm_indicates_task_completed(self, llm_response) -> bool
    # 通过检测 task_done 工具调用判断完成
    
    @override
    def _is_task_completed(self, llm_response) -> bool
    # 增强完成检测：must_patch 模式下检查是否有非空 patch
    
    @override
    async def cleanup_mcp_clients(self)
    # 清理所有 MCP 客户端连接
```

#### 4.2.4 Agent 数据结构 (agent_basics.py)

```python
class AgentStepState(Enum):
    THINKING = "thinking"
    CALLING_TOOL = "calling_tool"
    REFLECTING = "reflecting"
    COMPLETED = "completed"
    ERROR = "error"

class AgentState(Enum):
    IDLE = "idle"
    RUNNING = "running"
    COMPLETED = "completed"
    ERROR = "error"

@dataclass
class AgentStep:
    step_number: int
    state: AgentStepState
    thought: str | None
    tool_calls: list[ToolCall] | None
    tool_results: list[ToolResult] | None
    llm_response: LLMResponse | None
    reflection: str | None
    error: str | None
    extra: dict[str, object] | None
    llm_usage: LLMUsage | None

@dataclass
class AgentExecution:
    task: str
    steps: list[AgentStep]
    final_result: str | None
    success: bool
    total_tokens: LLMUsage | None
    execution_time: float
    agent_state: AgentState

class AgentError(Exception):
    # Agent 相关错误
```

---

### 4.3 工具层 (tools/)

#### 工具注册表

```python
# trae_agent/tools/__init__.py
tools_registry: dict[str, type[Tool]] = {
    "bash": BashTool,
    "str_replace_based_edit_tool": TextEditorTool,
    "json_edit_tool": JSONEditTool,
    "sequentialthinking": SequentialThinkingTool,
    "task_done": TaskDoneTool,
    "ckg": CKGTool,
}
```

#### 4.3.1 工具基类 (base.py)

```python
class Tool(ABC):
    @cached_property
    def name(self) -> str           # 工具名称
    @cached_property
    def description(self) -> str   # 工具描述
    @cached_property
    def parameters(self) -> list[ToolParameter]  # 参数定义
    
    @abstractmethod
    def get_name(self) -> str
    @abstractmethod
    def get_description(self) -> str
    @abstractmethod
    def get_parameters(self) -> list[ToolParameter]
    @abstractmethod
    async def execute(self, arguments) -> ToolExecResult
    
    def get_input_schema(self) -> dict[str, object]
    # 生成 JSON Schema（适配不同 Provider，OpenAI strict mode）
    
    def json_definition(self) -> dict[str, object]
    # 生成完整的工具定义

class ToolExecutor:
    def __init__(self, tools: list[Tool])
    async def execute_tool_call(self, tool_call: ToolCall) -> ToolResult
    async def parallel_tool_call(self, tool_calls) -> list[ToolResult]
    async def sequential_tool_call(self, tool_calls) -> list[ToolResult]
    async def close_tools(self)  # 释放工具资源
```

**数据结构**:

| 类 | 说明 |
|---|---|
| `ToolError` | 工具错误异常 |
| `ToolExecResult` | 工具执行中间结果 (output, error, error_code) |
| `ToolResult` | 工具执行最终结果 (call_id, name, success, result, error, id) |
| `ToolCall` | 解析后的工具调用 (name, call_id, arguments, id) |
| `ToolParameter` | 工具参数定义 (name, type, description, enum, items, required) |

#### 4.3.2 内置工具一览

| 工具名 | 类 | 文件 | 功能 |
|--------|-----|------|------|
| `bash` | `BashTool` | `bash_tool.py` | 在持久化 Bash 会话中执行命令，120秒超时，支持重启 |
| `str_replace_based_edit_tool` | `TextEditorTool` | `edit_tool.py` | 文件编辑：view / create / str_replace / insert |
| `json_edit_tool` | `JSONEditTool` | `json_edit_tool.py` | JSONPath 编辑：view / set / add / remove |
| `sequentialthinking` | `SequentialThinkingTool` | `sequential_thinking_tool.py` | 结构化逐步思考，支持修订与分支 |
| `task_done` | `TaskDoneTool` | `task_done_tool.py` | 任务完成信号（无参数） |
| `ckg` | `CKGTool` | `ckg_tool.py` | 代码知识图谱查询：search_function / search_class / search_class_method |
| MCP 工具 | `MCPTool` | `mcp_tool.py` | MCP 协议工具适配器，将 MCP tool schema 映射为 ToolParameter |

#### 4.3.3 Docker 工具执行器 (docker_tool_executor.py)

```python
class DockerToolExecutor:
    def __init__(self, original_executor, docker_manager, docker_tools, 
                 host_workspace_dir, container_workspace_dir)
    
    def _translate_path(self, host_path) -> str
    # 将宿主机路径翻译为容器内路径
    
    async def sequential_tool_call(self, tool_calls) -> list[ToolResult]
    # 顺序执行，按工具名路由到 Docker 或本地
    
    async def parallel_tool_call(self, tool_calls) -> list[ToolResult]
    # Docker 模式下退化为顺序执行
    
    def _execute_in_docker(self, tool_call) -> ToolResult
    # 在 Docker 中执行：构建命令 → 路径翻译 → Docker 执行
    # 支持 bash, str_replace_based_edit_tool, json_edit_tool
```

#### 4.3.4 代码知识图谱 (ckg/)

**数据模型** (`ckg/base.py`):

```python
@dataclass
class FunctionEntry:
    name: str
    file_path: str
    start_line: int
    end_line: int
    parameters: list[str]
    parent_class: str | None  # 所属类名（方法时）

@dataclass
class ClassEntry:
    name: str
    file_path: str
    start_line: int
    end_line: int
    methods: list[str]

extension_to_language: dict[str, str]  # 文件扩展名 → tree-sitter 语言名
```

**数据库** (`ckg/ckg_database.py`):

```python
CKG_DATABASE_PATH = LOCAL_STORAGE_PATH / "ckg"
CKG_DATABASE_EXPIRY_TIME = 60 * 60 * 24 * 7  # 1 周

class CKGDatabase:
    def __init__(self, codebase_path: str | Path)
    # 根据 codebase snapshot hash 决定复用或重建 SQLite 数据库
    # 使用 tree-sitter 解析 AST，提取函数/类/方法信息
    
def clear_older_ckg()
    # 清理过期的 CKG 数据库文件
```

**工作原理**:
1. 计算 codebase 的 git status hash
2. 若对应 hash 的 `.db` 文件存在且未过期 → 复用
3. 否则 → 使用 tree-sitter 解析所有源文件 AST
4. 遍历 AST 提取 `function_definition` 和 `class_definition`
5. 将 `FunctionEntry` / `ClassEntry` 写入 SQLite

---

### 4.4 LLM 客户端层 (utils/llm_clients/)

#### 4.4.1 数据结构 (llm_basics.py)

```python
@dataclass
class LLMMessage:
    role: str                          # "system" | "user" | "assistant"
    content: str | None
    tool_call: ToolCall | None        # 工具调用
    tool_result: ToolResult | None    # 工具结果

@dataclass
class LLMUsage:
    input_tokens: int
    output_tokens: int
    cache_creation_input_tokens: int = 0
    cache_read_input_tokens: int = 0
    reasoning_tokens: int = 0
    # 支持 __add__ 运算符累加

@dataclass
class LLMResponse:
    content: str
    usage: LLMUsage | None
    model: str | None
    finish_reason: str | None
    tool_calls: list[ToolCall] | None
```

#### 4.4.2 客户端工厂 (llm_client.py)

```python
class LLMProvider(Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    AZURE = "azure"
    OLLAMA = "ollama"
    OPENROUTER = "openrouter"
    DOUBAO = "doubao"
    GOOGLE = "google"

class LLMClient:
    def __init__(self, model_config: ModelConfig)
    # match provider → 实例化对应客户端
    
    def chat(self, messages, model_config, tools, reuse_history) -> LLMResponse
    def set_trajectory_recorder(self, recorder)
    def set_chat_history(self, messages)
    def supports_tool_calling(self, model_config) -> bool
```

#### 4.4.3 抽象基类 (base_client.py)

```python
class BaseLLMClient(ABC):
    def __init__(self, model_config: ModelConfig)
    # 初始化 api_key, base_url, api_version, trajectory_recorder
    
    @abstractmethod
    def set_chat_history(self, messages)
    @abstractmethod
    def chat(self, messages, model_config, tools, reuse_history) -> LLMResponse
    def supports_tool_calling(self, model_config) -> bool
```

#### 4.4.4 各 Provider 客户端

| 客户端 | 文件 | 继承关系 | 说明 |
|--------|------|----------|------|
| `OpenAIClient` | `openai_client.py` | `OpenAICompatibleBase` | OpenAI GPT 系列 |
| `AnthropicClient` | `anthropic_client.py` | `BaseLLMClient` | Anthropic Claude 系列 |
| `AzureClient` | `azure_client.py` | `OpenAICompatibleBase` | Azure OpenAI，支持 `max_completion_tokens` |
| `OpenRouterClient` | `openrouter_client.py` | `OpenAICompatibleBase` | OpenRouter 多模型路由 |
| `DoubaoClient` | `doubao_client.py` | `OpenAICompatibleBase` | 字节跳动豆包 (ARK) |
| `GoogleClient` | `google_client.py` | `BaseLLMClient` | Google Gemini |
| `OllamaClient` | `ollama_client.py` | `BaseLLMClient` | Ollama 本地模型 |

**`OpenAICompatibleBase`** (`openai_compatible_base.py`): 为所有 OpenAI 兼容 API 的客户端提供共享的 `chat()` 实现，统一处理消息格式转换、工具 schema 转换、响应解析等逻辑。

#### 4.4.5 重试工具 (retry_utils.py)

```python
def retry_api(max_retries: int, base_delay: float)
# 装饰器：实现随机退避重试逻辑，用于 LLM API 调用
```

---

### 4.5 配置管理 (utils/config.py)

#### 配置数据模型

```python
@dataclass
class ModelProvider:
    api_key: str
    provider: str           # "openai" | "anthropic" | "azure" | ...
    base_url: str | None
    api_version: str | None  # Azure 必需

@dataclass
class ModelConfig:
    model: str
    model_provider: ModelProvider
    temperature: float
    top_p: float
    top_k: int
    parallel_tool_calls: bool
    max_retries: int
    max_tokens: int | None
    supports_tool_calling: bool = True
    candidate_count: int | None  # Gemini 专属
    stop_sequences: list[str] | None
    max_completion_tokens: int | None  # Azure gpt-5/o3/o4-mini 专属
    
    def resolve_config_values(...)  # CLI/ENV 覆盖

@dataclass
class MCPServerConfig:
    command: str | None       # stdio 传输
    args: list[str] | None
    env: dict[str, str] | None
    cwd: str | None
    url: str | None           # SSE 传输
    http_url: str | None      # HTTP 传输
    headers: dict[str, str] | None
    tcp: str | None           # WebSocket 传输
    timeout: int | None
    trust: bool | None
    description: str | None

@dataclass
class AgentConfig:
    allow_mcp_servers: list[str]
    mcp_servers_config: dict[str, MCPServerConfig]
    max_steps: int
    model: ModelConfig
    tools: list[str]

@dataclass
class TraeAgentConfig(AgentConfig):
    enable_lakeview: bool = True
    tools: list[str] = ["bash", "str_replace_based_edit_tool", 
                        "sequentialthinking", "task_done"]

@dataclass
class LakeviewConfig:
    model: ModelConfig

@dataclass
class Config:
    lakeview: LakeviewConfig | None
    model_providers: dict[str, ModelProvider] | None
    models: dict[str, ModelConfig] | None
    trae_agent: TraeAgentConfig | None
    
    @classmethod
    def create(cls, config_file, config_string) -> "Config"
    # 从 YAML 或 JSON 字符串解析配置
    
    def resolve_config_values(self, provider, model, ...)
    # 应用 CLI/ENV 覆盖
    
    @classmethod
    def create_from_legacy_config(cls, ...) -> "Config"
    # 从旧版 JSON 配置转换
```

#### 配置优先级

```
命令行参数 > 配置文件 > 环境变量 > 默认值
```

#### 环境变量映射

| 环境变量 | 说明 |
|----------|------|
| `OPENAI_API_KEY` / `OPENAI_BASE_URL` | OpenAI |
| `ANTHROPIC_API_KEY` / `ANTHROPIC_BASE_URL` | Anthropic |
| `GOOGLE_API_KEY` / `GOOGLE_BASE_URL` | Google |
| `OPENROUTER_API_KEY` / `OPENROUTER_BASE_URL` | OpenRouter |
| `DOUBAO_API_KEY` / `DOUBAO_BASE_URL` | Doubao |
| `TRAE_CONFIG_FILE` | 配置文件路径 |

---

### 4.6 轨迹记录 (utils/trajectory_recorder.py)

```python
class TrajectoryRecorder:
    def __init__(self, trajectory_path: str | None = None)
    # 默认路径: trajectories/trajectory_YYYYMMDD_HHMMSS.json
    
    def start_recording(self, task, provider, model, max_steps)
    # 开始记录
    
    def record_llm_interaction(self, messages, response, provider, model, tools)
    # 记录每次 LLM 交互
    
    def record_agent_step(self, step_number, state, llm_messages, 
                          llm_response, tool_calls, tool_results, 
                          reflection, error)
    # 记录每个 Agent 步骤
    
    def update_lakeview(self, step_number, lakeview_summary)
    # 更新步骤的 Lakeview 摘要
    
    def finalize_recording(self, success, final_result)
    # 完成记录，写入文件
    
    def save_trajectory(self)
    # 保存到 JSON 文件
```

**轨迹 JSON 结构**:
```json
{
  "task": "",
  "start_time": "",
  "end_time": "",
  "provider": "",
  "model": "",
  "max_steps": 0,
  "llm_interactions": [],
  "agent_steps": [],
  "success": false,
  "final_result": null,
  "execution_time": 0.0
}
```

---

### 4.7 Lakeview 摘要 (utils/lake_view.py)

```python
@dataclass
class LakeViewStep:
    desc_task: str       # 简洁任务描述
    desc_details: str    # bug 相关细节
    tags_emoji: str      # 标签 emoji 字符串

class LakeView:
    def __init__(self, lake_view_config: LakeviewConfig | None)
    # 初始化 Lakeview 专用 LLM 客户端
    
    async def extract_task_in_step(self, prev_step, this_step) -> tuple[str, str]
    # 使用 EXTRACTOR_PROMPT 提取当前步骤的任务描述
    
    async def extract_tag_in_step(self, step) -> list[str]
    # 使用 TAGGER_PROMPT 为步骤打标签
    
    async def create_lakeview_step(self, agent_step) -> LakeViewStep | None
    # 生成完整的 Lakeview 步骤摘要
```

**标签体系**:

| 标签 | Emoji | 含义 |
|------|-------|------|
| `WRITE_TEST` | ☑️ | 编写/修改复现脚本 |
| `VERIFY_TEST` | ✅ | 运行复现脚本验证环境 |
| `EXAMINE_CODE` | 👁️ | 查看/搜索代码 |
| `WRITE_FIX` | 📝 | 修改源代码修复 bug |
| `VERIFY_FIX` | 🔥 | 运行测试验证修复 |
| `REPORT` | 📣 | 报告完成或进展 |
| `THINK` | 🧠 | 分析思考但无具体动作 |
| `OUTLIER` | ⁉️ | 不属于上述任何类别 |

---

### 4.8 MCP 客户端 (utils/mcp_client.py)

```python
class MCPServerStatus(Enum):
    DISCONNECTED = "disconnected"
    CONNECTING = "connecting"
    CONNECTED = "connected"

class MCPClient:
    def __init__(self)
    # 初始化 AsyncExitStack 和状态字典
    
    async def connect_and_discover(self, mcp_server_name, mcp_server_config, 
                                    mcp_tools_container, model_provider)
    # 连接 MCP 服务器 → 列出工具 → 创建 MCPTool 实例
    
    async def connect_to_server(self, mcp_server_name, transport)
    # 通过 stdio 传输连接到 MCP 服务器
    
    async def call_tool(self, name, args)
    # 调用 MCP 工具
    
    async def list_tools(self)
    # 列出 MCP 服务器提供的工具
    
    async def cleanup(self, mcp_server_name)
    # 关闭连接，清理资源
```

**支持的传输协议**:
- ✅ **stdio** — 通过子进程标准输入/输出通信
- ⏳ **HTTP** — 尚未实现 (`NotImplementedError`)
- ⏳ **WebSocket/SSE** — 尚未实现 (`NotImplementedError`)

---

### 4.9 CLI 控制台 (utils/cli/)

#### 抽象基类 (cli_console.py)

```python
class ConsoleMode(Enum):
    RUN = "run"           # 执行单次任务
    INTERACTIVE = "interactive"  # 多轮交互

class ConsoleType(Enum):
    SIMPLE = "simple"    # 简单文本控制台
    RICH = "rich"        # Rich TUI 控制台

@dataclass
class ConsoleStep:
    agent_step: AgentStep
    agent_step_printed: bool
    lake_view_panel_generator: asyncio.Task | None

class CLIConsole(ABC):
    def __init__(self, mode, lakeview_config)
    
    @abstractmethod
    async def start(self)
    @abstractmethod
    def update_status(self, agent_step, agent_execution)
    @abstractmethod
    def print_task_details(self, details)
    @abstractmethod
    def print(self, message, color, bold)
    @abstractmethod
    def get_task_input(self) -> str | None
    @abstractmethod
    def get_working_dir_input(self) -> str
    @abstractmethod
    def stop(self)
    
    def set_lakeview(self, lakeview_config)
```

#### 控制台工厂 (console_factory.py)

```python
class ConsoleFactory:
    @staticmethod
    def create_console(console_type, mode, lakeview_config) -> CLIConsole
    # SIMPLE → SimpleCLIConsole
    # RICH → RichCLIConsole
    
    @staticmethod
    def get_recommended_console_type(mode) -> ConsoleType
    # INTERACTIVE → RICH
    # RUN → SIMPLE
```

#### 实现类

| 类 | 文件 | 说明 |
|----|------|------|
| `SimpleCLIConsole` | `simple_console.py` | 基于 Rich 的简单文本输出，适合 `run` 模式 |
| `RichCLIConsole` | `rich_console.py` | 基于 Textual 的 TUI 应用，适合 `interactive` 模式 |

---

### 4.10 Prompt 模块 (prompt/agent_prompt.py)

```python
TRAE_AGENT_SYSTEM_PROMPT = """You are an expert AI software engineering agent.
...
"""
```

**系统提示词指导 Agent 执行以下步骤**:

1. **理解问题** — 仔细阅读问题描述
2. **探索定位** — 使用工具浏览代码库，定位相关文件
3. **复现 Bug** — 创建可复现的脚本或测试用例
4. **调试诊断** — 检查代码，追踪执行流程
5. **实施修复** — 精准、最小化的代码修改
6. **验证测试** — 运行复现脚本 + 现有测试 + 新测试
7. **总结** — 清晰说明 bug 原因和修复逻辑

**关键规则**:
- 所有 `file_path` 参数必须使用**绝对路径**
- 鼓励使用 `sequential_thinking` 工具进行深度分析
- 完成后调用 `task_done` 工具

---

### 4.11 Docker 管理 (agent/docker_manager.py)

```python
class DockerManager:
    CONTAINER_TOOLS_PATH = "/agent_tools"
    
    def __init__(self, image, container_id, dockerfile_path, 
                 docker_image_file, workspace_dir, tools_dir, interactive)
    
    def start(self)
    # 构建/加载镜像 → 创建/附加容器 → 挂载工作区 → 复制工具 → 启动持久 Shell
    
    def execute(self, command, timeout=300) -> tuple[int, str]
    # 在持久化 bash shell 中执行命令
    
    def stop(self)
    # 关闭 Shell → 停止并移除管理的容器
```

**Docker 模式启动方式**:

| 参数 | 说明 |
|------|------|
| `--docker-image` | 使用指定镜像创建新容器 |
| `--docker-container-id` | 附加到已存在的容器 |
| `--dockerfile-path` | 从 Dockerfile 构建镜像 |
| `--docker-image-file` | 从 tar 归档加载镜像 |
| `--docker-keep` | 任务完成后是否保留容器 |

**工作原理**:
1. 创建容器时挂载 `workspace_dir` → `/workspace`
2. 使用 `docker cp` 将打包好的工具二进制复制到 `/agent_tools`
3. 通过 `pexpect` 启动持久化 bash shell (`docker exec -it`)
4. 命令执行使用 marker 机制捕获退出码
5. 路径翻译：宿主机路径 ↔ 容器内路径

---

## 5. 关键类与函数说明

### 5.1 核心类继承关系

```
ABC
├── BaseAgent (agent/base_agent.py)
│   └── TraeAgent (agent/trae_agent.py)
├── Tool (tools/base.py)
│   ├── BashTool (tools/bash_tool.py)
│   ├── TextEditorTool (tools/edit_tool.py)
│   ├── JSONEditTool (tools/json_edit_tool.py)
│   ├── SequentialThinkingTool (tools/sequential_thinking_tool.py)
│   ├── TaskDoneTool (tools/task_done_tool.py)
│   ├── CKGTool (tools/ckg_tool.py)
│   └── MCPTool (tools/mcp_tool.py)
├── BaseLLMClient (utils/llm_clients/base_client.py)
│   ├── AnthropicClient
│   ├── GoogleClient
│   ├── OllamaClient
│   └── OpenAICompatibleBase
│       ├── OpenAIClient
│       ├── AzureClient
│       ├── OpenRouterClient
│       └── DoubaoClient
└── CLIConsole (utils/cli/cli_console.py)
    ├── SimpleCLIConsole
    └── RichCLIConsole
```

### 5.2 关键函数索引

| 函数 | 文件 | 说明 |
|------|------|------|
| `main()` | `cli.py` | CLI 入口，调用 `cli()` |
| `run()` | `cli.py` | `trae-cli run` 命令处理 |
| `interactive()` | `cli.py` | `trae-cli interactive` 命令处理 |
| `show_config()` | `cli.py` | `trae-cli show-config` 命令处理 |
| `resolve_config_value()` | `utils/config.py` | 配置值优先级解析 |
| `get_git_status_hash()` | `tools/ckg/ckg_database.py` | 获取 codebase 的 git 状态 hash |
| `clear_older_ckg()` | `tools/ckg/ckg_database.py` | 清理过期 CKG 数据库 |
| `run()` | `tools/run.py` | 通用异步 Shell 命令执行 |

---

## 6. 依赖关系

### 6.1 核心依赖 (pyproject.toml)

| 依赖 | 版本要求 | 用途 |
|------|----------|------|
| `openai` | >=1.86.0 | OpenAI/Azure/OpenRouter/Doubao API |
| `anthropic` | >=0.54.0, <=0.60.0 | Anthropic Claude API |
| `google-genai` | >=1.24.0 | Google Gemini API |
| `ollama` | >=0.5.1 | Ollama 本地模型 |
| `click` | >=8.0.0 | CLI 框架 |
| `asyncclick` | >=8.0.0 | 异步 CLI 扩展 |
| `pydantic` | >=2.0.0 | 数据验证 |
| `rich` | >=13.0.0 | 终端富文本输出 |
| `textual` | >=0.50.0 | TUI 应用框架 |
| `pyyaml` | >=6.0.2 | YAML 配置解析 |
| `python-dotenv` | >=1.0.0 | .env 文件加载 |
| `jsonpath-ng` | >=1.7.0 | JSONPath 编辑工具 |
| `tree-sitter` | ==0.21.3 | AST 解析 (CKG) |
| `tree-sitter-languages` | ==1.10.2 | 多语言 tree-sitter 支持 |
| `mcp` | ==1.12.2 | Model Context Protocol |
| `socksio` | >=1.0.0 | SOCKS 代理支持 |
| `ruff` | >=0.12.4 | 代码格式化/检查 |
| `pyinstaller` | ==6.15.0 | Docker 模式工具打包 |

### 6.2 可选依赖

| 分组 | 依赖 | 用途 |
|------|------|------|
| `test` | `pytest`, `pytest-asyncio`, `pytest-mock`, `pytest-cov`, `pre-commit` | 测试 |
| `evaluation` | `datasets`, `docker`, `pexpect`, `unidiff` | 评估 |

### 6.3 模块间依赖关系

```
cli.py
  ├── agent.Agent
  ├── utils.config.Config
  └── utils.cli.ConsoleFactory

agent.Agent / TraeAgent
  ├── agent.base_agent.BaseAgent
  ├── utils.llm_clients.LLMClient
  ├── tools.tools_registry → Tool 实例
  ├── utils.trajectory_recorder.TrajectoryRecorder
  ├── utils.mcp_client.MCPClient
  ├── agent.docker_manager.DockerManager
  └── prompt.agent_prompt.TRAE_AGENT_SYSTEM_PROMPT

BaseAgent
  ├── tools.base.ToolExecutor / DockerToolExecutor
  ├── utils.llm_clients.LLMClient → BaseLLMClient 子类
  └── agent.agent_basics (数据结构)

ToolExecutor
  └── tools.base.Tool 子类
      ├── BashTool → 内部 _BashSession
      ├── TextEditorTool → tools.run.run()
      ├── JSONEditTool → jsonpath_ng
      ├── CKGTool → tools.ckg.ckg_database.CKGDatabase
      └── MCPTool → utils.mcp_client.MCPClient
```

---

## 7. 评估系统 (evaluation/)

### 7.1 Benchmark 评估 (run_evaluation.py)

```python
class BenchmarkEvaluation:
    def __init__(self, benchmark, dataset, ...)
    # 支持 SWE-bench, SWE-bench-Live, Multi-SWE-bench
    
    def run_one_instance(self, instance) -> dict
    # 为单个实例准备 Docker 容器 → 运行 trae-cli → 生成 patch
    
    def run_all(self) -> list[dict]
    # 使用 ThreadPoolExecutor 并发运行所有实例
    
    def run_eval(self) -> dict
    # 执行评估 → 保存 results.json
```

**支持的 Benchmark**:

| Benchmark | 说明 |
|-----------|------|
| SWE-bench | 标准 SWE-bench 评估 |
| SWE-bench-Live | 实时 SWE-bench 评估 |
| Multi-SWE-bench | 多语言 SWE-bench 评估 |

### 7.2 Patch 选择评估 (patch_selection/)

用于评估 Agent 从多个候选 patch 中选择正确修复的能力。

```python
class SelectorEvaluation:
    def run_all(self)
    # 使用 ProcessPoolExecutor 批量提交实例
    
    def run_one(self, instance_id)
    # 执行单实例选择评估
    
def run_instance(instance, ...)
    # 将候选 patch 分组 → 逐组评估
    
def run_instance_by_group(instance, group, ...)
    # 清理 patch → 启动沙箱 → 调用 SelectorAgent → 保存结果
```

**SelectorAgent** (`selector_agent.py`):

```python
class SelectorAgent:
    def __init__(self, model_config, sandbox, ...)
    # 初始化 LLM 客户端、沙箱、工具
    
    async def run(self, candidate_patches, ...) -> str
    # 构造提示 → 循环调用 LLM → 解析选择结果 → 记录最终 patch

@dataclass
class CandidatePatch:
    # 封装候选 patch 的元信息
```

**Sandbox** (`sandbox.py`):

```python
class Sandbox:
    def __init__(self, image, working_dir, ...)
    # 创建 Docker 容器沙箱
    
    def execute(self, command, timeout) -> str
    # 在沙箱中执行命令
    
    def close(self)
    # 关闭并清理沙箱
```

---

## 8. 项目运行方式

### 8.1 环境准备

```bash
# 安装 UV
# 参考: https://docs.astral.sh/uv/

# 克隆项目
git clone https://github.com/bytedance/trae-agent.git
cd trae-agent

# 安装依赖
uv sync --all-extras
source .venv/bin/activate
```

### 8.2 配置

#### YAML 配置 (推荐)

```bash
cp trae_config.yaml.example trae_config.yaml
```

编辑 `trae_config.yaml`:

```yaml
agents:
  trae_agent:
    enable_lakeview: true
    model: trae_agent_model
    max_steps: 200
    tools:
      - bash
      - str_replace_based_edit_tool
      - sequentialthinking
      - task_done

model_providers:
  anthropic:
    api_key: your_anthropic_api_key
    provider: anthropic

models:
  trae_agent_model:
    model_provider: anthropic
    model: claude-sonnet-4-20250514
    max_tokens: 4096
    temperature: 0.5
    top_p: 1
    top_k: 0
    max_retries: 10
    parallel_tool_calls: true
  lakeview_model:
    model_provider: anthropic
    model: claude-3.5-sonnet
    max_tokens: 4096
    temperature: 0.5
    top_p: 1
    top_k: 0
    max_retries: 10
    parallel_tool_calls: true

lakeview:
  model: lakeview_model

# 可选: MCP 服务
allow_mcp_servers:
  - playwright
mcp_servers:
  playwright:
    command: npx
    args:
      - "@playwright/mcp@0.0.27"
```

#### 环境变量配置 (替代方案)

```bash
export OPENAI_API_KEY="your-openai-api-key"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export GOOGLE_API_KEY="your-google-api-key"
export OPENROUTER_API_KEY="your-openrouter-api-key"
export DOUBAO_API_KEY="your-doubao-api-key"
```

### 8.3 基本使用

```bash
# 执行单次任务
trae-cli run "Create a hello world Python script"

# 指定 Provider 和模型
trae-cli run "Fix the bug in main.py" --provider openai --model gpt-4o
trae-cli run "Add unit tests" --provider anthropic --model claude-sonnet-4-20250514
trae-cli run "Optimize this algorithm" --provider google --model gemini-2.5-flash
trae-cli run "Comment this code" --provider ollama --model qwen3

# 交互模式
trae-cli interactive

# 查看配置
trae-cli show-config

# 从文件读取任务
trae-cli run --file task_description.txt

# 保存执行轨迹
trae-cli run "Debug authentication" --trajectory-file debug_session.json

# 强制生成 patch
trae-cli run "Update API endpoints" --must-patch

# 指定工作目录
trae-cli run "Add tests" --working-dir /path/to/project

# 使用 Rich 控制台
trae-cli run "Your task" --console-type rich
```

### 8.4 Docker 模式

```bash
# 使用指定镜像
trae-cli run "Add tests" --docker-image python:3.11

# 挂载工作目录
trae-cli run "Write a script" --docker-image python:3.12 --working-dir test_workdir/

# 附加到已有容器
trae-cli run "Update API" --docker-container-id 91998a56056c

# 从 Dockerfile 构建
trae-cli run "Debug" --dockerfile-path test_workspace/Dockerfile

# 从 tar 归档加载镜像
trae-cli run "Fix bug" --docker-image-file test_workspace/custom.tar

# 任务完成后移除容器
trae-cli run "Add tests" --docker-image python:3.11 --docker-keep false
```

### 8.5 交互模式命令

| 命令 | 说明 |
|------|------|
| `<输入任务描述>` | 执行任务 |
| `status` | 显示 Agent 状态 |
| `help` | 显示帮助 |
| `clear` | 清屏 |
| `exit` / `quit` | 退出 |

---

## 9. 开发与测试

### 9.1 开发环境搭建

```bash
# 创建虚拟环境并安装所有依赖
make install-dev

# 或手动
uv venv
uv sync --all-extras
```

### 9.2 代码质量

```bash
# 运行 pre-commit
make uv-pre-commit

# 修复格式
make fix-format
# 等同于: ruff format . && ruff check --fix .
```

**Ruff 配置** (pyproject.toml):
- 行宽: 100
- Lint 规则: B (bugbear), SIM (simplify), C4 (comprehensions), E4/E9/E7, F (pyflakes), I (isort)

### 9.3 运行测试

```bash
# 通过 UV 运行测试
make uv-test

# 直接运行
SKIP_OLLAMA_TEST=true SKIP_OPENROUTER_TEST=true SKIP_GOOGLE_TEST=true uv run pytest
```

**测试标记**:
- `@pytest.mark.slow` — 慢测试
- `@pytest.mark.integration` — 集成测试
- `@pytest.mark.unit` — 单元测试

**测试覆盖范围**:

| 目录 | 测试内容 |
|------|----------|
| `tests/agent/` | Agent 核心逻辑测试 |
| `tests/tools/` | 工具测试 (bash, edit, json_edit, mcp) |
| `tests/utils/` | 工具类测试 (config, llm_clients, mcp_client) |
| `tests/test_cli.py` | CLI 入口测试 |

### 9.4 CI/CD

项目使用 GitHub Actions (`.github/workflows/`):

| 工作流 | 文件 | 说明 |
|--------|------|------|
| Pre-commit | `pre-commit.yml` | 代码质量检查 |
| Unit Test | `unit-test.yml` | 单元测试 |

### 9.5 清理

```bash
make clean
# 清理 build/, dist/, *.egg-info/, __pycache__, .pytest_cache 等
```

---

## 10. 路线图

基于 `docs/roadmap.md`，项目未来的发展方向包括：

| 方向 | 关键特性 | 预期收益 |
|------|----------|----------|
| **SDK 开发** | Headless API、流式轨迹记录 | 程序化集成、自动化、研究应用 |
| **沙箱环境** | 隔离执行、并行任务 | 安全性、可复现性、多租户 |
| **轨迹分析** | MLOps 集成 (Wandb/MLFlow) | 性能优化、调试、模型比较 |
| **工具 & MCP** | 结构化文件支持、MCP 标准化 | 生态扩展、互操作性 |
| **多 Agent** | 多 Agent 协作、高级工作流 | 复杂问题求解、专业化 |

---

## 附录

### A. 配置文件示例 (YAML)

参见 [trae_config.yaml.example](file:///workspace/trae-agent/trae_config.yaml.example)

### B. 旧版 JSON 配置

参见 [docs/legacy_config.md](file:///workspace/trae-agent/docs/legacy_config.md)

### C. 工具详细文档

参见 [docs/tools.md](file:///workspace/trae-agent/docs/tools.md)

### D. 轨迹记录文档

参见 [docs/TRAJECTORY_RECORDING.md](file:///workspace/trae-agent/docs/TRAJECTORY_RECORDING.md)

### E. 引用

```bibtex
@article{traeresearchteam2025traeagent,
    title={Trae Agent: An LLM-based Agent for Software Engineering with Test-time Scaling},
    author={Trae Research Team and Pengfei Gao and Zhao Tian and Xiangxin Meng and Xinchen Wang and Ruida Hu and Yuanan Xiao and Yizhou Liu and Zhao Zhang and Junjie Chen and Cuiyun Gao and Yun Lin and Yingfei Xiong and Chao Peng and Xia Liu},
    year={2025},
    eprint={2507.23370},
    archivePrefix={arXiv},
    primaryClass={cs.SE},
    url={https://arxiv.org/abs/2507.23370},
}
```
