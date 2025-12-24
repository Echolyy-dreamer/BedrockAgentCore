# Failure-First Multi-Agent AIOps (AWS Bedrock AgentCore)



## 📌 Overview

This repository showcases a deep analysis and engineering reflections from the **AWS Bedrock AgentCore Multi-Agent AIOps Workshop**.  

The goal is not to simply replicate a demo, but to use **black-box Reverse Inference** to uncover **systemic failure modes** in MAS systems and propose actionable design improvements.



**Highlights:**  

- In-depth exploration of **Logic Drift / Evidence Starvation** in Multi-Agent Systems  

- Introduction of **Failure-First Agent Architecture** concept  

- Demonstration of production-ready engineering and safety considerations  


---

## 🌍 Background: Scenario Context

The experiment simulates a **DynamoDB throttling fault** triggering a Lambda function, evaluating an AIOps-driven **Root Cause Analysis (RCA)** pipeline.  



**Challenge:** High-level dashboards may show Lambda as "Green," while API Gateway returns 5XX errors.  

The agentic workflow is expected to correlate **Logs + Traces + CloudTrail**, yet identical inputs occasionally lead to **Logic Drift** — failure to connect the dots despite access to raw data.  



---



## 🔍 Deep Analysis (Black-Box Reverse Inference)

Comparing successful and failed sessions revealed key issues:



### 1. Evidence Contamination & Attention Bias

- Dense upstream text (e.g., X-Ray traces) creates **contextual bias**.  

- In failure cases, the Orchestrator prioritized high-frequency keywords like "Timeout" over explicit "Throttling" logs.  

- Result: wrong `resource_id` passed downstream.  



### 2. Execution Layer Over-Obedience

- Downstream agents act as deterministic executors with no autonomy to challenge upstream instructions.  

- Result: `ChangeDetectionAgent` audited API Gateway instead of the Lambda function where the real issue occurred.  



### 3. Shallow Verification / Logic Closure Gap

- Events were detected but **Impact Validation** was missing (State Diff not fetched).  

- Result: incomplete evidence chain, causality not confirmed.  



### 4. Workflow Engineering > Prompt Engineering

- Even a perfect RCA prompt cannot fix a **starved context**.  

- If upstream agents fail to gather correct facts, the final decision-maker has no path to truth.  



---



## 📊 Comparative Evidence



| Metric | Success Case | Failure Case |

| :--- | :--- | :--- |

| Orchestration Decision | Lambda Function | API Gateway |

| Evidence Quality | Complete Logs + Change | Evidence Starvation |

| RCA Reasoning | Log-Priority Logic executed | Logic defeated by chain break |

| Final Outcome | Root cause identified | Diagnostic dead-end |



---



## 💡 Engineering Reflections: Production-Grade MAS Design



**Failure-First Agent Architecture** core principles:



1. **State vs Event Verification**  

   - Orchestrator must validate `resource_id` against CMDB or AWS Resource Groups  

   - Detected events must fetch **actual State Diff**  



2. **Critic Agent / Conflict Arbitration**  

   - Introduce a Red-Team Agent to challenge RCA findings  

   - Resolve temporal and causal conflicts to prevent logic drift  



3. **Evidence Matrix > Natural Language Summary**  

   - Output a structured evidence matrix quantifying supporting vs contradictory evidence  

   - Enhances traceability and SRE trust  



4. **Loose Autonomy + Logical Checkpoints**  

   - Maintain agent autonomy for edge cases  

   - Insert logical checkpoints to ensure reasoning converges on facts  



5. **Workflow Engineering > Prompt Engineering**  

   - Do not rely on a single prompt for correctness  

   - Ensure upstream/downstream data completeness and closed evidence chains  



---



## 🛠️ Audit Insights vs Official AWS Demo



| Feature | Official Demo | Production-Grade |

| :--- | :--- | :--- |

| Mapping | Probabilistic (LLM Guessing) | **Deterministic (CMDB Guardrail)** |

| Logic Flow | Sequential / Linear | **Cyclic / Adversarial (Critic Agent)** |

| Data Handoff | Natural Language Summaries | **Structured Evidence Matrix** |

| Verification | Event-based | **Impact-based (State Diff / Closure)** |



---



## 🏁 Conclusion

In MAS systems, **automation ≠ intelligence**.  

A reliable RCA process requires **Rigorous Orchestration + Cross-Verification**.





---



