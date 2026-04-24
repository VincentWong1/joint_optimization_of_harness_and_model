# The Theroy of Joint Optimization of Harness & Model

harness engineering的开发工作可以概括为6个部分：
1. context engineering
2. tool orchestration
3. planning & decomposition
4. memory & state management
    - working context
    - session state
    - long-term memory
5. verfication & guardrails
6. lifecycle management

![harness_engineering](asset/harness_hierarchy.png)
> - Layer 1: LLM
> - Layer 2: build-in Harness (runtime execution)：tool orchestation, memory&state management, planing&decomposition, verification&guardrail
> - Layer 3: add-on Harness (context engineering): skills&knowledge, few-shots, middleware&hooks, tool def, role&tone

## Construction: Initializer-exector split

初始化器（initializer）执行一次，并且建立持久化的项目环境（project environment）：文件夹结构、功能列表、init.sh，以及初始化git commit。每一个执行器都会从该环境（project environment）读取数据并完成一项独立功能的逐步开发、运行测试、提交代码、更新进度文件，然后清空退出。项目环境（project environment）是所有会话共享的记忆存储。

Claude Code采用的是这种两步解决方案：
<info>
1. Initialize agent：第一个agent会话使用特殊prompt来让模型设置初始环境：一个init.sh脚本、一个claude-progress.txt（用于记录agents已经完成的操作），以及一个初始git commit（用于显示已添加文件）。
    - Feature list: 为了解决智能体one-shotting an app 或者过早认为一个项目已经完成的问题，我们鼓励initialize agent编写一份全面的功能需求文件，以扩展用户的原始指令。
    - Incremental progress：在提供初始环境脚手架后，在下一轮的coding agent每次只会被分配一项功能开发。这种“渐进”方式被证明对于解决agent倾向于一次性做太多事情的问题，非常有效。实验显示，要求模型使用描述性的提交信息将开发进度commit to git，并在进度文件中写入进度摘要，是agent实现操作回滚、错误修复的最佳方式。
    - Testing：agent倾向于在没有进行充分测试的情况下将某个功能开发标记为已完成（最主要的错误模式之一），通过提供特定测试工具，可以显著提高智能体的任务完整性表现——这样agent可以识别并修复哪些仅从代码语法层面无法明显看出的错误。

    ```markdown
    [Assistant] I'll start by getting my bearings and understanding the current state of the project.
    [Tool Use] <bash - pwd>
    [Tool Use] <read - claude-progress.txt>
    [Tool Use] <read - feature_list.json>
    [Assistant] Let me check the git log to see recent work.
    [Tool Use] <bash - git log --oneline -20>
    [Assistant] Now let me check if there's an init.sh script to restart the servers.
    <Starts the development server>
    [Assistant] Excellent! Now let me navigate to the application and verify that some fundamental features are still working.
    <Tests basic functionality>
    [Assistant] Based on my verification testing, I can see that the fundamental functionality is working well. The core chat features, theme switching, conversation loading, and error handling are all functioning correctly. Now let me review the tests.json file more comprehensively to understand what needs to be implemented next.
    <Starts work on a new feature>
    ```
2. Coding agent：后续的每一个会话会要求模型逐步取得进展，并实现记忆/状态（memory/state）文件的结构化更新（claude-progress.txt、git commit）。快速进入状态的操作：
    - 执行pwd查看当前工作目录，了解可以获取的文件系统现状；
    - 读取git logs和progress files，了解最新的工作进展
    - 阅读feature list file并挑选尚未完成的功能中优先级最高的项目推进。
</info>

## Principles of dev

### Principle 1: Model和Harness的一体化开发，Harness先行，不约束在EBM和Qwen上
<div style="background-color: #1c374f; padding: 10px;">
为什么首先是harness？因为harness相较于llm，存在下述4点不可替代的作用。同时，由于模型agentic capability本身具备的通用性，LLM越来越被作为跑plugable commodity：

| 原因 | 说明 |
| --- | --- |
|Capability Expansion|模型越强能做的事儿越多 → 能做的事儿越多带来的错误模式也更多样化 → 更多样化的错误模式需要更复杂的错误处理机制 → 错误处理机制由harnesses负责。|
|Cost Optimization|更强的模型成本也更高。好的harnesses设计会把简单任务路由到低成本模型，复杂任务路由给高成本模型，实现边际效用最大化。这些都应该通过测试校准，而不是无脑的取SOTA。|
|Reliability requirements|生产环境需要达到99.9%的可用性（任务能正常运行的时间/任务数占比）。模型是基于概率预测的，Harnesses用过重试逻辑、回退和验证机制保证了任务的可靠性。|
|Organizational integration|模型无法处理身份验证、权限控制、速率限制、合规检查。Harnesses可以。|

<div style="font-size: 10px">

> 以Manus为例，在6个月内重写了5次，但模型基座并没有变化。每一次架构的重写都显著提升了manus的可靠性和任务完成率；<br>而Vercel最初配备了功能齐全的工具库，涵盖搜索、代码、文件和API工具的方方面面。但结果却令人失望。Agents在执行过程中变得困惑、工具调用冗余、无必要的执行步骤泛滥。然后，他们去除了80%的agent工具并且获得了更好的结果。

</div>

模型之间有能力差异，主要通用一些agentic capabiltiy benchmark体现，但我们需要注意2点：
1. 这种差异化优势具备显著的迁移性，在上述benchmark表现好的模型，无论采用何种框架，都表现出更好的效果（例如claude、Kimi-k2.5）。
2. 这种差异会被customized harness的适配性放大（例如qwen和claude之间；但可能是因为kimi消费了大量claude code合成数据，kimi系列在claude code上和claude相比，表现差距并没有显著放大）
</div>

### Principle 2: 从简单的原型开始，从场景下的单一任务开始，“由俭入奢”
<div style="background-color: #1c374f; padding: 10px;">
框架和结构都是用既有的方案，所以我们并不需要花时间在原型初始化上。我们一开始就有可在开发环境执行的agent实例，一旦场景确定，基于场景的模拟环境构建完毕，主要工作就变成了找任务、跑数据、发现失败模式、找harness解决办法，如此循环，直到我们既定的测试目标全部成功。

---

|序号|步骤|说明|
|---|---|----|
|1|Start with one task end-to-end|选择一个具备交付价值的agent任务，基于该任务构建最小化的支撑框架以确保其可靠性（MVP），然后部署上线，直接从生产环境反馈中学习迭代。概括起来：<span style='color: red'>价值导向 → 轻量起步 → 快速上线 → 数据驱动</span>|
|2|Instrument everyting|对所有环节做监控，记录每一次工具调用、错误环节、人为干预、超时情景。只有可以被准确衡量的点，才有改进可能。|
|3|Iterate based on failure modes|每一次故障都暴露出防护措施缺失的点。添加防护措施，部署，修复，然后寻找下一个故障，如此循环。|
|4|Measure outcomes，not activity|衡量结果，而非活动量——最总任务完成情况，而非token数。衡量业务满意度，而不是速度。优化任务可靠性，而不是任务功能丰富度。|
</div>

### Principle 3: 不要想着如何运行成功，而是想着什么是成功的情景，通过情景测试找到当前agent和期望的成功之间的差异，指导更新
<div style="background-color: #1c374f; padding: 10px;">
claude code作为比较通用的agent代表，Boris及其团队在开发中总结了如下4个比较常见的失败场景，以及他们对应的解决办法，对我们也比较有启发：

---

|问题描述|Initializer Agent解决办法|Coing Agent解决办法|
|---|---|----|
|agent过早认为整个项目目标已经达成|设置feature list file：根据输入规范，设置一个结构化的JSON文件，其中包括了end-to-end功能描述。|在会话开始时读取feature list file，并选择一个功能进行开发。|
|agent在上轮执行后遗留的环境存在bugs或者未记录的进展|初始化git repo、将过程记入progress notes file|会话开始时读取progress notes file和git commit日志，并且在开发服务器上运行基本测试，以发现任何未记录的bugs。<br>会话结束时，提交最新的git commit并更新progress note files|
|agent过早的认为功能开发已经实现|创建a feature list file|self-verify所有功能。只有在仔细测试后，才能将功能标记为“passing”|
|agent需要花费时间掌握如何运行app|编写一个init.sh脚本，用于启动开发服务器（自动完成依赖安装、环境变量设置、代码编译等）。|会话开始时先读取`init.sh`|
</div>

# 场景实现
<div style="background-color: #f0bdbd; color: black">

业务场景的选择虽然是开放的，但必须遵循2个基本条件：
1. **可直接、准确衡量任务成功与否**
2. **可见的生产力价值导向**

注：所以，任何“xxx分析”、“xxx诊断”、“xxx报告”类的任务，场景上都应该抵制；可验证的场景做好，这些场景自然会有迁移提升。
</div>


# Reference
- 站点
    - [1] 2025 Was Agents. 2026 Is Agent Harnesses. Here’s Why That Changes Everything.
    - [2] What Is an Agent Harness? The Infrastructure That Makes AI Agents Actually Work
    - [3] Effective harnesses for long-running agents
    - [4] https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding
    - [5] Mitchell Hashimoto, My AI Adoption Journey
    - [7] Harness engineering for coding agent users
- 论文
    - [1] Yuxuan Zhang, Haoyang Yu, et. el.. General Modular Harness for LLM Agents in Multi-Turn Gaming Environments.