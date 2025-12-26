# Analysis: Failure-First Audit of Multi-Agent AIOps  
## From Workshop Demo to Production-Grade Reasoning  
*(AWS Bedrock AgentCore Case Study)*

---

## 📌 Overview

This repository presents a **failure-first architectural audit** of a multi-agent AIOps Root Cause Analysis (RCA) workflow built on **AWS Bedrock AgentCore**.

Rather than validating functionality or reproducing a workshop outcome, this project intentionally analyzes **both successful and logically drifted executions** of the same official AWS demo.  
Using **black-box reverse inference**, it identifies systemic failure modes in **multi-agent orchestration (MAS)** that can silently undermine reliability in production environments.

> **Key premise**: In AIOps, an incorrect RCA is often more dangerous than no RCA at all.

---

## 🌍 Scenario Context

This experiment is based on the **AWS Bedrock AgentCore AIOps Workshop – DynamoDB Throttling scenario**.

- A Lambda function intentionally simulates DynamoDB throttling via configuration changes.
- API Gateway surfaces **5XX errors**, while high-level Lambda metrics may appear nominal.
- The agentic workflow is expected to correlate:
  - CloudWatch Logs  
  - X-Ray Traces / Service Graph  
  - CloudTrail Change Events  

Despite identical input conditions, **non-deterministic RCA outcomes** were observed — revealing a class of failures defined in this project as **Logic Drift**.

---

## 🔍 Methodology: Black-Box Reverse Inference

The internal implementation of the agents was **not accessible**.

All analysis was performed by auditing:
- Agent task routing decisions
- Intermediate execution logs
- Evidence selection paths
- Final RCA outputs

A **differential analysis** was conducted between:

- ✅ Successful session — *2025-12-22*  
- ❌ Logic-drifted session — *2025-12-20*

This mirrors real-world scenarios where architects must evaluate **opaque, managed AI systems** without internal visibility.

---

## 🚨 Key Failure Modes Identified

### 1. Hardcoded Anchoring (“God-View” Bias)

**Observation**  
In Task-1 (log analysis), the system implicitly mapped an API Gateway alarm directly to a downstream Lambda log group.

**Insight**  
This hidden anchoring ensured demo success but introduced a brittle coupling between alert source and root cause.

**Production Risk**  
Topology evolution breaks implicit assumptions, causing reasoning chains to fail silently.

---

### 2. Pre-Execution Bias in Planning

**Observation**  
Before any expert agent executed, the Brain Agent finalized **static resource bindings**.

- Successful path: Change detection targeted the Lambda function
  - Query CloudTrail for changes to `aiops-demo-put-item` in last 24 hours
- Drifted path: Change detection locked onto API Gateway
  - Query CloudTrail for changes to `aiops-demo-sample-api` in last 24 hours 

**Insight**  
Once an incorrect resource was selected, downstream agents became blind by design.

**Production Risk**  
Static planning eliminates recovery paths from early reasoning errors.

---

### 3. Hallucinated Certainty (Confidence Without Closure)

**Observation**  
Both successful and failed runs produced **100% confidence RCA outputs**.

**Insight**  
The system equated *absence of conflicting evidence* with *positive confirmation*.

**Production Risk**  
High-confidence but unverified RCA can trigger incorrect automated remediation.

---

### 4. Reasoning Loops & Evidence Starvation

**Observation**  
Expert agents strictly followed the provided `resource_id`.  
When no evidence was found, repeated queries were executed against the same scope.

**Insight**  
The workflow lacks monotonicity checks and cross-step memory of failed attempts.

**Production Risk**  
Compute resources are exhausted without increasing diagnostic certainty.

---

### 5. Logic Closure Gap (Correlation ≠ Causation)

**Observation**  
Change events were detected, but no mandatory **post-change state diff** was performed.

**Insight**  
Detection was treated as a conclusion rather than a trigger for validation.

**Production Risk**  
RCA stops at correlation and fails to confirm physical causality.

---

## 📊 Comparative Audit Summary

| Dimension | Successful Path (2025-12-22) | Logic-Drift Path (2025-12-20) |
|---------|-----------------------------|-------------------------------|
| Initial Planning | Lambda correctly targeted | API Gateway mis-targeted |
| Evidence Chain | Logs + Change form closure | Evidence starvation |
| Confidence | 100% (fact-backed) | 100% (hallucinated) |
| Final Outcome | Root cause verified | Diagnostic dead-end |

---

## 💡 Production-Grade Design Recommendations

### 1. Topology Injection Over Probabilistic Guessing
Inject **AWS AppRegistry / Resource Groups** into the planning phase to anchor reasoning to physical topology.

---

### 2. Dynamic Re-Planning (ReAct Restoration)
Replace one-shot task lists with iterative planning.  
Each step must validate **evidence sufficiency** before proceeding.

---

### 3. Mandatory State Diff for Causal Closure
Change detection must trigger **state comparison** (e.g., configuration diffs), not serve as final proof.

---

### 4. Conflict-Aware Arbitration
Introduce a Critic or Arbiter Agent to challenge conclusions when:
- Logs contradict traces  
- Changes lack observable impact  

High blast-radius actions should enforce **Human-in-the-Loop** approval.

---

## 🧠 Architectural Reflection: Autonomy vs. Rigor

AWS Bedrock AgentCore enables **native agent autonomy** through ReAct-style reasoning.

This audit demonstrates that autonomy alone does not guarantee reliability.

> Robust AIOps is not about restricting intelligence,  
> but about ensuring reasoning always converges back to verifiable facts.

The architect’s role shifts from scripting behavior to **designing guardrails for safe exploration**.

---

## 🏁 Conclusion

This project shows that **workflow engineering dominates prompt engineering**.

Once an evidence chain is broken, no amount of reasoning can recover lost facts.  
Production-grade AIOps systems must treat **reasoning integrity** as a first-class reliability concern.

---

## 📎 Appendix (Separated Artifacts)

Detailed audit logs, execution traces, and evidence matrices are maintained outside this README to preserve clarity and signal quality.
