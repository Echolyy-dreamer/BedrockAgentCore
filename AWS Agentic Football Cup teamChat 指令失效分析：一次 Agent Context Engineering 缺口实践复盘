# AWS Agentic Football Cup teamChat 指令失效分析：一次 Agent Context Engineering 缺口实践复盘

## 背景简介

AWS Agentic Football Cup 是基于 Amazon Bedrock、Bedrock AgentCore 以及
Strands SDK 构建的智能体实战赛事。

赛事提供 Player Portal
临场指挥能力，允许开发者通过自然语言输入实时战术指令（teamChat），动态调整
AI 球员的攻防行为。

例如：

``` text
Fwd shoot on sight

Defend deep
```

然而，在实际调试过程中发现：

> Player Portal 可以正常发送 teamChat 指令，但 AI
> 球员不会根据指令调整行为。

经过对 Balanced、Memory、Gateway 三套模板进行源码链路分析后发现：

**问题并非来自平台通信、模型能力或 Memory 服务，而是 teamChat
数据没有进入 Agent 推理上下文（Agent Context Injection 缺失）。**

------------------------------------------------------------------------

# 1. 问题现象与定位

游戏引擎每个 Tick 会发送完整 gameState 数据，包括比赛状态以及 teamChat
战术指令。

数据流：

``` text
Player Portal
      |
      ↓
gameState.teamChat
      |
      ↓
Agent Service
      |
      X
      |
Agent Context
      |
      ↓
LLM Decision
```

通信链路正常。

问题发生在 Agent 数据处理阶段：

> teamChat 被携带到了业务层，但没有被转换成 LLM 可感知的上下文。

------------------------------------------------------------------------

# 2. 根因分析

标准 Agent 执行流程：

``` text
Game Engine
      |
      ↓
gameState payload
      |
      ↓
agent_base.py
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

当前模板中没有读取：

``` python
game_state.get("teamChat")
```

因此进入模型的上下文只有：

``` text
System Prompt

+

Current Game State Summary
```

缺少：

``` text
Coach Tactical Instruction
```

最终导致 AI 按默认策略行动。

------------------------------------------------------------------------

# 3. Memory 模板核心误区

## Memory ≠ Input Parser

Memory 只能保存已经进入 Agent Context 的信息。

正确流程：

``` text
External Data

↓

Context Construction

↓

LLM Reasoning

↓

Memory Persistence
```

如果：

``` text
teamChat

↓

X

↓

Agent Context
```

那么：

``` text
teamChat

↓

X

↓

Memory
```

也不会发生。

因此开启 Memory 并不会自动让 AI 感知 Player Portal 指令。

------------------------------------------------------------------------

# 4. 修复方案

由于三个模板共享公共 Agent 层，只需要修改：

``` text
lib/agent_base.py
```

在状态摘要生成后、LLM 推理前增加：

``` python
coach_instruction = game_state.get("teamChat", "").strip()

if coach_instruction:
    state_summary += (
        "\n\n"
        "## Coach Tactical Command\n"
        "Follow this instruction when making tactical decisions:\n"
        f"{coach_instruction}"
    )
```

------------------------------------------------------------------------

# 5. 修复后的数据流

## 修复前

``` text
Player Portal

↓

gameState.teamChat

↓

agent_base.py

↓

ignored

↓

LLM

↓

Default Strategy
```

## 修复后

``` text
Player Portal

↓

gameState.teamChat

↓

agent_base.py

↓

Context Injection

↓

Strands Agent

↓

LLM Reasoning

↓

Adaptive Team Strategy
```

------------------------------------------------------------------------

# 6. 修复效果

修复后：

-   Player Portal 战术指令实时进入 Agent Context
-   Memory 模式可以保存已经注入的战术信息
-   所有球员通过公共 Agent 层统一生效

------------------------------------------------------------------------

# 7. Agent 系统开发中的架构启示

本次问题表面上是功能未生效，但本质不是传统代码 Bug。

传统软件关注：

``` text
Input
 ↓
Code Logic
 ↓
Output
```

而 AI Agent 系统更加依赖：

``` text
External Data
      |
      ↓
Context Construction
      |
      ↓
LLM Reasoning
      |
      ↓
Tool / Action Execution
      |
      ↓
Memory Persistence
```

任何一个上下文连接断层，都可能导致 Agent 行为与预期不一致。

本案例中：

-   Player Portal 正常发送指令
-   GameState 正常携带数据
-   Agent 正常运行
-   LLM 正常推理

但是：

``` text
teamChat

↓

X

↓

Agent Context
```

导致整个控制链路失效。

未来智能体工程中的大量问题，将从传统的：

> 代码执行错误

逐渐转变为：

> 上下文架构设计问题。

Context Engineering 将成为 Agent 系统设计中的核心能力。

------------------------------------------------------------------------

# 8. 最终结论

  层级                         状态
  ---------------------------- ---------
  Player Portal 通信           ✅ 正常
  GameState 数据传输           ✅ 正常
  Bedrock 模型能力             ✅ 正常
  Strands Memory               ✅ 正常
  teamChat Context Injection   ❌ 缺失

最终根因：

> teamChat 数据已经进入游戏系统，但没有进入 Agent 推理上下文，因此 LLM
> 无法感知临场战术指令。

最终修复：

> 在 Agent 公共入口中，将 gameState.teamChat 显式注入
> state_summary，使其成为 LLM 可见上下文。

------------------------------------------------------------------------

## Keywords

``` text
AWS Agentic Football Cup
Amazon Bedrock
Bedrock AgentCore
Strands SDK
Agent Memory
Context Engineering
Context Injection
LLM Agent Architecture
teamChat
```
