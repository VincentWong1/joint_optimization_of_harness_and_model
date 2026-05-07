# selfDeepAgents 框架设计方案

## Context

构建一个两层架构的 AI Agent 系统：**Harness Layer (DeepAgents)** 负责运行时编排，**Optimization Layer (DSPy)** 负责离线寻优。两层通过 Langfuse trace 数据和 artifacts 目录实现闭环迭代。

用户确认：多 provider 支持、所有类型 MCP 工具、先通用后特化。

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Runtime (Harness Layer)                   │
│                                                             │
│  User Input → Planner → Executor ←→ MCP Tools → Verifier  │
│                 ↑                                   │       │
│          PromptManager ← artifacts/            Guardrails   │
│                                                     │       │
│                        → Responder → Output         │       │
│                              │                      │       │
│                     Langfuse Traces ─────────────────┘       │
└──────────────────────────┬──────────────────────────────────┘
                           │ traces
┌──────────────────────────▼──────────────────────────────────┐
│                 Optimization Layer (DSPy)                    │
│                                                             │
│  Langfuse Export → Dataset → Evaluate → Optimizer           │
│                                          │                  │
│                      MIPROv2 / BootstrapFewShot / Finetune  │
│                                          │                  │
│                               → artifacts/ (prompts, demos, │
│                                  skills, guardrails)        │
└─────────────────────────────────────────────────────────────┘
```

## 目录结构

```
selfDeepAgents/
├── pyproject.toml                     # 项目定义、依赖、CLI入口
├── .env.example                       # 环境变量模板
├── .gitignore
├── README.md
│
├── config/
│   ├── settings.yaml                  # 基础配置 (模型、MCP servers、Langfuse)
│   ├── settings.dev.yaml              # 开发环境覆盖
│   └── settings.prod.yaml            # 生产环境覆盖
│
├── src/self_deep_agents/
│   ├── __init__.py
│   │
│   ├── contracts/                     # 两层共享的数据契约
│   │   ├── __init__.py
│   │   ├── agent_state.py             # LangGraph AgentState TypedDict
│   │   ├── prompt_artifacts.py        # OptimizedPrompt, FewShotSet, CompiledSkill, ArtifactManifest
│   │   ├── metric_types.py            # Metric Protocol, MetricResult
│   │   ├── trace_schemas.py           # TraceRecord, TraceDataset (Langfuse导出格式)
│   │   └── tool_schemas.py            # MCP tool definitions, tool call/result schemas
│   │
│   ├── config/                        # 配置加载与校验
│   │   ├── __init__.py
│   │   ├── loader.py                  # YAML + env merge loader
│   │   └── models.py                  # Pydantic settings: LLMConfig, MCPServerConfig, LangfuseConfig, OptimizerConfig
│   │
│   ├── harness/                       # Layer 1: 运行时编排
│   │   ├── __init__.py
│   │   ├── agent.py                   # DeepAgent 主类: 组装graph、加载artifacts、运行
│   │   ├── graph/
│   │   │   ├── __init__.py
│   │   │   ├── builder.py             # StateGraph 构建: nodes, edges, conditional routing
│   │   │   ├── nodes.py               # 图节点: plan, execute, verify, respond
│   │   │   ├── state.py               # AgentState 定义 (re-export from contracts)
│   │   │   └── conditions.py          # 条件路由函数: should_verify, needs_replan, etc.
│   │   ├── prompts/
│   │   │   ├── __init__.py
│   │   │   ├── manager.py             # PromptManager: 加载base模板 + DSPy优化覆盖
│   │   │   └── templates/             # 基础prompt模板 (Jinja2)
│   │   │       ├── system.txt
│   │   │       ├── planner.txt
│   │   │       └── verifier.txt
│   │   ├── tools/
│   │   │   ├── __init__.py
│   │   │   ├── mcp_client.py          # MCP client: 连接MCP servers, 发现tools
│   │   │   ├── tool_node.py           # LangGraph ToolNode适配器: MCP tools → graph
│   │   │   └── registry.py            # Tool注册表: 可用tools, 权限, rate limits
│   │   ├── memory/
│   │   │   ├── __init__.py
│   │   │   ├── store.py               # 对话记忆、工作记忆
│   │   │   └── checkpointer.py        # LangGraph checkpointer (SQLite)
│   │   ├── middleware/
│   │   │   ├── __init__.py
│   │   │   ├── hooks.py               # Pre/post hooks: logging, cost tracking
│   │   │   └── guardrails.py          # 运行时guardrails: 内容过滤, tool call校验
│   │   └── observability/
│   │       ├── __init__.py
│   │       └── langfuse_handler.py    # Langfuse callback handler for LangGraph
│   │
│   ├── optimizer/                     # Layer 2: DSPy 寻优
│   │   ├── __init__.py
│   │   ├── pipeline.py                # OptimizationPipeline: evaluate → optimize → export
│   │   ├── programs/
│   │   │   ├── __init__.py
│   │   │   ├── base.py                # Base DSPy Module
│   │   │   ├── planner.py             # Planner节点的 DSPy Module
│   │   │   └── verifier.py            # Verifier节点的 DSPy Module
│   │   ├── metrics/
│   │   │   ├── __init__.py
│   │   │   ├── base.py                # Metric protocol + base class
│   │   │   ├── task_completion.py     # 任务完成度 metric
│   │   │   ├── tool_efficiency.py     # 工具调用效率 metric
│   │   │   ├── safety.py              # 安全合规 metric
│   │   │   └── composite.py           # 加权组合 metric
│   │   ├── datasets/
│   │   │   ├── __init__.py
│   │   │   ├── loader.py              # 从JSON/JSONL加载测试集
│   │   │   ├── from_langfuse.py       # Langfuse traces → DSPy Examples
│   │   │   └── splitter.py            # Train/dev/test 切分
│   │   └── exporters/
│   │       ├── __init__.py
│   │       ├── artifact_exporter.py   # DSPy artifacts → harness可消费格式
│   │       ├── skill_compiler.py      # 优化demos → SKILL.md
│   │       └── guardrail_compiler.py  # 评估失败模式 → guardrail rules
│   │
│   └── cli/                           # 命令行接口
│       ├── __init__.py
│       ├── main.py                    # CLI入口 (typer)
│       └── commands/
│           ├── run.py                 # sda run -- 运行agent
│           ├── optimize.py            # sda optimize -- 运行优化
│           ├── evaluate.py            # sda evaluate -- 运行评估
│           └── export_traces.py       # sda export-traces -- 导出traces
│
├── artifacts/                         # Git-tracked 优化产物 (两层的桥梁)
│   ├── prompts/                       # DSPy优化后的prompt (JSON)
│   ├── fewshot/                       # 编译的few-shot examples (JSON)
│   ├── skills/                        # 编译的SKILL.md
│   ├── guardrails/                    # 编译的guardrail rules (YAML)
│   └── finetune/                      # Fine-tune任务元数据
│
├── data/                              # .gitignore: 数据集、traces
│   ├── testsets/                      # 测试集 JSON/JSONL
│   └── traces/                        # 导出的Langfuse trace dumps
│
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── harness/
│   │   └── optimizer/
│   └── integration/
│
└── scripts/
    └── seed_testset.py                # 创建初始测试集
```

## 核心设计决策

### 1. contracts/ 作为唯一共享层

两层**绝不互相导入**。所有共享的数据结构（state、artifact schema、metric protocol、trace schema）都定义在 `contracts/` 中。这确保了层间松耦合。

### 2. artifacts/ 目录 = 两层的桥梁

- **Git-tracked**：优化结果可版本化、可review、可回滚
- Optimizer 写入 → Harness 读取
- 格式：prompts (JSON), fewshot (JSON), skills (Markdown), guardrails (YAML)

### 3. DSPy Program 镜像 Graph Node

不对整个 Agent 做端到端优化（metric 太模糊），而是**逐节点优化**：
- `PlannerProgram` → 优化 planner 节点的 prompt
- `VerifierProgram` → 优化 verifier 节点的 prompt

每个 DSPy Module 有清晰的 Signature (`task, tools → plan`)，适合 MIPROv2 和 BootstrapFewShot。

### 4. 多 Provider 支持

通过 `config/models.py` 的 `LLMConfig` 统一抽象：

```python
class LLMConfig(BaseModel):
    provider: Literal["openai", "anthropic", "dashscope", "openai_compatible"]
    model: str
    base_url: str | None = None  # for openai_compatible
    temperature: float = 0.0
    max_tokens: int = 4096
```

`harness/agent.py` 根据 provider 选择对应的 `langchain-*` 包实例化 LLM。DSPy 侧通过 `dspy.LM()` 配置。

### 5. Langfuse 闭环

```
Runtime traces (Langfuse) 
  → from_langfuse.py 导出+过滤 
  → DSPy Examples 
  → Optimize 
  → artifacts/ 
  → PromptManager 加载 
  → 下次 Runtime 使用
```

人类可以在 Langfuse UI 中打分（feedback score），`from_langfuse.py` 按分数过滤高质量 trace 作为训练数据。

## 关键数据契约

### OptimizedPrompt (optimizer → harness)

```python
class OptimizedPrompt(BaseModel):
    node_name: str          # "planner" | "verifier"
    system_prompt: str      # 优化后的prompt文本
    optimizer: str          # "miprov2" | "manual"
    metric_score: float     # 在eval set上的得分
    optimized_at: datetime
```

### ArtifactManifest (索引所有active artifacts)

```python
class ArtifactManifest(BaseModel):
    version: str
    prompts: list[OptimizedPrompt]
    fewshot_sets: list[FewShotSet]
    skills: list[CompiledSkill]
    guardrail_rules: list[str]
    active_finetune_model: str | None = None
```

### Metric Protocol (两层共用)

```python
class Metric(Protocol):
    name: str
    threshold: float
    def __call__(self, example, prediction, trace=None) -> float: ...
```

## 核心依赖

```toml
[project]
name = "self-deep-agents"
version = "0.1.0"
requires-python = ">=3.11"

dependencies = [
    # LLM
    "langchain-core>=0.3,<0.4",
    "langchain-openai>=0.3,<0.4",
    "langchain-anthropic>=0.3,<0.4",
    "langgraph>=0.4,<1.0",
    # MCP
    "mcp>=1.5,<2.0",
    "langchain-mcp-adapters>=0.1",
    # Observability
    "langfuse>=2.50,<3.0",
    # Optimization
    "dspy>=2.6,<3.0",
    # Config
    "pydantic>=2.0,<3.0",
    "pydantic-settings>=2.0,<3.0",
    "pyyaml>=6.0",
    # CLI
    "typer>=0.12,<1.0",
    "rich>=13.0",
    # Template
    "jinja2>=3.1",
]

[project.scripts]
sda = "self_deep_agents.cli.main:app"
```

## 开发路线 (7个Phase)

### Phase 1: Foundation (最小可运行Agent)
**目标**: 一个能跑的 LangGraph Agent + Langfuse tracing

创建文件：
- `pyproject.toml`, `.gitignore`, `.env.example`
- `config/settings.yaml`, `src/self_deep_agents/config/models.py`, `config/loader.py`
- `contracts/agent_state.py`
- `harness/graph/state.py`, `nodes.py` (2节点: plan + respond), `builder.py`
- `harness/prompts/manager.py`, `templates/system.txt`, `templates/planner.txt`
- `harness/observability/langfuse_handler.py`
- `harness/agent.py`
- `cli/main.py`, `cli/commands/run.py`

**验证**: `sda run "你好"` → 得到响应 + Langfuse中出现trace

### Phase 2: MCP Tool Integration
**目标**: Agent能通过MCP调用外部工具

创建文件：
- `harness/tools/mcp_client.py`, `tool_node.py`, `registry.py`
- `contracts/tool_schemas.py`
- 更新 `graph/nodes.py` (加executor, tool_caller节点), `graph/builder.py`, `graph/conditions.py`

**验证**: Agent回答需要工具调用的问题，trace中可见tool calls

### Phase 3: Verification + Guardrails + Memory
**目标**: Agent自验证输出 + 运行时guardrails

创建文件：
- `graph/nodes.py` 加verifier节点
- `middleware/guardrails.py`, `middleware/hooks.py`
- `memory/store.py`, `memory/checkpointer.py`
- `harness/prompts/templates/verifier.txt`

**验证**: Agent有验证循环，guardrail拒绝在trace中可见

### Phase 4: Optimization Foundation (DSPy评估)
**目标**: 能对planner节点跑DSPy Evaluate

创建文件：
- `contracts/metric_types.py`, `contracts/prompt_artifacts.py`
- `optimizer/programs/planner.py`
- `optimizer/metrics/task_completion.py`, `metrics/composite.py`, `metrics/base.py`
- `optimizer/datasets/loader.py`, `datasets/splitter.py`
- `optimizer/pipeline.py`
- `cli/commands/evaluate.py`
- `data/testsets/` 手工标注20-30条测试数据

**验证**: `sda evaluate` 产出baseline分数

### Phase 5: Prompt Auto-Optimization (MIPROv2)
**目标**: MIPROv2优化prompt → 导出artifact → Harness加载

创建文件：
- `optimizer/optimizers/prompt_optimizer.py`
- `optimizer/exporters/artifact_exporter.py`
- `cli/commands/optimize.py`
- `contracts/trace_schemas.py`

**验证**: `sda optimize --type prompt` → `artifacts/prompts/` 有新文件 → 重跑agent使用优化后prompt

### Phase 6: Langfuse Trace Loop (闭环)
**目标**: Langfuse traces → DSPy训练数据 → Few-shot编译

创建文件：
- `optimizer/datasets/from_langfuse.py`
- `optimizer/optimizers/fewshot_optimizer.py`
- `optimizer/exporters/skill_compiler.py`, `guardrail_compiler.py`
- `cli/commands/export_traces.py`

**验证**: 运行agent积累traces → 导出 → few-shot优化 → 分数提升

### Phase 7: Fine-tuning Pipeline
**目标**: Teacher-student蒸馏 (BootstrapFinetune)

创建文件：
- `optimizer/optimizers/finetune_optimizer.py`
- 更新 `optimizer/pipeline.py` 支持完整 Evaluate → BootstrapFewShot → BootstrapFinetune 链路

**验证**: 完整pipeline跑通

## 验证方式

每个Phase完成后：
1. **单元测试**: `pytest tests/unit/` 验证各组件独立正确
2. **CLI验证**: 通过 `sda` 命令行实际运行
3. **Langfuse检查**: 确认trace数据结构正确、可被optimizer消费
4. **Artifact检查**: 确认优化产物格式正确、可被harness加载
