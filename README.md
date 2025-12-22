# Failure Mode Analysis: Logic Drifting in Multi-Agent AIOps Workflows

## 📌 Overview
This project evaluates an automated Root Cause Analysis (RCA) pipeline built on **AWS Bedrock Agents**. Operating in a "Black Box" environment (containerized agents), I utilized **Reverse Inference** based on execution logs to identify a critical architectural flaw.

This repository documents how identical fault signals can lead to inconsistent outcomes due to **Contextual Bias** in the orchestration layer.

---

## 🔍 Investigation: Black-Box Reverse Inference
Since the internal logic of the containerized agents is inaccessible, I performed a **Differential Analysis** between successful and failed diagnostic sessions.



### The Observed "Logic Drift"
By auditing the `task_description` and parameter extraction patterns, the following failure chain was inferred:

1.  **Contextual Contamination**: 
    In the failed session (2025-12-20), the findings from `Task-2 (TraceGraphAgent)` were heavily weighted towards "API Gateway Integration" and "Timeouts." 
2.  **Orchestrator Drift**: 
    The Brain Agent (Orchestrator) exhibited a **textual bias**. It prioritized the high-frequency keywords in Task-2's Summary over the explicit error logs in Task-1, leading it to extract the wrong `resource_id`.
3.  **Strict Parameter Dependency**: 
    `Task-3 (ChangeDetection)` acted as a deterministic executor. Once the Orchestrator routed the wrong Resource ID, Task-3 was effectively "blinded" to the actual root cause, as its search scope was locked by the upstream drift.

---

## 📊 Comparative Case Study

| Metric | Case A: Observed Success | Case B: Observed Failure |
| :--- | :--- | :--- |
| **Session Date** | 2025-12-22 | 2025-12-20 |
| **Upstream Findings** | Lambda Errors + APIGW 5XX | Lambda Errors + APIGW 5XX |
| **Inferred Brain Decision** | Prioritized Log Evidence | Prioritized Trace Summary Bias |
| **Outcome** | **Success** (Root Cause Identified) | **Failure** (Search Scope Mismatch) |

---

## 💡 Engineering Reflections & Best Practices
1.  **The Peril of Opaque Orchestration**: 
    In multi-agent systems, the "Brain" often lacks a formal conflict-resolution protocol. Without explicit priority rules (e.g., `Logs > Traces`), the system's accuracy becomes stochastic based on the LLM's attention mechanism.
    
2.  **Decoupling Observation from Inference**: 
    To prevent logic drift, intermediate agents should ideally report raw observations. When agents provide interpretative summaries (like Task-2's timeout theory), they risk "hijacking" the orchestrator's planning phase.

3.  **Reverse Engineering as a Debugging Tool**: 
    In modern AI-native stacks where source code is often hidden behind APIs or containers, **log-based reverse inference** is the most effective way to audit system reliability.
