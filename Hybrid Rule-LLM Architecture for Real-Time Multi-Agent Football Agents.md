# Hybrid Rule-LLM Architecture for Real-Time Multi-Agent Decision Making

## Background

AWS Agentic Football Cup is a real-time multi-agent football simulation environment built around autonomous player agents.

In the current architecture, each player agent independently invokes an LLM-based decision process at a fixed interval (approximately every 2 seconds).

The current decision pipeline can be summarized as:
``` text
Game Environment State

        |

        v

Individual Player Agent

        |

        v

LLM Reasoning

        |

        v

Player Command

        |

        v

Game Engine Execution
```

Each agent receives the current game state, performs LLM reasoning, and returns an action command controlling the corresponding player.

This architecture enables flexible tactical reasoning, but it also introduces several challenges in a real-time environment:

Every decision requires an LLM inference cycle.
Time-critical reactions depend on LLM latency.
Deterministic situations consume unnecessary reasoning resources.
LLM-generated actions may require additional validation before execution.

In football simulation, many decisions have fundamentally different characteristics.

For example:

Deterministic and time-sensitive decisions

Examples:

Immediate shooting opportunity.
Emergency interception.
Blocking an incoming shot.
Avoiding impossible actions.

These situations often have clear conditions and do not require complex reasoning.

Strategic and context-dependent decisions

Examples:

Whether to press or retreat.
How to respond to coach instructions.
How to coordinate positioning with teammates.
Whether to maintain possession or attempt an attack.

These situations benefit from LLM reasoning.

Therefore, instead of using LLM reasoning for every decision, a hybrid architecture is proposed:

A hybrid architecture is proposed:

``` text
Fast Decision Layer
        +
LLM Reasoning Layer
        +
Validation Control Layer
```

The goal is to use deterministic systems for certainty and LLMs for uncertainty.

------------------------------------------------------------------------

# 1. Limitations of LLM-only Architecture

Current pipeline:

``` text
    Game State
        |
        v
   LLM Reasoning
        |
        v
   Action Execution
```

The LLM is responsible for tactical reasoning, immediate reactions, and action validity.

However:

  Decision Type         Better Approach
  --------------------- ---------------------
  Emergency reaction    Deterministic rules
  Tactical planning     LLM reasoning
  Constraint checking   Validation rules

Not every decision benefits from LLM reasoning.

------------------------------------------------------------------------

# 2. Proposed Hybrid Architecture

``` text
                 Game Engine State

                        |
                        v

              Decision Router Layer

                        |

        +---------------+---------------+
        |                               |
        v                               v

Deterministic Control Path       LLM Reasoning Path

(Fast Decision Layer)            (Strategic Reasoning)

        |                               |
        +---------------+---------------+

                        |
                        v

             Validation Control Layer

                        |
                        v

                  Action Execution
```

------------------------------------------------------------------------

# 3. Decision Router Layer

The Decision Router determines whether a situation requires LLM
reasoning or can be solved through deterministic rules.

The routing criteria include:

  Situation                          Processing Path
  ---------------------------------- ---------------------
  High-confidence emergency action   Fast Decision Layer
  Time-critical reaction             Fast Decision Layer
  Tactical planning                  LLM Reasoning Layer
  Ambiguous decision                 LLM Reasoning Layer

------------------------------------------------------------------------

# 4. Fast Decision Layer

## Purpose

The Fast Decision Layer handles situations where the optimal action can
be defined with clear conditions.

Goals:

-   Reduce unnecessary LLM calls.
-   Provide immediate reactions.
-   Improve real-time responsiveness.

------------------------------------------------------------------------

## Example Scenarios

### Clear Shooting Opportunity

Situation:

``` text
Player has possession

+

Clear shooting angle

+

Suitable shooting distance
```

Instead of:

``` text
Game State

    |

    v

LLM Reasoning

    |

    v

Shoot
```

The Fast Decision Layer can directly trigger:

``` text
Rule Engine

    |

    v

SHOOT
```

------------------------------------------------------------------------

### Emergency Interception

Situation:

``` text
Opponent shot detected

+

Ball trajectory threatens goal

+

Defender can intercept
```

The Fast Decision Layer can immediately execute:

``` text
INTERCEPT / BLOCK
```

without waiting for LLM reasoning.

------------------------------------------------------------------------

# Fast Decision Layer Examples

## Example 1: Shooting Window Detection

Reserved for implementation example:

``` text
Input:
TODO

Rule:
TODO

Output:
SHOOT
```

------------------------------------------------------------------------

## Example 2: Emergency Defensive Reaction

Reserved for implementation example:

``` text
Input:
TODO

Rule:
TODO

Output:
INTERCEPT / BLOCK
```

------------------------------------------------------------------------

# 5. LLM Reasoning Layer

## Purpose

The LLM handles decisions requiring interpretation and tactical
reasoning.

Examples:

``` text
Should I press or retreat?

Should I pass or carry the ball?

How should I respond to coach instructions?
```

The LLM provides:

-   Tactical reasoning.
-   Strategic decisions.
-   Adaptive behavior.

------------------------------------------------------------------------

# 6. Validation Control Layer

## Purpose

The Validation Control Layer checks whether LLM-generated actions are
valid before execution.

Pipeline:

``` text
LLM Decision

        |

        v

Validation Rules

        |

        +---- Valid ----> Execute

        |

        +---- Invalid --> Fallback
```

------------------------------------------------------------------------

## Example: Invalid Goalkeeper Action

LLM output:

``` json
{
  "action": "PASS",
  "targetPlayer": 3
}
```

Current state:

``` text
Role: Goalkeeper

Ball possession: Opponent
```

Validation result:

``` text
Reject action

Reason:
Player does not have possession
```

Fallback:

``` text
MOVE_TO_POSITION
```

------------------------------------------------------------------------

# Validation Layer Examples

## Example 1: Possession Validation

Reserved:

``` text
LLM Action:
TODO

Validation:
TODO
```

------------------------------------------------------------------------

## Example 2: Role Constraint Validation

Example:

``` text
Goalkeeper cannot perform passing action without possession.
```

------------------------------------------------------------------------

# 7. Benefits

## Latency Control

Critical events bypass LLM:

``` text
Emergency Event

        |

        v

Fast Decision Layer

        |

        v

Immediate Response
```

Complex decisions use:

``` text
LLM Reasoning
```

------------------------------------------------------------------------

## Reliability

Rules provide:

-   Deterministic behavior.
-   Constraint enforcement.
-   Predictable execution.

LLM provides:

-   Reasoning.
-   Adaptation.
-   Strategic flexibility.

------------------------------------------------------------------------

## Observability

Each action has a clear source:

``` text
Action Source:

RULE

or

LLM
```

------------------------------------------------------------------------

# 8. Design Principle

The goal is not:

``` text
Replace Rules with LLM
```

or:

``` text
Replace LLM with Rules
```

The correct architecture is:

``` text
Rules handle certainty.

LLMs handle uncertainty.
```

------------------------------------------------------------------------

# Conclusion

For real-time multi-agent decision systems, an LLM-only architecture may
introduce latency and reliability challenges.

A hybrid architecture combining:

``` text
Fast deterministic rules

+

LLM tactical reasoning

+

Validation constraints
```

provides a better balance between:

-   Reaction speed.
-   Tactical intelligence.
-   System reliability.

LLMs should be used where reasoning is required, while deterministic
layers should handle situations where the correct action is already
known.
