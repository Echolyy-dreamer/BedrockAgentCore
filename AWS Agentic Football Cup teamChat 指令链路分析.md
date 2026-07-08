# AWS Agentic Football Cup teamChat 指令链路分析：从 Context Injection 缺口到 Agent 指令作用域设计

## 背景简介

AWS Agentic Football Cup 是基于 Amazon Bedrock、Bedrock AgentCore 以及 Strands SDK 构建的智能体实战赛事。

赛事提供 Player Portal 教练临场指挥能力，允许参赛者通过自然语言输入实时战术指令（teamChat），动态影响 AI 球员的攻防决策。

下图为赛事官方 Player Portal 教练指挥界面。
教练可在底部输入框编辑战术指令，并通过 `Shout!` 按钮实时下发。
该指令会随比赛状态一起传递至后端 `gameState.teamChat` 字段，由 Agent 服务接收。

![AWS Agentic Football Cup Player Portal](https://github.com/Echolyy-dreamer/BedrockAgentCore/raw/main/Player%20Portal.jpg)

在现场观赛和调试过程中，我注意到许多参赛者都会在 Portal 中输入：

```text
shoot

defend

press
```
等实时战术指令。

这引发了一个关于 Multi-Agent 指令作用域的问题：

- 在多个独立 Player Agent 协同决策的架构中，一条 Coach Command 如何影响每个球员？
- 它是通过中央控制器将指令路由给目标球员，还是通过全局广播让 Agent 根据自身 Context 自主调整行为？

带着这个关于 Multi-Agent Tactical Scope（多智能体战术作用域） 的问题，我进一步分析了官方 sample-agent 中 Balanced、Aggressive、Defensive、Memory、Gateway 等模板的源码调用链。


然而分析过程中发现：

> 当前 sample-agent 实现中，gameState.teamChat 虽然已经到达 Agent 服务，但没有经过 Context Construction 层进入 Agent Context，导致Player Portal 可以正常发送 teamChat 指令，但 AI 球员不会根据指令调整行为。

---

# 1. Agent Context Pipeline 源码分析

当前 Agent 执行流程：

```text
Game Engine
      |
      ↓
gameState payload
      |
      ↓
agent_base.py / gateway invoke
      |
      ↓
summarize_state()
      |
      ↓
Agent(state_summary)
      |
      ↓
LLM reasoning
      |
      ↓
Player action
```

核心调用链：

```python
game_state = prompt_data.get("gameState", {})

state_summary = summarize_state(
    game_state,
    team_id,
    effective_pid,
    position_label
)

response = agent(state_summary)
```

关键点：

Agent 接收的是：

```text
state_summary
```

而不是完整：

```text
gameState
```

因此：

> 任何未进入 state_summary 的字段，都不会进入 LLM 推理上下文。

---

# 2. 当前 summarize_state() 提取范围

在当前 sample-agent 架构中：

summarize_state()

承担：

Raw GameState

↓

Agent Input Context

的转换职责。

它是整个 Agent Context Pipeline 中的重要 Context Builder。

当前主要提取：

## 比赛基础状态

包含：

- 比赛时间
- 比分
- 比赛模式
- 队伍 ID

---

## 球状态

包含：

- 球位置
- 控球信息

---

## 当前控制球员状态

包含：

- 控制球员场上位置 
- 体力值 （stamina）
- 距离足球距离 
- 是否持球
- 与对方球门距离

---

## 队友状态

包含：

- 队友位置
- 球员编号
- 进攻相关距离信息

---

## 对手状态

包含：

- 对手位置
- 距离己方球门距离
- 距离当前球员距离


当前状态映射：

```text
gameState

├── ball              ✅ extracted
├── players           ✅ extracted
├── score             ✅ extracted
├── gameTime          ✅ extracted
├── playMode          ✅ extracted
└── teamChat          ❌ not extracted
```

因此当前 Agent Context 实际描述的是：

```text
Current Match Situation

+

Player Observation

+

Opponent Information
```

但缺少：

```text
Coach Tactical Instruction
```

---
![summarize_state Context Builder](https://github.com/Echolyy-dreamer/BedrockAgentCore/raw/main/summarize_state.png)

该代码片段展示当前 Context Builder 的字段映射逻辑：
ball、score、gameTime、playMode、players 均被提取，
但未发现 teamChat 字段读取逻辑。

# 3. teamChat 指令失效根因分析

当前模板代码链路中，没有发现任何逻辑将：

```text
gameState.teamChat
```

转换进入：

```text
Agent Context
```

缺失链路：

```text
gameState.teamChat

        ↓ X

Context Construction

        ↓

Agent Context
```

最终传递给 LLM 的上下文：

```text
System Prompt

+

Current Game State Summary
```

缺少：

```text
Coach Tactical Instruction
```

因此 AI 球员只能依据默认策略执行动作。

---

# 4. Memory 模板核心误区

## Memory ≠ Input Parser

一个常见认知误区是：

> 开启 Memory 后，Agent 会自动记住 Player Portal 下发的指令。

实际上并不会。


Strands Memory 的工作流程：

```text
External Data

↓

Context Construction

↓

LLM Reasoning

↓

Memory Persistence
```

Memory 主要作用是对已经进入 Agent 交互流程的信息进行存储和检索，它不会替代前置的 Context Construction。


当前：

```text
teamChat

↓

X

↓

Agent Context
```

所以：

- teamChat 不会进入 LLM；
- teamChat 不会进入 Memory；
- Memory 无法自动修复这个问题。

---

# 5. Gateway 模板能力边界

Gateway 模板增加：

```python
with mcp_client:
    response = agent(state_summary)
```

MCP Gateway 提供的是：

```text
Agent

↓

Tools
```

能力扩展。

但它不会自动完成：

```text
gameState

↓

Context Construction

↓

Agent Context
```

因此：

- 即使开启 Gateway，如果 teamChat 没进入 Context，LLM 仍然无法感知战术指令。

---

# 6. Context Injection 修复方案

虽然 agent_base.py 是调用入口，但 teamChat 属于 GameState 到 Context 的转换逻辑，因此更适合在 summarize_state() 层完成注入。

根据当前架构，建议修改位置：

```text
state.py

summarize_state()
```

原因：

这里是：

```text
Raw Game State

↓

LLM Context
```

的主要转换层。


增加：

```python
coach_instruction = game_state.get("teamChat", "").strip()

if coach_instruction:
    lines.append("")
    lines.append("## Coach Tactical Command")
    lines.append(
        "Follow this instruction when making tactical decisions:"
    )
    lines.append(coach_instruction)
```
![Context Injection Fix](https://github.com/Echolyy-dreamer/BedrockAgentCore/raw/main/Adding.png)

> 该方案保持原有 Agent 输入结构不变，仅扩展 Context Builder 输出内容。
修复后数据流：

```text
Player Portal

↓

gameState.teamChat

↓

summarize_state()

↓

Context Injection

↓

Strands Agent

↓

LLM Reasoning

↓

Adaptive Team Strategy
```

完成 Context Injection 后：

- Player Portal 战术指令进入 Agent 推理上下文；
- Memory 模式可以保存已经进入 Context 的战术信息；
- Balanced、Aggressive、Defensive、Memory、Gateway 各模板共享修复逻辑；
- 全队球员可以获得临场战术信息,并在自身状态和局部观察基础上进行决策。

说明：

由于 workshop 仅开放短暂时效，本人当前已失去赛事环境操作权限，无法完成最终部署验证。

本文方案基于：

- 源码链路分析；
- Agent Context 构造逻辑；
- Strands Agent 调用流程；

进行推导。

---

# 7. teamChat 指令作用域设计

Context Injection 修复解决的是：

```text
teamChat

↓

Agent Context
```

的数据链路问题。

但它不等于：

```text
teamChat

↓

指定球员执行
```

当前架构中：

每个球员拥有独立 Agent。

例如：

```text
MY_PLAYER_ID = 3

POSITION_LABEL = "FWD1"
```

这些信息只用于描述：

当前 Agent 控制的是哪一个球员。

但所有 Agent 接收到的是同一个：

```text
gameState.teamChat
```

因此当前机制天然更适合：

全局战术广播

例如：

```text
Defend deep

Press high

Keep possession

Attack left wing
```

这类指令用于改变整体战术风格。

类似真实比赛中的场边喊话：

所有球员接收到同一战术意图，
每个 Agent 根据自身位置、球权、视野和状态自主调整行为。

---

# 8. 单球员指令与实时延迟权衡

如果希望支持教练采用点名式口语喊话，如真实比赛里场边直接呼喊 「FWD1 抓住机会立刻射门」：

```text
FWD1 shoot now

Player 3 move forward

GK stay back
```

则需要增加额外机制, 例如采用 Agent-side Command Filtering（Agent 自主指令过滤机制)，所有 Agent 接收同一个教练指令，由每个 Agent 根据自身 Context 判断是否与自己相关：

```text
Coach Command

↓

All Agents

↓

Self Filtering

↓

Individual Action
```

例如：

```text
FWD1 shoot now

↓

Agent-side Command Filtering

↓

POSITION_LABEL == "FWD1"

↓

Only FWD1 Agent apply execution 
```

这种设计可以提高指令精确度。 
但是在实时足球环境中，需要权衡：

```text
- 指令精确度
- 决策延迟
```

两者之间需要进行实时性权衡。

更精确的单球员控制能力，可能增加 Agent 决策路径长度。

对于实时比赛：

低延迟、高可靠性的全局广播机制，可能比复杂的实时指令路由更加稳定。

因此当前 Context Injection 方案主要实现：

```text
Coach Command Awareness
```

而不是：

```text
Individual Player Command Routing
```

---

# 9. Agent 系统架构启示

传统软件：

```text
Input

↓

Code Logic

↓

Output
```


AI Agent：

```text
External Data

↓

Context Construction

↓

LLM Reasoning

↓

Tool / Action Execution

↓

Memory Persistence
```


Agent 系统中的大量问题，并不一定表现为：

> 代码执行失败


而可能表现为：

> 数据没有进入正确 Context Boundary。


本案例：

- Player Portal 正常发送指令；
- GameState 正常携带 teamChat；
- Agent 服务正常运行；
- LLM 正常完成推理；


但是：

```text
teamChat

↓

X

↓

Agent Context
```

导致当前 sample-agent 实现中的临场教练指挥链路无法生效。


未来 Agent Engineering 的核心能力，将逐渐从：

```text
Traditional Debugging
```

转向：

```text
Context Engineering
```

---
**> 在 Agent 系统中，模型能力决定推理上限，而 Context Pipeline 决定模型能够感知什么。当关键输入未跨越上下文边界时，再强大的推理模型也无法基于不存在的信息做出正确决策。**


# 10. 最终结论

## 各层级链路状态汇总


| 层级 | 状态 |
|---|---|
| Player Portal 通信 | ✅ 正常 |
| GameState 数据传输 | ✅ 正常 |
| Context Builder 字段映射 | ❌ 缺少 teamChat |
| Bedrock 模型能力 | ✅ 正常 |
| Strands Memory | ✅ 正常 |
| MCP Gateway 能力 | ✅ 正常 |
| teamChat Context Injection | ❌ 未实现 |


## 核心原因

`gameState.teamChat` 已经成功到达 Agent 服务。

但是当前 Context Builder 未读取该字段，因此：

```text
teamChat

↓ X

Agent Context
```

链路断裂。


最终结果：

- LLM 无法读取临场战术；
- AI 球员不会调整策略；
- Player Portal 指挥能力无法影响最终决策。

---

# 参考链接

Official Agentic Football Cup Sample Workshop Workshop:
https://catalog.workshops.aws/agentic-football/en-US 

# 作者简介

**Echo Liu**

Cloud & AI Agent 技术爱好者一枚.


---

