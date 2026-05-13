# 项目：finetuned model agentic capability 评测框架

> 状态：构思阶段，独立于 `inventory_patrol_pilot`。本文件记录初步方案与评审结论，后续由独立项目承接落地。
>
> 来源：2026-05-13 钉钉文档评审讨论（钉钉文档 `m9bN7RYPWdlGnPMrubbj0pR3WZd1wyK0`）。

## 背景与目标

需要一套评测框架，用于回归 finetune 出的 base model 的 **agentic 能力**（tool use、planning、tool correctness、task completion、plan adherence、code correctness）。

评测有两类需求，必须分开走：

- **通用 agentic 能力**：与具体业务 skill 解耦，跑公开 agent benchmark，看 finetune ckpt 之间的相对趋势。
- **垂域业务能力**：基于线上数据合成评测样本，看在我们自己业务（openclaw / inventory_patrol 这类 skill）上的实际表现。

## 方案分工（评审后调整版）

```
┌─────────────────────────────────────────────────────────────┐
│ Track A: 开源通用 agentic 回归                                │
│   harness: OpenHands evaluation harness   ← 不是 EvalScope  │
│     ├── SWE-bench Lite        (代码)                         │
│     ├── GAIA 子集             (办公/多步推理)                 │
│     ├── WebArena / WebShop    (网页操作)                     │
│     └── τ²-bench              (工具使用)                     │
│   结果走 OpenHands 自带 reporter，趋势图入 Langfuse           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Track B: 垂域 skill 回归 (复用 skill_optimizer v1 框架)       │
│   评测集分层（关键）:                                          │
│     L1 真实 trace replay  (≥50%, 主指标，对接 ingestion)     │
│     L2 trace + 表达扰动   (30%, 鲁棒性)                       │
│     L3 LLM 合成新场景     (≤20%, 边界压测，不进主指标)         │
│                                                             │
│   执行: SkillAdapter → DeepEval scorers (pytest 风格)        │
│   归因: trace 落 Langfuse → span 级 debug                    │
│                                                             │
│   指标对齐 inventory_patrol 现有口径:                         │
│     routing_recall + reply_quality                          │
│     efficiency = 0.7·steps_score + 0.3·latency_score        │
└─────────────────────────────────────────────────────────────┘
```

## 关键评审结论

### ✅ 同意的部分

1. **"DeepEval（判定）+ Langfuse（归因）" 双层架构**——和我们项目 `routing_recall + reply_quality + efficiency` 三件套的演化方向一致，前者负责 CI 化回归，后者负责 trace 级 debug。
2. **DeepEval 的 `AgentTestCase` / `ToolCorrectnessMetric`**——和 inventory_patrol 评测里 `tool_calls` 字段一一对应，迁移成本约等于把 `runner.py` 的 metric 计算包成 DeepEval scorer。
3. **WebShop / GAIA / τ²-bench 取子集而非跑全榜**——方向对，需要的是和业务对齐的子集。
4. **markdownlint 量化输出格式**——补充 reply_quality 之外的可机判维度，可直接复用现有 `reply_format_score` 思路。

### ⚠️ 三个必须先解决的问题

#### 1. 评测对象边界要分两个 Track

钉钉文档原方案把 B-end Business（端到端 skill 系统）和 WebShop/GAIA/SWE-bench（开放域基准）混在一张表里，但这两类**评的不是同一个东西**：

- B-end Business 评的是 `model + skill prompt + tools` 的系统效果（≈ inventory_patrol v0~v3 这条线）
- WebShop/GAIA/SWE-bench 评的是 `finetuned base model` 的通用 agentic 能力（≈ 底座 SFT 回归）

→ 必须显式拆成 Track A / Track B，**复用 skill_optimizer v1 pivot 的 SkillAdapter 框架** 直接支撑 Track B；Track A 接 OpenHands。

#### 2. 开源 framework 选型：OpenHands evaluation 优于 EvalScope

钉钉文档原方案候选是 EvalScope。实际上 EvalScope 强项是**知识/能力榜（MMLU、C-Eval）**，agent 不是其主场。

| 选型 | 评价 |
|---|---|
| **OpenHands evaluation harness** ⭐推荐 | 自带 SWE-bench/GAIA/WebArena/τ-bench runner，社区活跃，与原方案"OpenHands 推理"已对齐 |
| inspect-ai (UK AISI) | 专为 agent eval 设计，API 干净；中文生态弱 |
| lm-evaluation-harness | 知识榜事实标准；**不是 agent 框架，不要用它评 agentic** |
| EvalScope | agent 支持薄 |

→ **不要重复造**：原方案"OpenHands 产生 trace 给 DeepEval 评"=自己写一遍 OpenHands evaluation harness 已经在做的事。直接用其 harness 跑公开榜，结果汇总到 Langfuse 看趋势即可。知识能力回归再叠 EvalScope/lm-eval-harness。

#### 3. "基于线上数据合成"必须分层（最关键的隐患）

钉钉文档原方案 B-end Business 一句话写"使用 DeepEval Synthesizer 合成 case"——**这正是 inventory_patrol v3 翻车的同一条路**（见 `feedback`/`project` 记忆 `inventory_patrol_v2.1_v3_outcome`：合成 GT 在 syn55 上 −1.8% recall，"合成 GT 路线已彻底用尽"）。

合成有两种本质不同的做法：

| 做法 | 性质 | 风险 |
|---|---|---|
| A. 线上 trace 直接当 GT（≈ replay） | input/output 都来自真实 trace，仅做表达扰动 | 低 |
| B. 线上 trace 做种子，LLM 扩展生成新 case + **新 GT** | DeepEval Synthesizer 默认行为，GT 也是 LLM 写的 | 高（v3 翻车根因） |

→ 评测集必须分层，L1 占大头：

```
垂域评测集
├── L1 真实 trace replay      （≥50%, 主指标锚）
├── L2 真实 trace + 表达扰动   （30%, 鲁棒性）
└── L3 LLM 合成新场景         （≤20%, 边界压测，不进主指标）
```

→ 我们的 `scripts/ingest_openclaw_trace.py` 已把 L1 路径铺到"占位 GT"。**关键路径瓶颈是把"占位 GT → 可机判 GT"的人工标注流水线补上，不是 DeepEval/Langfuse 接入。**

#### 4. efficiency 口径对齐

钉钉文档原方案把 efficiency 写为 `StepEfficiencyMetric`（仅步数）。但实测（见 `project_inventory_patrol_metric_latency`）：步数和 wall-clock 不对称——`v1-clean` 剥离 demos 后步数没动、wall-clock +326ms。

→ efficiency 必须沿用 inventory_patrol 已落地口径：

```
efficiency = 0.7 · steps_score + 0.3 · latency_score
```

避免一个体系两套口径。

## 接入难度速估

| 模块 | 工作量 | 风险 |
|---|---|---|
| DeepEval 框架本体 | 小（pip 装，pytest 风格） | 低 |
| Langfuse self-host | 中（ClickHouse + PG + Redis） | 中（部署较重） |
| 复用 SkillAdapter 打通 DeepEval | 小 | 低 |
| OpenHands evaluation harness | 中（业务 tool schema 适配） | 中 |
| WebShop 环境 | 中（110 万商品镜像） | 中 |
| GAIA 子集 | 小 | 低 |
| **真实 trace → 可机判 GT 标注流水线** | **中** | **中 — 关键路径瓶颈** |

## 决议

- 用户决定**另起独立项目**承接落地，不在 `selfDeepAgents` 内推进。
- 本文件作为构思 / 评审记录留存，独立项目动工时直接引用。
- 与 selfDeepAgents 的衔接点：
  - `skill_optimizer v1 pivot` 的 SkillAdapter 框架 → Track B 复用
  - `scripts/ingest_openclaw_trace.py` 的 trace ingestion → 垂域 L1 数据源
  - inventory_patrol 三件套指标口径 → Track B 主指标对齐基线

## 一句话总结

**分工对——开源 agent 榜走 OpenHands evaluation（不是 EvalScope）、垂域走 DeepEval + Langfuse；但"基于线上数据合成"必须拆成 L1/L2/L3、L1 占大头，否则会重蹈 inventory_patrol v3 覆辙。关键路径瓶颈是真实 trace 的人工标注流水线，不是工具链选型。**
