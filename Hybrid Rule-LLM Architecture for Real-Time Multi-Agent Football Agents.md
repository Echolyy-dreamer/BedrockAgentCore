# Beyond Prompt Engineering: Architecting a Hybrid Rule-LLM Decision System for AWS Agentic Football Cup

# Introduction

**AWS Agentic Football Cup** is a real-time multi-agent football simulation environment built around autonomous player agents. During the AWS Agentic Football Cup workshop, teams actively iterated on their agents, experimenting with prompt adjustments and observing how different instructions influenced agent behavior.

While prompt optimization improves agent behavior, another optimization perspective is the decision pipeline itself. In real-time multi-agent systems, how decisions are routed and executed can significantly influence latency, reasoning efficiency, and action reliability.

In the current architecture, each player agent independently invokes an LLM-based decision process at a fixed interval (every 2 seconds). LLM inference is the primary decision mechanism for every decision cycle. 

The current decision pipeline can be summarized as:

```mermaid
flowchart LR

    classDef env fill:#e8f3ff,stroke:#4a90e2,stroke-width:2px
    classDef agent fill:#f5f0ff,stroke:#8b5cf6,stroke-width:2px
    classDef llm fill:#fff4e5,stroke:#f59e0b,stroke-width:3px
    classDef action fill:#f0fdf4,stroke:#22c55e,stroke-width:2px
    classDef issue fill:#ffecec,stroke:#ef4444,stroke-width:2px


    ENV["<b>Game Environment</b><br/>GameState Payload"]:::env


    subgraph Decision["Independent Agent Decision Loops<br/>(Every 2s Tick)"]
        direction TB

        subgraph P1["Player Agent 1"]
            A1["Agent State"]:::agent
            L1["LLM Decision Layer<br/>Reasoning"]:::llm
            C1["Action Example<br/><b>MOVE_TO</b>"]:::action

            A1 --> L1 --> C1
        end


        subgraph P2["Player Agent 2"]
            A2["Agent State"]:::agent
            L2["LLM Decision Layer<br/>Reasoning"]:::llm
            C2["Action Example<br/><b>PASS</b>"]:::action

            A2 --> L2 --> C2
        end


        subgraph PN["Player Agent N"]
            AN["Agent State"]:::agent
            LN["LLM Decision Layer<br/>Reasoning"]:::llm
            CN["Action Example<br/><b>MARK</b>"]:::action

            AN --> LN --> CN
        end

    end


    ENGINE["<b>Game Engine</b><br/>Execution"]:::env


    ENV --> A1
    ENV --> A2
    ENV --> AN


    C1 --> ENGINE
    C2 --> ENGINE
    CN --> ENGINE



    subgraph Challenges["Architectural Challenges"]
        direction TB

        I1["LLM inference<br/>for every decision"]:::issue
        I2["Unnecessary reasoning<br/>in deterministic scenarios"]:::issue
        I3["Generated actions<br/>require validation"]:::issue

    end


    Decision -.-> Challenges
```

Each agent receives the current game state, performs LLM reasoning, and returns an action command (one of MOVE_TO, PASS, SHOOT, SLIDE_TACKLE, GK_DISTRIBUTE ...) controlling the corresponding player.

This architecture enables flexible tactical reasoning, but it also introduces several challenges in a real-time environment:

- Every decision requires an LLM inference cycle.
- Time-critical reactions depend on LLM latency.
- Deterministic situations consume unnecessary reasoning resources.
- LLM-generated actions may require additional validation before execution.

This article explores an architectural optimization approach: improving the agent decision pipeline through a hybrid architecture that combines deterministic rules, selective LLM reasoning, and validation control.

The design principle behind this architecture is:

> **Rules handle certainty. LLMs handle uncertainty. Validation ensures reliability.**

---

# 1. Proposed Hybrid Architecture

The proposed architecture introduces a hybrid decision pipeline with three core layers: a Fast Decision Layer for deterministic scenarios, an LLM Reasoning Layer for ambiguous tactical decisions, and a Command Validation Layer for ensuring command reliability.

A decision router selects the appropriate path based on the uncertainty of each decision scenario.


```mermaid
%%{init: { 
    "flowchart": {
        "nodeSpacing": 50,
        "rankSpacing": 60
    }
}}%%

flowchart TB

    classDef env fill:#e8f3ff,stroke:#4a90e2,stroke-width:2px,color:#1e3a8a
    classDef router fill:#fff4e5,stroke:#f59e0b,stroke-width:3px,color:#78350f
    classDef rule fill:#ecfdf5,stroke:#22c55e,stroke-width:2px,color:#14532d
    classDef llm fill:#f5f0ff,stroke:#8b5cf6,stroke-width:2px,color:#581c87
    classDef validation fill:#fff7ed,stroke:#f97316,stroke-width:2px,color:#9a3412
    classDef command fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#14532d


    STATE["<b>Game Environment State</b><br/>Position • Ball • Players • Context"]:::env


    ROUTER["<b>Decision Router</b><br/>Reasoning Requirement Analysis<br/><br/>Certainty vs. Uncertainty"]:::router


    RULES["<b>Rule-Based Fast Path</b><br/>Deterministic Decisions<br/><br/>
    ✓ Shooting Window<br/>
    ✓ Emergency Interception<br/>
    ✓ Safety Constraints"]:::rule


    LLM["<b>LLM-Based Reasoning Path</b><br/>Tactical Decisions<br/><br/>
    • Press / Retreat<br/>
    • Pass / Carry<br/>
    • Position Adjustment"]:::llm


    VALIDATION["<b>Command Validation Layer</b><br/>Physical & Rule Constraints<br/><br/>
    Command Verification"]:::validation


    CMD["<b>Validated Player Command</b>"]:::command


    ENGINE["<b>Game Engine</b><br/>Execution"]:::env


    STATE --> ROUTER

    ROUTER -->|Deterministic Scenario| RULES
    ROUTER -->|Ambiguous Scenario| LLM

    RULES --> VALIDATION
    LLM --> VALIDATION

    VALIDATION --> CMD
    CMD --> ENGINE
```
---

# 2. Decision Routing

The Decision Router evaluates the current game situation against predefined deterministic conditions. When a situation matches a known rule pattern, the corresponding fast decision path is executed directly. Otherwise, the decision is delegated to the LLM reasoning layer for further tactical analysis.

![Routing](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/routing_en.jpg)

The routing criteria can be summarized as follows:

| Situation | Processing Path | Example |
| --- | --- | --- |
| Matches deterministic rule conditions | Fast Decision Layer | Shooting window, emergency interception |
| No deterministic rule match | LLM Reasoning Layer | Pressing, passing, positioning adjustment |

---

# 3. Fast Decision Layer

The Fast Decision Layer avoids unnecessary LLM reasoning in deterministic scenarios and eliminates generation uncertainty when the optimal action can already be derived from explicit game-state constraints.

The layer evaluates structured game variables such as possession state, player positions, distances, angles, and role constraints. When predefined conditions are satisfied, the corresponding action command is generated directly.

Only scenarios requiring tactical interpretation remain delegated to the LLM Reasoning Layer.

## Example Scenarios

### Clear Shooting Opportunity

![Shoot](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/fast_en.jpg)

Situation:

``` text
Player has possession   +   Clear shooting angle   +   Suitable shooting distance
```
```mermaid
flowchart LR

    classDef condition fill:#e8f3ff,stroke:#4a90e2,stroke-width:2px,color:#1e3a8a
    classDef llm fill:#f5f0ff,stroke:#8b5cf6,stroke-width:2px,color:#581c87
    classDef rule fill:#ecfdf5,stroke:#22c55e,stroke-width:2px,color:#14532d
    classDef action fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#14532d

    STATE["<b>Shooting Opportunity</b><br/><br/>
    Has possession<br/>
    Clear angle<br/>
    Suitable distance"]:::condition


    LLM["<b>LLM Reasoning</b><br/><br/>
    Possible Decisions"]:::llm


    OUTPUT["SHOOT<br/>PASS<br/>MOVE_TO"]:::llm


    RULE["<b>Fast Decision Rule</b><br/><br/>
    Condition Match"]:::rule


    ACTION["<b>Direct Action</b><br/><br/>
    SHOOT"]:::action


    STATE --> LLM
    LLM --> OUTPUT

    STATE --> RULE
    RULE --> ACTION
```

### Emergency Interception

Situation:

``` text
Opponent shot detected  +  Ball trajectory threatens goal  +   Defender can intercept
```

The Fast Decision Layer can immediately execute:

``` text
INTERCEPT
```

---

# 4. LLM Reasoning Layer

## Purpose

While the Fast Decision Layer handles high-confidence deterministic scenarios, the LLM Reasoning Layer handles situations requiring interpretation and tactical reasoning.

Examples:

``` text
Should I press or retreat?

Should I pass or carry the ball?

How should I respond to coach instructions?
```

The LLM provides:

- Tactical interpretation.
- Context-aware decisions.
- Adaptive responses.

---

# 5. Validation Control Layer

## Purpose

The Validation Control Layer performs post-LLM command verification before execution.

The current system already contains fallback handling for invalid command formats or parsing failures.

However, command parsing validation does not guarantee that the generated action is physically executable under the current game state.

The proposed Validation Control Layer adds semantic and environment-level validation.

It enforces deterministic constraints such as:

- possession requirements;
- player role constraints;
- action feasibility;
- game state consistency.

Pipeline:
```mermaid
%%{init: { 
    "flowchart": {
        "nodeSpacing": 50,
        "rankSpacing": 60
    }
}}%%

flowchart TB

    classDef llm fill:#f5f0ff,stroke:#8b5cf6,stroke-width:2px,color:#581c87
    classDef command fill:#e8f3ff,stroke:#4a90e2,stroke-width:2px,color:#1e3a8a
    classDef validation fill:#fff4e5,stroke:#f59e0b,stroke-width:3px,color:#78350f
    classDef execute fill:#ecfdf5,stroke:#22c55e,stroke-width:2px,color:#14532d
    classDef fallback fill:#ffecec,stroke:#ef4444,stroke-width:2px,color:#991b1b


    LLM["<b>LLM Reasoning Layer</b><br/>Generate Action Intent"]:::llm

    CMD["<b>Generated Command</b><br/>SHOOT / PASS / MOVE / ..."]:::command

    PARSER["<b>Command Parser</b><br/>Format & Schema Check"]:::command


    VALIDATION["<b>Validation Control Layer</b><br/><br/>
    Semantic Validation<br/>
    • Possession Check<br/>
    • Role Constraints<br/>
    • Action Feasibility<br/>
    • State Consistency"]:::validation


    EXECUTE["<b>Game Engine</b><br/>Execute Valid Command"]:::execute


    FALLBACK["<b>Fallback Handling</b><br/>Reject / Replace / Safe Action"]:::fallback


    LLM --> CMD
    CMD --> PARSER

    PARSER -->|Valid Format| VALIDATION
    PARSER -->|Parse Error| FALLBACK

    VALIDATION -->|Valid State| EXECUTE
    VALIDATION -->|Invalid Action| FALLBACK
```

## Example Scenario: Environment Constraint Validation

![Validation](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/validation_en.jpg)

Observed failure scenario:

A goalkeeper agent generated a PASS command while the player did not have ball possession.

Raw Log Observation:

![Validation_Layer](https://raw.githubusercontent.com/Echolyy-dreamer/BedrockAgentCore/main/images/validation_layer_example.png)

Summarized State Summary:

``` text
YOUR PLAYER (GK, id=0): pos=(-5.5,0.0) distBall=4.6 hasBall=False
Ball: (-0.9,0.1) held by MY player 4
```

LLM generated command:

``` json
[{"commandType":"PASS","target_player_id":4,"type":"GROUND"}]
```

Validation Check Logic
Rule: Actions PASS, SHOOT require the executing player to hold possession (hasBall=True).

Validation result:

``` text
Reject command
Reason: Goalkeeper (id=0) does not possess the ball; PASS cannot be executed.
```

>The existing fallback mechanism only handles syntactic failures. Since the command format is valid, the existing fallback mechanism is not triggered.The command proceeds to execution despite violating game-state constraints.

# 6. Key Architectural Benefits

| Benefit | Contribution |
| --- | --- |
| ⚡ Real-Time Responsiveness | Deterministic scenarios bypass LLM inference latency and receive immediate responses. |
| 🧠 Decision Quality | Rules handle high-confidence situations, while LLM reasoning focuses on contextual tactical decisions. |
| 🛡 Execution Reliability | Validation ensures generated commands are executable under current game state and role constraints. |
| 💰 Resource Efficiency | Avoiding unnecessary LLM calls reduces token consumption and inference overhead. |

# 7. Conclusion

Improving multi-agent systems requires exploring multiple dimensions. Different optimization approaches address different challenges, from reasoning quality to decision efficiency and execution reliability.

In real-time environments, not every decision requires the same level of intelligence. Some situations benefit from fast and deterministic responses, while others require contextual reasoning from LLMs.

The AWS Agentic Football Cup provides a practical environment for exploring these trade-offs. Rethinking the decision pipeline itself can reveal new opportunities for building more efficient and reliable multi-agent systems. 

Keep exploring, keep experimenting, and have fun building smarter agents.



