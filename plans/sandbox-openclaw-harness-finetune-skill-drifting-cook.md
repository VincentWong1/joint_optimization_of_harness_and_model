# 试点：inventory_patrol skill 的 harness finetune

## Context

工程团队已基于 sandbox 部署 openclaw 商家助手（一个类 Claude Code 的 markdown skill harness：`SOUL.md` / `AGENTS.md` / `TOOLS.md` + `skills/<name>/`）。其中 `inventory_patrol`（必选品库存巡检）skill 通过 OpenCLI 把查数操作转成终端指令，由 skill 编排大模型调度。线上反馈两个问题：(1) 执行慢；(2) 一定概率查数结果无效。

本项目 `selfDeepAgents` 是一个 LangGraph harness + DSPy 优化的两层框架，目前只有 Phase 1 最小骨架（无工具、无评估集、无 metric 实现），需要以这个 skill 作为首个落地试点跑通"采集 → 评估 → 优化 → 交付"的最小闭环。

**用户已确认的边界：**

- **优化对象范围**：仅 `skills/inventory_patrol/` 内的 markdown（SKILL.md 及配套 few-shot），不动 SOUL/AGENTS/TOOLS。
- **评估数据来源**：Langfuse trace + 业务侧可抽取 ground truth（必选品清单、正确库存值）。
- **交付形式**：markdown 改写 + diff，工程同学 review 后由他们重新部署 sandbox。本项目**不**端到端联调 openclaw runtime。
- **skill 内容获取**：用户会把现有 `inventory_patrol/SKILL.md` 拷到 `/Users/wigi/PycharmProjects/openclaw_merchant_assistant/skills/inventory_patrol/`（目前是空目录）。
- **三个业务 metric**：①必选品无遗漏（recall）②返回数据正确（accuracy）③恢复越快越好（latency）。

**可行性结论：可行，且与本项目定位高度契合。** 但要把"慢"和"错"分开看：
- "错"是 prompt/few-shot 可优化问题，DSPy bootstrap-fewshot/MIPROv2 适用。
- "慢"主要由调用链长度（串行 OpenCLI 调用、重试、模型 thinking）决定，markdown 优化能贡献的是"少调用 / 一次调对 / 并行调度提示"，不能从根本上解决，需要工程侧配合。

## 试点目标（首阶段）

围绕 `inventory_patrol/SKILL.md` 单文件，以离线评估证明：

- recall ≥ 0.98（必选品无遗漏，**报告口径**硬约束）
- accuracy ≥ 0.95（数据返回全部正确，**报告口径**硬约束）
- 工具调用步数中位数下降（latency 的代理指标，不直接用墙钟时间，避免引入网络抖动）
- **首阶段不承诺指标 delta**，承诺：建立闭环 + 给出方向性结论（prompt 还能挤多少 / 瓶颈是否在 runtime）

## 工作流程

### Phase A — 前置准备（用户协作，**关键路径，需第 0 天就启动**）

1. 用户把现网 `inventory_patrol/SKILL.md`（含同目录 few-shot/示例）拷到本地 clone。
2. 用户提供 Langfuse 项目读取凭证 + trace 时间窗口 + 抽样过滤条件（按 skill 名 / cron job tag）。
3. 用户提供必选品清单 + 库存正确值的取数口径（业务库 SQL 或 API），用作 ground truth。**ground truth 必须从业务 DB 在 trace 同一时间戳取，不得从 trace 的 tool 输出反推**——否则 recall/accuracy 会被泄漏到 1.0。
4. 用户提供 PII 字段清单（商家 ID / 手机 / 门店地址等），用于脱敏白名单。

A 步骤是 B/C/D 全部模块的前置。任何一项卡住都要立即升级。

### Phase A.1 — 当前 skill.md 静态审计与基线版本固化

读完 `/Users/wigi/PycharmProjects/openclaw_merchant_assistant/skills/inventory_patrol/skill.md`（v0，40 行）后已识别的硬伤，写入 `data/inventory_patrol/known_issues.md`：

| # | 位置 | 问题 | 影响 metric |
|---|---|---|---|
| 1 | frontmatter `description` | 写成"处理当前商家未回复差评"，与本 skill 业务无关 | recall（router 选不中→直接遗漏） |
| 2 | Step 2 | 假设只有一个 REQUIRED 分组；多分组场景未定义 | recall |
| 3 | Step 4 | 未强制遍历必选品分组下**全部 SKU** | recall |
| 4 | Step 5 阈值 | `库存 < 最大库存 × 10%` 未处理 `maxStock = 0 / null`（除零 / 语义无效） | accuracy（"查数结果无效"高度相关） |
| 5 | 全流程 | 多个 REQUIRED 分组场景未提示可并发 Step 3 | latency |
| 6 | 全流程 | CLI 报错（401 / timeout / 非法 JSON）处理缺失 | accuracy + recall |
| 7 | Step 2/5 | "禁止输出任何内容"语义模糊（思考链 / 工具调用是否也禁止？） | LLM 行为漂移 |

`known_issues.md` 同时承担两个用途：
- 作为 `failure_taxonomy.py` 的来源——每条问题映射到一个失败类，确保 D1 的 mock 失败采样器能覆盖。
- 作为 Phase F 对比 checklist——优化后必须逐条标注"已解决 / 已接受残留 / 引入新失败"。

**三方基线对照**（避免把"修 bug"误算成"DSPy 价值"）：
- **v0**：当前 skill.md，原样跑 baseline。
- **v0.5**：人工修 #1（description）+ #4（maxStock 边界）这两条"显然 bug"，不做任何 prompt engineering，作为 DSPy 优化的下限。
- **v1**：DSPy 优化产物（Phase E）。

Phase F 的对比报告必须三方并列。**若 v1 ≈ v0.5**：DSPy 没贡献，本试点真正的价值是建立闭环 + 暴露了 skill 写作质量问题——这本身是结论，不是失败。

### Phase B — 数据基建（本项目）

新增目录与模块：

- `data/inventory_patrol/raw_traces/`：从 Langfuse 拉取的原始 trace（JSONL）。
- `data/inventory_patrol/eval_set.jsonl`：每条形如
  ```
  {"case_id": ..., "context": {...商家身份/时段/...},
   "expected_must_have_skus": [...],
   "expected_inventory": {sku: stock},
   "trace_id": "...", "ts": "...",
   "failure_modes_observed": ["timeout"|"partial_result"|"stale"|...],
   "notes": "..."}
  ```
- `scripts/pull_langfuse_traces.py`：拉取 + PII 脱敏（按 Phase A 的字段清单）+ 落盘（一次性，不进 CI）。
- `scripts/build_eval_set.py`：从 trace 提取 case context，**从业务 DB 在同一 ts 取 ground truth**（绝不从 trace tool 输出反推），合成 `eval_set.jsonl`。
- `src/self_deep_agents/optimizer/datasets/inventory_patrol.py`：把 `eval_set.jsonl` 包装成 DSPy `Example` 列表，并按 `failure_modes_observed` 做**分层划分**为 train（用于 BootstrapFewShot demo 池）/ val（用于优化器内部评分）/ test（held-out，仅在 Phase F 报告时用一次）。比例约 50/25/25。

目标 case 数：**80~120**（Plan agent 指出 30~50 在概率失败问题上方差过大；按 failure mode 分层抽样以保证每类失败至少 5 条）。如果业务 trace 量不足，先以 80 起步并标注样本量上的不确定性。

### Phase C — Metric 实现

在 `src/self_deep_agents/optimizer/metrics/inventory_patrol.py` 实现：

- `recall_metric(pred, gold) -> float`：覆盖到的必选品 / 应覆盖的必选品。
- `accuracy_metric(pred, gold) -> float`：库存值精确匹配率。
- `tool_call_steps(pred) -> int`：本次 run 的工具调用次数。
- `steps_score(steps) -> float`：`max(0, 1 - steps / STEP_CAP)`，`STEP_CAP` 取 baseline median × 2，固定后写进 metric 模块（保证可复现，不随调用变化）。
- **`training_metric(pred, gold) -> float`（DSPy 优化目标，平滑 reward）**：
  ```
  0.6 * recall + 0.3 * accuracy + 0.1 * steps_score
  ```
  全程平滑，不带硬截断——硬截断会让 BootstrapFewShot 的 demo 池在早期几乎全部为 0，搜索退化（Plan agent 重要反馈）。
- **`reporting_metric(pred, gold) -> dict`（仅用于报告）**：返回 recall / accuracy / steps，并在 `diff_skill.py` 里检查 ≥0.98 / ≥0.95 的硬约束门槛。
- **`bootstrap_ci`**：在 `diff_skill.py` 里用 1000 次 bootstrap 出 95% CI，避免 80~120 case 上把噪声当提升。

新增 `src/self_deep_agents/contracts/failure_taxonomy.py`：枚举 `timeout / partial_result / stale_data / malformed_response / sku_mismatch / unauthorized` 等失败类，metric 与 mock 共享同一套定义。

### Phase D — 评估 harness（**全 plan 最关键**，拆为 D1+D2，中间设硬门）

本项目交付 markdown diff，**不**联调 openclaw runtime；同时 DSPy 优化又必须能跑出 `pred`。结论：**用本项目的 harness 模拟 openclaw 的 skill 执行环境**。

#### D1 — Mock OpenCLI + 失败重放保真度审计（**硬门**）

用户报告的 "查数结果无效" 失败模式恰好发生在 OpenCLI/数据源边界——也就是被我们 mock 掉的那一层。如果 mock 只回放成功响应，DSPy 会针对一个永远不出错的世界优化 prompt，与真实失败模式正交。

`src/self_deep_agents/harness/tools/openclaw_cli_mock.py` 必须：

- **按 trace 真实失败频率重放失败**（timeout、malformed JSON、partial result、stale data 等，按 `failure_taxonomy` 分类）。失败采样器从原始 trace 估计每类失败的发生率，并按相同分布注入到回放中。
- 命中真实 trace 输入时：直接 replay 真实响应（成功/失败原样）。
- **未命中时：默认返回 `error/unknown`，禁止合成"完美"响应**。否则会教 prompt 盲目信任工具输出，与真实环境相反。

D1 的硬门：在交 D2 前必须输出 `mock_fidelity_report.md`，至少包含：

- 失败模式覆盖率：`failure_taxonomy` 中每一类在 mock 中的命中率与 trace 中的真实频率对比，差值 ≤ 10%。
- replay 命中率：eval_set 中 mock 能命中真实 trace 的比例 ≥ 60%。
- 不达标则不进 D2，先回去补 trace 或调整失败分布。

#### D2 — Skill runner + LangGraph tool 节点

将 runner 放到 `src/self_deep_agents/optimizer/runners/inventory_patrol_runner.py`（**不**放在 `harness/skills/` 下——它存在的目的是给 metric 用，不属于通用 harness）。

- 输入：当前 SKILL.md 内容（候选 prompt）+ 一条 case 的 context。
- 行为：把 SKILL.md 当 system prompt 喂给 LLM，按 LangGraph 的 `plan → tool_call → respond` 循环跑（需要在 `harness/graph/nodes.py` 里加 tool 节点 + 循环边）。
- 输出：`pred = {"queried_skus": [...], "queried_inventory": {...}, "tool_call_count": N, "tool_call_trace": [...]}`。

### Phase E — 基线 + DSPy 优化

1. `scripts/run_baseline.py`：用当前 SKILL.md 跑**全量 eval set**，落盘 baseline 报告（每条 case 的 recall/accuracy/steps + 失败归因），固定 `STEP_CAP`。
2. 在 `src/self_deep_agents/optimizer/programs/inventory_patrol.py` 里把 SKILL.md **按子步骤粒度**拆成 DSPy `Signature`，而非整篇 prompt 整体优化：
   - `SelectRequiredGroups`（对应 Step 2，输入：groups 列表；输出：REQUIRED 分组 ID 列表，需处理多分组）
   - `EnumerateGroupSkus`（对应 Step 3-4，输入：groupGlobalId；输出：SKU 列表 + stock/maxStock）
   - `StockAlertDecision`（对应 Step 5，输入：sku 列表；输出：告警列表，需处理 maxStock = 0/null）
   - 每个 Signature 单独优化，metric 也按子步骤归因——这样能定位是哪一步在掉点，而不是只看整体一个数字。
   - 配置：`dspy.configure(lm=..., cache=True)`（80+ case × N 候选必须缓存，否则成本爆炸）。
   - 优先用 `BootstrapFewShotWithRandomSearch(metric=training_metric, max_bootstrapped_demos=4, max_labeled_demos=8, num_candidate_programs=8)`。
   - 若某个子 Signature baseline 已 ≥0.9 但 plateau，再对该子步骤上 `MIPROv2(auto="light", prompt_model=..., task_model=...)`。
   - **优化器只看 train + val，test 完全 held-out**。
3. `scripts/optimize_inventory_patrol.py`：跑优化、产出 `artifacts/skills/inventory_patrol/`（含编译后的 prompt 文本 + few-shot 示例 + 优化器超参快照）。
4. 把 DSPy artifact 还原成符合 openclaw 约定格式的 `SKILL.md`——**这一步内联在 `optimize_inventory_patrol.py` 里**，先不抽 `optimizer/exporters/markdown.py`，等出现第二个 skill 时再抽象。

### Phase F — 交付

1. `scripts/diff_skill.py`：v0 / v0.5 / v1 三方对比报告：
   - 在 **held-out test set** 上分别跑 v0、v0.5、v1，给出 recall / accuracy / steps + bootstrap 95% CI。
   - 三方 unified diff（v0 → v0.5、v0.5 → v1）。
   - 检查 ≥0.98 / ≥0.95 报告口径硬约束门槛是否达成。
   - 按 `failure_taxonomy` 分类列出 v1 相比 v0.5 解决了哪些失败、引入了哪些新失败。
   - 对照 `known_issues.md` 7 条问题逐条标注状态（解决 / 接受残留 / 新增）。
2. 把 diff + 报告交回工程侧。本项目侧不直接 push openclaw 仓库。

## 关键文件清单

需要新增（按依赖顺序）：

- `data/inventory_patrol/known_issues.md`（A.1 静态审计产物）
- `data/inventory_patrol/skill_v0.5.md`（人工修 bug 版，对照基线）
- `data/inventory_patrol/eval_set.jsonl`（含 train/val/test 划分字段）
- `scripts/pull_langfuse_traces.py`、`scripts/build_eval_set.py`
- `src/self_deep_agents/contracts/failure_taxonomy.py`
- `src/self_deep_agents/optimizer/datasets/inventory_patrol.py`
- `src/self_deep_agents/optimizer/metrics/inventory_patrol.py`
- `src/self_deep_agents/harness/tools/openclaw_cli_mock.py`（含失败采样器）
- `src/self_deep_agents/optimizer/runners/inventory_patrol_runner.py`
- `src/self_deep_agents/optimizer/programs/inventory_patrol.py`
- `scripts/run_baseline.py`、`scripts/optimize_inventory_patrol.py`、`scripts/diff_skill.py`
- `data/inventory_patrol/mock_fidelity_report.md`（D1 硬门产物）

需要扩展：

- `src/self_deep_agents/harness/graph/nodes.py`：补一个 `tool_call` 节点，让 plan → tool_call → respond 能循环。
- `src/self_deep_agents/contracts/metric_types.py`：若现有 `MetricResult` 不够表达三维 metric，补字段。

## 风险与已知边界

1. **Mock 与真实环境的偏差**：靠 D1 硬门兜底，详见 Phase D1。绝不允许"未命中合成完美响应"。
2. **延迟 metric 是代理指标**：`tool_call_steps` 不等于墙钟延迟。真实 P95 需工程侧重新部署后做线上 A/B，本试点不承诺。
3. **样本量与显著性**：80~120 仍偏小，所有报告必须带 bootstrap CI；如果 baseline recall 已 ≥0.95，把试点结论定位为"建立闭环 + 给出瓶颈方向（prompt vs runtime）"。
4. **Ground truth 泄漏**：必须从业务 DB 在 trace 同 ts 取，禁止从 trace tool 输出反推（Plan agent 重要反馈）。
5. **train/val/test 隔离**：DSPy 优化器只看 train+val，最终报告用 test。任何把 test 喂给优化器的代码改动需作为 PR 阻断项。
6. **PII**：Phase A 提供脱敏字段清单，`pull_langfuse_traces.py` 内置脱敏 + 单测验证字段不漏。
7. **DSPy 2.6+ 注意点**：必须 `dspy.configure(lm=..., cache=True)`；MIPROv2 需 `auto="light"` 起步并指定 `prompt_model`/`task_model`，不要默认配置直接跑。

## 验证

按以下顺序自验：

1. `python scripts/pull_langfuse_traces.py --skill inventory_patrol --since 2026-04-01`：能落盘 ≥80 条原始 trace，PII 字段全脱敏（单测覆盖）。
2. `python scripts/build_eval_set.py`：生成 `eval_set.jsonl`，人肉抽查 5 条 ground truth 与业务 DB 在同一 ts 一致；分层划分后每个 failure mode 在 train 至少 5 条。
3. `pytest tests/unit/optimizer/test_inventory_patrol_metric.py`：recall/accuracy/training_metric 在 toy case 上行为正确，`training_metric` 是平滑的（无 0 截断）。
4. `python scripts/audit_mock_fidelity.py`（D1 硬门）：输出 `mock_fidelity_report.md`，失败模式分布偏差 ≤10%，replay 命中率 ≥60%。不达标不进 D2。
5. `python scripts/run_baseline.py`：跑通全 eval，输出 baseline 报告 + 固定 `STEP_CAP`。
6. `python scripts/optimize_inventory_patrol.py`：跑通 DSPy 优化，产出 artifact，确认优化器只读 train+val。
7. `python scripts/diff_skill.py`：在 held-out test 上输出 v0/v0.5/v1 三方 markdown diff + 带 bootstrap CI 的对比报告 + failure mode 分类对比 + known_issues 逐条状态。人工 review 改写。
