# Failure-First Multi-Agent AIOps Audit  
## From AWS Bedrock AgentCore Workshop to Production-Grade Architecture

---

## 📌 Background

This project is a deep audit of the **AWS Bedrock AgentCore Multi-Agent AIOps Workshop (DynamoDB Throttling scenario)**.

Instead of validating whether the demo “works”, this audit focuses on a more critical question:

> **Can a multi-agent AIOps system recover when its investigation path deviates from reality?**

By comparing **successful** and **logically drifted** investigation workflows, this project identifies structural risks in the current Workshop implementation and proposes concrete design improvements for production environments.

---

## 🎯 Audit Objectives

- Evaluate whether the Workshop truly reflects AgentCore’s **Native ReAct agency**
- Identify systemic bias introduced by static orchestration
- Assess remediation risks caused by incorrect RCA
- Define the boundary between demo-friendly and production-ready AIOps

---

## 🔍 Key Findings

### 1. Hardcoded Anchoring Bias

**Evidence**  
Despite receiving only an API Gateway 5XX alarm, the system directly queried:/aws/lambda/aiops-demo-put-item
**Analysis**  
This mapping is implicitly predefined rather than inferred through agent reasoning.

**Risk**  
- Ensures demo success  
- Fails under topology changes in real systems

---

### 2. Pre-execution Resource Binding

**Evidence**

| Scenario | Task 3 Target |
|---|---|
| Success Path | Lambda |
| Drifted Path | API Gateway |

Resources are bound before any agent executes.

**Risk**  
When no change is found, the investigation cannot replan.

---

### 3. False Certainty Under Incomplete Evidence

**Observation**  
The system outputs **100% confidence** even when no causal evidence exists.

**Root Cause**  
- Log-centric priority  
- No handling of negative evidence

**Impact**  
May trigger incorrect automated remediation.

---

### 4. Evidence Starvation Loop

Agents repeatedly query the same resource without expanding scope.

---

### 5. Missing Causal Closure

**Critical Fact**  
The DynamoDB throttling was **simulated via Lambda environment variables**, not real capacity exhaustion.

**Impact**  
Scaling DynamoDB would not resolve the issue.

This represents a **Silent Failure with Misleading Signals**.

---

## 📊 Path Comparison

| Dimension | Success | Drift |
|---|---|---|
| Planning | Correct | Incorrect |
| Evidence | Closed loop | Broken |
| Confidence | Valid | Illusory |
| Remediation Risk | Low | High |

---

## 💡 Production-Ready Recommendations

1. **Topology Injection** via Resource Groups  
2. **Dynamic Re-planning** (true ReAct loops)  
3. **State Diff Validation** before RCA finalization  
4. **Adversarial Review + Human Approval**

---

## 🧠 Conclusion

AI accelerates reasoning, but system reliability depends on the **lower bound defined by architecture**.

Production-grade AIOps requires not smarter agents, but **deterministic guardrails that preserve engineering truth under uncertainty**.

---
