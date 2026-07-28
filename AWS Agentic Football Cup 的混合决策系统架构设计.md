# AWS Agentic Football Cup 的 Rule-LLM 混合决策系统架构设计

# 引言

燃情的世界杯赛事在球迷们的欢呼与期待中落下帷幕，而北京的 **AWS Agentic Football Cup** 赛事工作坊也同期展开并圆满结束。

**AWS Agentic Football Cup** 是一个基于自主智能体（Autonomous Agent）的实时多智能体足球仿真环境，通过模拟足球比赛场景，探索多智能体之间的协作、决策与实时交互能力。

![Workshop](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/workshop.jpg)

在赛事工作坊中，各参赛团队通过 Prompt Engineering、行为观察以及持续迭代优化，不断提升智能体足球队的整体表现。

除了 Prompt 优化之外，引入智能体短期状态记忆、增加外部工具能力等方式，同样可以提升智能体系统的表现。

本文将探索另一个优化方向：跳出单个智能体优化视角，重新思考整个 Agent 决策流水线（Decision Pipeline）。

在实时多智能体系统中，决策如何产生、如何路由以及如何执行，会直接影响：

- 决策延迟（Latency）
- 推理效率（Reasoning Efficiency）
- 行为可靠性（Action Reliability）

当前架构中，每个 Player Agent 在固定周期（每 2 秒一个 Tick）独立调用 LLM 决策流程。

![CurrentArchitecture](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/LLM-current-CN.png)

每个智能体接收实时比赛状态，经由大模型推理后输出动作指令（包括移动、传球、射门、滑铲、门将发球等），控制对应场上球员执行。

这套架构赋予智能体灵活的战术推演能力，但在实时运行场景下也带来了一些架构挑战：

- 每一轮决策都需要经过完整的 LLM 推理流程；
- 高时效应急动作受到大模型推理延迟影响；
- 对于结果已经确定的场景，仍然消耗额外推理资源；
- LLM 生成指令可能违反当前环境约束，需要执行前校验。

本文提出一种架构优化方案：通过**确定性规则（Deterministic Rules）+ 选择性 LLM 推理（Selective LLM Reasoning）+ 指令校验（Validation Control）**构建混合决策流水线，重新设计智能体的决策流程。

核心设计原则：

> 确定性场景交由规则处理，复杂场景交由大模型推理，校验机制保障动作可靠执行。
------------------------------------------------------------------------

# 1. 架构提案

整体架构拆解为三层核心模块：快速决策层处理确定性场景、LLM 推理层处理模糊战术判断、指令校验层保证指令合法可用。
由一个决策路由器根据当前场景不确定性，自动分流至对应处理链路。

![ProposedArchitecture](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/ProposedArchtiect_CN.png)

---

# 2. 决策路由逻辑

决策路由器将实时对局状态与预设确定性条件做匹配：一旦命中固定规则模式，直接走快速决策链路；无法通过规则判定的场景，则转交 LLM 做深度分析。
路由判定标准对照表：

![Routing](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/routing_en.jpg)


路由逻辑：

| 场景类型 | 决策路径 | 示例 |
| --- | --- | --- |
| 满足确定性规则条件 | 快速决策层 | 绝佳射门机会、门前紧急拦截 |
| 无法通过规则直接确定 | LLM 推理层 | 逼抢或回撤策略、传球选择、阵型调整 |

---

# 3. 快速决策层 (Fast Decision Layer)

快速决策层的核心价值：在结果唯一确定的场景跳过 LLM 推理，消除大模型输出随机性，同时大幅降低耗时与算力消耗。
该模块解析结构化比赛变量：持球状态、球员坐标、距离、角度、场上角色限制等进行判断，当规则条件满足后，直接输出固定动作指令。只有需要主观战术解读的复杂局面，才继续交由 LLM 处理。

![FastComic](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/fastdecision.jpg)


典型场景 1：绝佳射门窗口

触发条件:

``` text
球员持球 + 射门角度无封堵 + 距离球门在合理射门区间
```

![FastFLOW](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/fast_CN.png)

备注：即便通过提示词对行为规则做了约束，也不属于强硬性限制；加之 LLM 本质为概率性输出，模型决策无法稳定选取最优动作，有可能输出不同行为，错失良机。

典型场景 2：紧急回追拦截

触发条件：

``` text
检测到对方射门动作 + 皮球飞行路线威胁球门 + 本方防守球员可完成拦截
```

快速决策层直接下发固定指令：INTERCEPT 拦截

---

# 4. LLM推理层

LLM 脱离层保留原有 LLM 决策能力，但职责更加聚焦。

它不再处理所有 Decision Tick，而主要处理：

-   高不确定性场景；
-   需要战术理解的问题；
-   动态环境下的策略选择。

例如：

-   此刻应该前场逼抢还是全员回撤？
-   选择短传渗透还是继续带球推进？
-   场外战术指令调整跑位逻辑？

LLM 提供三大能力：战术意图解读、上下文关联决策、动态自适应应对。

规则负责确定性，LLM 负责复杂推理。

---

# 5. 指令校验控制层 (Validation Control Layer )

指令校验层对规则链路或 LLM 链路输出的指令做后置核验。
原有系统仅针对指令格式错误、解析失败做兜底，无法判断指令在当前对局环境下是否具备物理可执行性,格式正确 ≠ 动作一定可执行。

新增的校验层叠加语义校验 + 环境状态校验，强制约束如下规则：
-   持球权限校验；
-   场上角色动作限制（门将 / 前锋 / 后卫动作权限区分）；
-   动作物理可行性；
-   对局状态逻辑自洽性。

![VlationFlow](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/validation-CN.png)


## 真实失效案例：环境约束校验

![VlationComic](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/validation.png)
观测到的失误：门将智能体在未持球状态下，LLM 仍然生成了传球指令。

观测的日志：
![Validation_Layer](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/validation_layer_example.png)

原始对局状态摘要：
``` text
YOUR PLAYER (GK, id=0): pos=(-5.5,0.0) distBall=4.6 hasBall=False
Ball: (-0.9,0.1) held by MY player 4
```
LLM 输出的指令:

``` json
[{"commandType":"PASS","target_player_id":4,"type":"GROUND"}]
```

校验层判定规则：
传球、射门动作前置条件：执行球员必须持有球权（hasBall=True）

校验驳回结果：
``` text
指令驳回
原因：ID为0的门将未持有皮球，无法执行传球动作
```

> 原有兜底机制仅拦截语法错误指令，本条 JSON 格式完全合规，会直接进入执行环节造成逻辑异常。新增语义校验层可彻底规避此类问题。

---

# 6、架构核心收益汇总

| 收益 | 贡献 |
| --- | --- |
| ⚡ 实时响应能力 | 确定性场景绕过 LLM 推理延迟，实现快速响应 |
| 🧠 决策质量提升 | Rules 处理确定性场景，LLM 处理战术不确定性 |
| 🛡 执行稳定性 | Validation 确保动作符合当前环境约束 |
| 💰 资源成本优化 | 减少不必要的 LLM 调用，降低推理开销 |
---

# 7. 总结

多智能体系统的性能优化可以从多个维度展开：既可以提升模型推理与决策质量，也可以重新设计智能体的决策执行链路，从而改善系统效率、行为可靠性与实时响应能力。

在强实时运行环境中，并非所有决策都需要经过完整的 LLM 推理流程。对于确定、简单且时间敏感的场景，可以通过确定性规则快速响应；而对于复杂、动态、需要上下文理解与战术判断的场景，则交由 LLM 发挥推理优势。

AWS Agentic Football Cup 提供了一个探索智能体架构设计与决策优化权衡的实践环境。通过重新审视 Agent 的决策流程，可以进一步构建更加高效、可靠的多智能体系统。

持续探索，持续迭代，让智能体系统持续演进。



