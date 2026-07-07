# AWS Agentic Football Cup teamChat 指令失效分析：一次 Agent Context Engineering 缺口实践复盘

## 背景简介

AWS Agentic Football Cup 是基于 Amazon Bedrock、Bedrock AgentCore 以及 Strands SDK 构建的智能体实战赛事。

赛事提供 Player Portal 临场指挥能力，允许开发者通过自然语言输入实时战术指令（teamChat），动态调整 AI 球员的攻防行为。

示例：

```text
Fwd shoot on sight
Defend deep
```

然而在实际调试过程中发现：

> Player Portal 可以正常发送 teamChat 指令，但 AI 球员不会根据指令调整行为。

基于公开 sample-agent 实现版本，对 Balanced、Memory、Gateway 三套模板进行源码链路分析后发现：

> 问题并非来自平台通信、模型能力或 Memory 服务，而是 gameState.teamChat 没有经过 Context Construction 层进入 Agent Context。

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

当前 Context Builder 主要提取以下信息。

## 比赛基础状态

包含：

- gameTime
- score
- playMode
- teamId

---

## 球状态

包含：

- ball position
- possession player
- 当前控球归属

---

## 当前控制球员状态

包含：

- player position
- stamina
- distance to ball
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

        ↓

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

很多开发者会认为：

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

Memory 只能保存：

> 已经进入 Agent Context 的信息。


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

# 5. Gateway 模板核心误区

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

> MCP Gateway ≠ Context Injection

即使开启 Gateway，如果 teamChat 没进入 Context，LLM 仍然无法感知战术指令。

---

# 6. Context Injection 修复方案

根据当前架构，最佳修改位置：

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

的唯一转换层。


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

---

# 7. 理论修复效果

完成 Context Injection 后：

- Player Portal 战术指令进入 Agent 推理上下文；
- Memory 模式可以保存已经进入 Context 的战术信息；
- Balanced / Memory / Gateway 三种模板共享修复逻辑；
- 全队球员可以获得临场战术信息。

说明：

由于 workshop 仅开放短暂时效，本人当前已失去赛事环境操作权限，无法完成最终部署验证。

本文方案基于：

- 源码链路分析；
- Agent Context 构造逻辑；
- Strands Agent 调用流程；

进行推导。

---

# 8. Agent 系统架构启示

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

> 数据没有进入正确 Context。


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

导致整个临场指挥链路失效。


未来 Agent Engineering 的核心能力，将逐渐从：

```text
Traditional Debugging
```

转向：

```text
Context Engineering
```

---

# 9. 最终结论

## 各层级链路状态汇总


| 层级 | 状态 |
|---|---|
| Player Portal 通信 | ✅ 正常 |
| GameState 数据传输 | ✅ 正常 |
| Bedrock 模型能力 | ✅ 正常 |
| Strands Memory | ✅ 正常 |
| MCP Gateway 能力 | ✅ 正常 |
| teamChat Context Injection | ❌ 缺失 |


## 核心原因

`gameState.teamChat` 已经成功到达 Agent 服务。

但是当前 Context Builder 未读取该字段，因此：

```text
teamChat

↓

Agent Context
```

链路断裂。


最终结果：

- LLM 无法读取临场战术；
- AI 球员不会调整策略；
- Player Portal 指挥能力无法影响最终决策。


说明：

本人当前无赛事环境操作权限，无法完成最终实战部署验证。

该分析基于：

- sample-agent 源码链路；
- Agent Context 构造逻辑；
- Strands Memory 工作机制。


---

