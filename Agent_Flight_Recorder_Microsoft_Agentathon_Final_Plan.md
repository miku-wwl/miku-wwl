# Agent Flight Recorder — Microsoft Agent-a-Thon 最终方案

> **定位：Production Reliability for Microsoft Foundry Agents**  
> **目标：冲击 Microsoft Agent-a-Thon Asia 区 Top 6**  
> **核心原则：不做普通 Agent Demo，不做 AWS 项目的简单 Azure 移植，而是做真正的 Agent Reliability Platform。**

---

## 1. 项目名称

# Agent Flight Recorder

### Detect, Diagnose and Contain Production AI Agent Failures

一句话：

> **A production reliability layer for Microsoft Foundry agents that detects, diagnoses, and contains reasoning loops, tool timeouts, retry storms, and unsafe side effects.**

项目不是：

- Agent Observability Demo
- Azure version of `strands-powertools-observability`
- 普通聊天机器人
- RAG Demo
- 多 Agent 炫技 Demo

项目真正定位：

> **AI Agent Reliability Platform**

---

## 2. Inspiration

参考项目：

- Darshit Pandya — `strands-powertools-observability`
- AWS Strands Agents
- AWS Lambda Powertools
- AWS X-Ray
- CloudWatch Logs / Metrics / Dashboard

但只借鉴核心思想：

```text
tool-level
logs + traces + metrics
```

最终实现全部重新设计为：

```text
Microsoft Foundry
+
OpenTelemetry
+
Application Insights
+
Agent Reliability Controls
+
Failure Injection
+
Automated RCA
```

不能简单 fork / port AWS 仓。

---

# 3. 最终技术架构

```text
                         ┌────────────────────┐
                         │       User         │
                         └─────────┬──────────┘
                                   │
                                   ▼
                    ┌──────────────────────────┐
                    │      Demo Web UI         │
                    │                          │
                    │ Chat                     │
                    │ Fault Injection          │
                    │ Incident Timeline        │
                    │ Human Approval           │
                    └────────────┬─────────────┘
                                 │
                                 ▼
┌──────────────────── Microsoft Foundry ──────────────────────┐
│                                                             │
│             ┌─────────────────────────────────┐             │
│             │      Cloud Operations Agent     │             │
│             │         Hosted Agent            │             │
│             │                                 │             │
│             │   Microsoft Agent Framework     │             │
│             └──────────────┬──────────────────┘             │
│                            │                                │
│                            ▼                                │
│             ┌─────────────────────────────────┐             │
│             │      Reliability Gateway        │             │
│             │                                 │             │
│             │ correlation_id                  │             │
│             │ tool schema validation          │             │
│             │ canonical args fingerprint      │             │
│             │ retry budget                    │             │
│             │ timeout budget                  │             │
│             │ reasoning-loop detector         │             │
│             │ policy enforcement              │             │
│             │ approval validation             │             │
│             │ idempotency                     │             │
│             └──────────────┬──────────────────┘             │
│                            │                                │
│             ┌──────────────┼───────────────┐                │
│             │              │               │                │
│             ▼              ▼               ▼                │
│     service_status   database_health  dns_resolution        │
│                                                             │
│                            │                                │
│                            ▼                                │
│                    restart_service()                        │
│                     SIDE EFFECT                             │
│                            │                                │
│                   Human approval required                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

                         │ telemetry
                         ▼

              ┌────────────────────────┐
              │     OpenTelemetry      │
              └───────────┬────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │ Application Insights   │
              └───────────┬────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
       Foundry Traces          Foundry Monitor
              │
              ▼
       Incident Context
              │
              ▼
    ┌───────────────────────┐
    │ Reliability Analyst   │
    │                       │
    │ probable root cause   │
    │ evidence              │
    │ recommended action    │
    └───────────────────────┘
```

---

# 4. 为什么选 Microsoft Foundry Hosted Agent

最终方案不要只用 Prompt Agent。

Hosted Agent 的价值：

```text
YOUR Python CODE
        ↓
YOUR orchestration
        ↓
YOUR tool dispatcher
        ↓
YOUR policy
        ↓
YOUR retry strategy
        ↓
YOUR loop detection

运行在
Microsoft Foundry Agent Service
```

这样 Reliability Gateway 不是 PPT 概念，而是真正掌控：

```text
Model
  ↓
tool requested
  ↓
Reliability Gateway
  ↓
allow / block / approve
  ↓
tool execution
```

---

# 5. Agent 只做两个

## Agent 1 — Cloud Operations Agent

负责正常业务任务。

用户：

```text
Check production platform health.
```

Agent：

```text
check_service_status()
        ↓
check_database_health()
        ↓
check_dns_resolution()
        ↓
produce health summary
```

额外危险工具：

```text
restart_service(service)
```

这个工具不能直接执行。

## Agent 2 — Reliability Analyst

平时不参与业务执行。

只有发生 Incident 时：

```text
Trace
+
Tool history
+
Policy decisions
+
Fault information
+
Metrics
       ↓
Reliability Analyst
       ↓
Incident Report
```

输出：

```text
Incident Type
Probable Root Cause
Affected Tool
Evidence
Containment Action
Recommended Remediation
```

如果时间不足，Agent 2 可以降级为 P1。

---

# 6. Reliability Gateway

这是整个项目的核心。

建议包含：

```text
correlation_id
schema validation
canonical args fingerprint
retry budget
timeout budget
reasoning-loop detector
deterministic policy enforcement
approval validation
idempotency
```

核心原则：

> LLM 可以提出动作，但不能自己决定授权。

---

# 7. 四种 Failure Injection

永久锁定为 4 个，不再增加。

## F1 — Tool Timeout

工具：

```text
database_health()
```

故意延迟：

```text
sleep(10)
```

Reliability Gateway：

```text
timeout_budget = 3s
```

结果：

```text
Tool start
   ↓
3 seconds
   ↓
TIMEOUT DETECTED
   ↓
cancel
   ↓
incident
```

Trace：

```text
Agent
 └─ tool.database_health
       duration: 3.02s
       status: ERROR
       failure.type: TOOL_TIMEOUT
```

## F2 — Reasoning Loop

核心 Demo。

让：

```text
dns_resolution()
```

不断返回：

```text
INCONCLUSIVE
```

Agent：

```text
dns_resolution("api")
dns_resolution("api")
dns_resolution("api")
dns_resolution("api")
...
```

生成 fingerprint：

```text
SHA256(
    tool_name
    +
    canonical_json(args)
)
```

检测：

```text
same fingerprint
        ↓
call #1   OK
call #2   OK
call #3   LOOP SUSPECTED
        ↓
CIRCUIT BREAK
        ↓
Agent stopped
```

Incident：

```text
INCIDENT: REASONING_LOOP

Tool:
dns_resolution

Repeated Calls:
3

Containment:
Execution terminated

Probable Cause:
Tool returned inconclusive result repeatedly,
causing the agent to retry without acquiring
new information.
```

## F3 — Retry Storm

工具真实失败：

```text
database_health()
        ↓
HTTP 503
```

Agent：

```text
retry
retry
retry
retry
...
```

设置：

```text
retry_budget = 3
```

超过：

```text
RETRY_BUDGET_EXHAUSTED
```

执行：

```text
BLOCK
+
incident
```

Telemetry：

```text
agent.retry.count
agent.retry.blocked
agent.tool.failure
```

## F4 — Unsafe Side Effect

Agent 判断：

```text
Database unhealthy.
I should restart payment-service.
```

准备执行：

```text
restart_service(
    service="payment-service",
    reason="database connectivity degraded"
)
```

Reliability Gateway：

```text
side_effect = true
        ↓
ActionProposal
```

UI：

```text
╔════════════════════════════════╗
║ Approval Required              ║
║                                ║
║ Tool: restart_service          ║
║ Service: payment-service       ║
║ Reason: database degraded      ║
║                                ║
║       [Approve] [Reject]       ║
╚════════════════════════════════╝
```

Approval 必须绑定：

```text
tool_name
+
canonical_args
+
reason
+
correlation_id
+
expiry
```

再增加：

```text
idempotency_key
```

保证 side effect 只能真正执行一次。

---

# 8. Detect → Diagnose → Contain

这是整个项目必须坚持的产品故事。

普通 Observability：

```text
Agent loops
    ↓
Dashboard tells you
"Agent is looping"
```

Agent Flight Recorder：

```text
Agent loops
     ↓
Detect repeated tool pattern
     ↓
Identify same tool + same args
     ↓
Execution budget exceeded
     ↓
BLOCK
     ↓
Incident generated
     ↓
Reliability Analyst
     ↓
Probable root cause
     ↓
Recommended remediation
```

最终定位：

> **Detect → Diagnose → Contain**

---

# 9. Observability

采用：

```text
Foundry Tracing
+
OpenTelemetry
+
Application Insights
+
Foundry Monitor
```

再增加自己的 Reliability Telemetry：

```text
afr.tool.call
afr.tool.timeout
afr.loop.detected
afr.retry.exhausted
afr.policy.blocked
afr.approval.requested
afr.approval.granted
afr.side_effect.executed
afr.incident.created
```

每个 span/event 包含：

```text
trace_id
conversation_id
tool_name
tool_call_id
args_fingerprint
attempt
duration_ms
failure_type
policy_decision
```

---

# 10. Security

只保留真正重要的三部分：

```text
Microsoft Entra
+
Managed Identity
+
RBAC Least Privilege
```

核心原则：

```text
LLM decision
≠
authorization decision
```

即：

```text
LLM:
"I want to restart service."

Deterministic Policy:
"Are you allowed?"

Human:
"Do I approve?"
```

禁止：

```text
API key hard-coded in source
Agent 自己决定 authorization
Agent 直接执行危险 side effect
```

---

# 11. Terraform

基础设施使用 Terraform。

建议管理：

```text
Resource Group
Foundry Resource
Foundry Project
Model Deployment
Application Insights
Storage
RBAC
Connections
```

Hosted Agent 部署不强求 Terraform。

最终：

```text
Terraform
     ↓
Infrastructure

azd / Foundry SDK
     ↓
Hosted Agent Version Deployment
```

不要为了“100% Terraform”制造额外复杂度。

---

# 12. Repository Structure

仓名：

```text
agent-flight-recorder
```

结构：

```text
agent-flight-recorder/
│
├── app/
│   ├── agent/
│   │   ├── cloud_ops_agent.py
│   │   └── reliability_analyst.py
│   │
│   ├── reliability/
│   │   ├── dispatcher.py
│   │   ├── context.py
│   │   ├── fingerprint.py
│   │   ├── timeout.py
│   │   ├── retry_budget.py
│   │   ├── loop_detector.py
│   │   ├── policy.py
│   │   ├── approval.py
│   │   └── idempotency.py
│   │
│   ├── tools/
│   │   ├── service_status.py
│   │   ├── database_health.py
│   │   ├── dns_resolution.py
│   │   └── restart_service.py
│   │
│   ├── telemetry/
│   │   ├── tracing.py
│   │   ├── metrics.py
│   │   └── events.py
│   │
│   └── faults/
│       └── injection.py
│
├── ui/
│
├── infra/
│   ├── providers.tf
│   ├── foundry.tf
│   ├── model.tf
│   ├── monitoring.tf
│   ├── storage.tf
│   ├── identity.tf
│   ├── rbac.tf
│   ├── variables.tf
│   └── outputs.tf
│
├── deploy/
│   └── azd/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── negative/
│   └── failure_benchmark/
│
├── docs/
│   ├── architecture.md
│   ├── reliability-model.md
│   ├── threat-model.md
│   ├── benchmark.md
│   ├── incident-examples.md
│   └── demo-script.md
│
└── README.md
```

---

# 13. Benchmark

总计：

```text
25 × Tool Timeout
25 × Reasoning Loop
25 × Retry Storm
25 × Unsafe Action

= 100 injected incidents
```

比较：

```text
Baseline
vs
Agent Flight Recorder
```

记录：

| Metric | Baseline | AFR |
|---|---:|---:|
| Failure detection rate | | |
| Containment rate | | |
| Detection latency P50 | | |
| Detection latency P95 | | |
| Duplicate tool calls | | |
| Retries executed | | |
| Unsafe side effects | | |
| Tool calls per incident | | |
| Trace completeness | | |
| Reliability overhead P95 | | |

原则：

> 不提前编 Benchmark 数字。

必须真实运行后才能写真实提升数据。

---

# 14. 最终 Dashboard

目标 UI：

```text
┌──────────────── Agent Flight Recorder ───────────────┐

Incidents                                4

Contained                                4

Unsafe Actions Prevented                 1

Retries Prevented                        7


Recent Incident

REASONING_LOOP

dns_resolution()
████████████████████████

Call #1        240ms
Call #2        219ms
Call #3        226ms
               │
               ▼
        CIRCUIT BREAKER

Root Cause
Agent repeatedly called dns_resolution with
identical arguments after inconclusive responses.

Action
Execution terminated.

────────────────────────────────────────────────────────
```

另一个页面直接展示：

```text
Microsoft Foundry
→ Traces
→ Full OpenTelemetry Trace
```

形成：

```text
Agent Flight Recorder Product View
+
Microsoft Native Observability Evidence
```

---

# 15. Microsoft Agent-a-Thon 3 分钟 Demo

| 时间 | 内容 |
|---|---|
| 0:00–0:20 | Problem：Production AI agents are black boxes when things go wrong |
| 0:20–0:40 | Architecture + Agent Flight Recorder |
| 0:40–1:05 | 正常 Cloud Operations Agent |
| 1:05–1:30 | 注入 Reasoning Loop |
| 1:30–1:50 | 自动 Detect + Contain |
| 1:50–2:10 | Foundry / Application Insights Trace |
| 2:10–2:30 | Reliability Analyst RCA |
| 2:30–2:45 | Unsafe Action → Human Approval |
| 2:45–3:00 | Benchmark + Closing |

重点演示：

```text
Reasoning Loop
→ Detect
→ Kill
→ RCA
```

Timeout / Retry Storm / Unsafe Action 快速展示即可。

---

# 16. Demo Opening

> **Traditional observability tells you that an application failed. Agent Flight Recorder tells you why the agent behaved incorrectly—and stops it before it causes more damage.**

---

# 17. Demo Closing

```text
Observe agents.
Understand failures.
Contain damage.

Agent Flight Recorder.
```

---

# 18. AWS AIP 之后的 Foundry 学习路线

不要漫无目的刷 Foundry 文档，只学 6 个主题。

## H1 — Foundry 基础 — 2h

```text
Foundry Resource
Project
Model Deployment
Project Endpoint
Responses API
```

目标：

```text
Python
→ Foundry
→ model
→ response
```

## H2 — Agent — 2h

做最小 Prompt Agent：

```text
Agent
+
one tool
```

搞懂：

```text
Agent
Conversation
Tool Call
Tool Result
```

## H3 — Hosted Agent — 3h

主线：

```text
Python Agent
→ Local
→ Container
→ Hosted Agent
→ Foundry Endpoint
```

## H4 — Observability — 2h

连接：

```text
Application Insights
```

然后：

```text
Foundry
→ Traces
```

确认能看到：

```text
request
model
tool
latency
tokens
```

## H5 — Identity — 1.5h

搞懂：

```text
Project Managed Identity
Agent Identity
Entra
RBAC
```

完成：

```text
Agent
→ Identity
→ Azure Resource
```

不使用 secret。

## H6 — Terraform — 2h

从零：

```text
terraform apply
```

得到：

```text
Foundry
Project
Model
Application Insights
RBAC
```

总学习时间：

> **约 12 小时**

之后立刻开始项目。

---

# 19. 项目时间预算

目标控制在：

> **50–60 小时**

## Phase 0 — Foundry Learning

```text
12h
```

## Phase 1 — Baseline Agent

```text
6h
```

完成：

```text
Hosted Cloud Ops Agent
3 read tools
1 side-effect tool
```

## Phase 2 — Flight Recorder Core

```text
10h
```

完成：

```text
correlation
fingerprint
timeout
loop detector
retry budget
dispatcher
```

## Phase 3 — Safety

```text
8h
```

完成：

```text
policy
approval
immutable binding
idempotency
```

## Phase 4 — Observability

```text
6h
```

完成：

```text
OpenTelemetry
Application Insights
custom events
custom metrics
Foundry traces
```

## Phase 5 — Analyst + UI

```text
7h
```

完成：

```text
RCA Agent
Chat
Fault Selector
Incident Panel
Approve / Reject
```

## Phase 6 — Benchmark

```text
6h
```

运行 100 次 Failure Injection。

输出：

```text
results.json
results.csv
benchmark.md
```

## Phase 7 — Competition Polish

```text
5h
```

完成：

```text
README
Architecture Diagram
Screenshots
3-minute Demo
Submission Text
```

---

# 20. P0 — 必须完成

```text
Hosted Agent
3 read tools
1 side-effect tool

OpenTelemetry
Application Insights

Tool Timeout
Reasoning Loop
Retry Storm
Unsafe Side Effect

Loop Containment
Retry Budget
Human Approval
Idempotency

Fault Injection
Benchmark

Minimal UI
3-minute Demo
```

---

# 21. P1 — 有时间再做

```text
Reliability Analyst Agent
Azure Functions Remote Tools
Foundry Toolbox
Continuous Evaluation
Azure Monitor Alerts
高级 Dashboard
CI/CD
```

如果时间不够：

> Reliability Analyst Agent 可以砍。

Detect / Diagnose / Contain 主链不能砍。

---

# 22. 严格禁止 Scope Creep

以下内容不做：

```text
RAG
AI Search
Vector DB
GraphRAG
Cosmos DB（除非真正需要）
Kubernetes
AKS
MCP 大合集
5 个 Agent
Voice
Image
Web Search
复杂 React 前端
自己造 tracing platform
```

判断标准：

> **它有没有增强 Agent Reliability？**

没有：

> **删。**

---

# 23. 比赛策略

目标：

> **Asia Top 6**

围绕三个评分方向反向设计：

## Innovation

```text
Agent-specific failure detection
+
deterministic containment
```

## Usability

```text
One-click fault injection
+
clear incident timeline
+
approval UI
+
trace evidence
```

## Impact

```text
100 injected incidents
+
real benchmark
+
measured reduction in redundant calls/retries
+
unsafe action prevention
```

---

# 24. 最终项目定位

最终不要再改变题目。

```text
Observability
        ↓
Reliability Engineering
        ↓
Failure Detection
        ↓
Deterministic Containment
        ↓
Human Control
        ↓
Measured Impact
```

最终一句话：

> **Agent Flight Recorder is a production reliability layer for Microsoft Foundry agents that detects, diagnoses, and contains tool timeouts, reasoning loops, retry storms, and unsafe actions before they become production incidents.**

---

# 25. 执行顺序

当前：

```text
AWS AIP
```

完成之后：

```text
12h New Microsoft Foundry Study
        ↓
Hosted Agent
        ↓
Tracing / Identity / Terraform
        ↓
Agent Flight Recorder
        ↓
50–60h Build
        ↓
100 Failure Benchmark
        ↓
3-minute Competition Demo
        ↓
Microsoft Agent-a-Thon Submission
```

---

# 26. Final Decision

项目：

# **Agent Flight Recorder**

核心卖点：

```text
Detect
Diagnose
Contain
```

核心技术：

```text
Microsoft Foundry Hosted Agent
Microsoft Agent Framework
OpenTelemetry
Application Insights
Foundry Traces
Managed Identity
RBAC
Terraform
Deterministic Tool Policy
Human Approval
Idempotency
Failure Injection
Reliability Benchmark
```

最终目标：

> **不是做一个会调用工具的 Agent，而是做一个能够让 Production AI Agent 可观察、可诊断、可控制的 Reliability Layer。**
