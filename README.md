# Failure Mode Analysis: Logic Drifting in Multi-Agent AIOps Workflows

## 📌 Overview
This project evaluates an automated Root Cause Analysis (RCA) pipeline built on **AWS Bedrock Agents**. By conducting cross-temporal log analysis (comparing sessions from 2025-12-20 and 2025-12-22), I identified an **architectural flaw** where the orchestration logic drifts due to "misleading recommendations" from upstream agents.

This repository documents how a perfectly functioning LLM can still yield incorrect results due to weak **Parameter Routing** strategies in a multi-agent ecosystem.

---

## 🔍 Core Discovery: The "Logic Black Hole"
**"Data accuracy ≠ Result accuracy. The bottleneck lies in the Robustness of Orchestration."**

In a simulated **DynamoDB Throttling** scenario, the system demonstrated inconsistent diagnostic outcomes despite receiving identical failure signals. I traced the root cause to a structural defect in how the **Orchestrator (Brain)** processes conflicting evidence between Log and Trace agents.

### Failure Propagation Chain
1.  **Upstream Bias (Task-2 TraceAgent)**: 
    The TraceAgent reported a "Success" for the Lambda function based on high-level metrics, while ignoring the internal exceptions found in logs. It produced a misleading recommendation: *"Investigate API Gateway integration."*
2.  **Orchestration Drift (The Brain)**: 
    The Orchestrator over-weighted the "Recommendation" field from Task-2. Consequently, it extracted the **wrong Resource ID** (API Gateway) to pass down to the next task, ignoring the explicit error logs from Task-1.
3.  **Downstream Execution Blindness (Task-3 ChangeDetection)**: 
    As a "strict executor," Task-3 was constrained by its prompt to only audit the resource provided by the Brain. It lacked the autonomy to cross-check the actual error logs, leading to a "No Changes Found" dead-end.

---

## 📊 Comparative Evidence (Log-Driven)

| Metric | Case A: Stochastic Success | Case B: Deterministic Failure |
| :--- | :--- | :--- |
| **Log Timestamp** | 2025-12-22 | 2025-12-20 |
| **Primary Evidence** | Lambda Logs: 78 Throttling Errors | Lambda Logs: 78 Throttling Errors |
| **Brain's Decision** | Route to **Lambda Function** | Route to **API Gateway** |
| **Root Cause Found** | **YES** (UpdateFunctionConfiguration) | **NO** (Queried the wrong resource) |
| **Failure Reason** | N/A | **Logic Drift** via Task-2 Recommendation |

---

## 🛠️ Proposed Architectural Fixes
To mitigate such failures in production-grade AI Agents, I propose the following enhancements:

* **Conflict-Aware Arbitration**: Implement an arbitration layer in the Orchestrator. If **Logs (Task-1)** report explicit errors while **Traces (Task-2)** report success, the system must prioritize the Log-source resource.
* **Decentralized Resource Discovery**: Grant the ChangeDetection agent (Task-3) the authority to scan all raw findings from upstream tasks, rather than relying on a single filtered parameter from the Brain.
* **Fact-Only Communication Protocol**: Restrict intermediate agents from outputting "Recommendations" to prevent them from "commanding" the Orchestrator. Agents should only report **Observable Facts**.

---

## 💡 Professional Takeaways for Interviewers
1.  **Deep Architectural Insight**: I don't just "run demos"; I stress-test the communication protocols between agents to identify single points of failure.
2.  **Observability Expertise**: Proficient in using `traceId` and `timestamps` to reconstruct the **Thinking Process** of LLMs across distributed system logs.
3.  **System Robustness Thinking**: I apply "Defensive Prompting" and "Audit Trails" to ensure that downstream agents remain robust even when upstream data is noisy or contradictory.

---

### 📂 Repository Structure
* `/logs`: Original JSON log files capturing the failure/success cases with timestamps.
* `/analysis`: Detailed walkthrough of the parameter extraction flaw.
* `/solution`: Optimized System Prompts for Task-3 to enable cross-task verification.
