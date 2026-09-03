# Agent Flight Recorder — Microsoft Agent-a-Thon 最终方案 v2.1

> **定位：Production Reliability for Microsoft Foundry Agents**  
> **目标：冲击 Microsoft Agent-a-Thon Global Top 18**  
> **核心原则：不做普通 Agent Demo，而是做真正的 Agent Reliability Platform。**

---

# 1. 项目名称

# **Agent Flight Recorder**

### Detect, Diagnose, Contain and Safely Release Production AI Agents

一句话：

> **A production reliability layer for Microsoft Foundry agents that detects, diagnoses, contains, and prevents production failures caused by reasoning loops, tool timeouts, retry storms, unsafe side effects, and bad agent releases.**

项目不是：

- 普通 Agent Observability Demo
- 聊天机器人
- RAG Demo
- 多 Agent 炫技 Demo
- 普通 CI/CD Demo
- 普通 Canary Deployment Demo

项目真正定位：

> **AI Agent Reliability Platform**

---

# 2. 产品故事

传统应用发生故障时，SRE 可以依赖日志、指标、Tracing、限流、熔断、金丝雀发布和回滚。

但 Production AI Agent 多了一层新的不确定性：

```text
LLM reasoning
     ↓
tool selection
     ↓
tool arguments
     ↓
retry decisions
     ↓
side effects
```

Agent 可能：

```text
调用错误工具
重复调用同一工具
因为模糊结果陷入 reasoning loop
因为 503 形成 retry storm
执行未经授权的 side effect
新 Agent 版本上线后行为退化
```

所以 Agent Flight Recorder 解决两个阶段的问题：

```text
Before Production
    ↓
Evaluate
    ↓
Canary
    ↓
Reliability Gate
    ↓
Promote / Rollback


During Production
    ↓
Detect
    ↓
Diagnose
    ↓
Contain
```

最终：

> **Agent Flight Recorder protects agents both when they change and when they run.**

---

# 3. 最终技术架构

```text
                              ┌────────────────────┐
                              │        User        │
                              └─────────┬──────────┘
                                        │
                                        ▼
                         ┌──────────────────────────┐
                         │       Demo Web UI        │
                         │                          │
                         │ Chat                     │
                         │ Fault Injection          │
                         │ Incident Timeline        │
                         │ Human Approval           │
                         │ Release Status           │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │ External Router / APIM   │
                         │                          │
                         │ Session-aware Routing    │
                         │ Stable / Canary Routing  │
                         └────────────┬─────────────┘
                                      │
                           ┌──────────┴──────────┐
                           │                     │
                           ▼                     ▼
                 ┌─────────────────┐   ┌─────────────────┐
                 │ Hosted Endpoint v1 │   │ Hosted Endpoint v2 │
                 │      95%          │   │        5%          │
                 └────────┬────────┘   └────────┬────────┘
                          │                     │
                          └──────────┬──────────┘
                                     │
                                     ▼
┌────────────────────── Microsoft Foundry ──────────────────────┐
│                                                               │
│               ┌─────────────────────────────────┐             │
│               │      Cloud Operations Agent     │             │
│               │         Hosted Agent            │             │
│               │                                 │             │
│               │    Microsoft Agent Framework    │             │
│               └──────────────┬──────────────────┘             │
│                              │                                │
│                              ▼                                │
│               ┌─────────────────────────────────┐             │
│               │       Reliability Gateway       │             │
│               │                                 │             │
│               │ correlation_id                  │             │
│               │ tool schema validation          │             │
│               │ canonical args fingerprint      │             │
│               │ retry budget                    │             │
│               │ timeout budget                  │             │
│               │ reasoning-loop detector         │             │
│               │ deterministic policy            │             │
│               │ approval validation             │             │
│               │ idempotency                     │             │
│               └──────────────┬──────────────────┘             │
│                              │                                │
│                ┌─────────────┼─────────────┐                  │
│                │             │             │                  │
│                ▼             ▼             ▼                  │
│       service_status   database_health   dns_resolution       │
│                                                               │
│                              │                                │
│                              ▼                                │
│                     restart_service()                         │
│                      SIDE EFFECT                              │
│                              │                                │
│                     Human Approval                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
                                     │
                                     │ telemetry
                                     ▼
                           ┌───────────────────────┐
                           │     OpenTelemetry     │
                           └──────────┬────────────┘
                                      │
                                      ▼
                           ┌───────────────────────┐
                           │ Application Insights  │
                           └──────────┬────────────┘
                                      │
                     ┌────────────────┴────────────────┐
                     ▼                                 ▼
               Foundry Traces                    Azure Monitor
                     │                                 │
                     └────────────────┬────────────────┘
                                      ▼
                           Reliability Metrics
                                      │
                 ┌────────────────────┼────────────────────┐
                 │                    │                    │
                 ▼                    ▼                    ▼
          Runtime Incidents     Canary Metrics      Release Metrics
                 │                    │                    │
                 └────────────────────┼────────────────────┘
                                      ▼
                           ┌───────────────────────┐
                           │   Reliability Analyst │
                           │                       │
                           │ probable root cause   │
                           │ evidence              │
                           │ containment           │
                           │ remediation           │
                           └───────────────────────┘


Release Path
────────────

GitHub / Release Event
        │
        ▼
   Azure Event Grid
        │
        ▼
 Azure Durable Functions
        │
        ├── Deploy candidate
        ├── Start 5% canary
        ├── Collect reliability signals
        ├── Evaluate release gate
        └── Promote / Rollback
```

---

# 4. 为什么使用 Microsoft Foundry Hosted Agent

最终方案不要只做 Prompt Agent。

Hosted Agent 的价值：

```text
YOUR Python CODE
      ↓
YOUR orchestration
      ↓
YOUR tool dispatcher
      ↓
YOUR reliability policy
      ↓
YOUR retry strategy
      ↓
YOUR loop detection
      ↓
YOUR approval / idempotency
```

运行在：

```text
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

核心原则：

> **LLM 可以提出动作，但不能自己决定授权。**

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
restart_service(service, reason)
```

这个工具不能直接执行。

---

## Agent 2 — Reliability Analyst

平时不参与正常业务执行。

只有发生 Incident 或 Release Regression 时：

```text
Trace
+
Tool history
+
Policy decisions
+
Fault information
+
Reliability metrics
+
Release metadata
        ↓
Reliability Analyst
        ↓
Incident / Release Report
```

输出：

```text
Incident Type
Probable Root Cause
Affected Tool
Affected Agent Version
Evidence
Containment Action
Recommended Remediation
Release Recommendation
```

如果时间不足：

> Reliability Analyst 可以降级为 P1。

---

# 6. Reliability Gateway

这是整个项目的核心。

必须包含：

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

调用链：

```text
Agent
  ↓
Tool Request
  ↓
Reliability Gateway
  ↓
Schema Validation
  ↓
Policy
  ↓
Retry / Timeout Budget
  ↓
Loop Detector
  ↓
Approval / Idempotency
  ↓
Tool
```

核心原则：

```text
LLM decision
≠
authorization decision
```

---

# 7. 四种 Runtime Failure Injection

永久锁定为 4 个，不再增加。

---

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

---

## F2 — Reasoning Loop

这是核心 Demo。

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
Tool returned inconclusive results repeatedly,
causing the agent to retry without acquiring
new information.
```

---

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
afr.retry.count
afr.retry.blocked
afr.tool.failure
```

---

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
+
agent_version
```

再增加：

```text
idempotency_key
```

保证 side effect 只能真正执行一次。

---

# 8. Runtime Reliability Story

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

# **Detect → Diagnose → Contain**

---

# 9. Reliability-Gated Agent Release

这是 v2 新增的高价值能力。

不要做普通 Canary Deployment。

要做：

# **Reliability-Gated Agent Release**

目的：

> 新 Agent Version 不能因为“部署成功”就直接进入 Production，而必须通过真实 Agent Reliability Signals。

---

# 10. Canary Release Architecture

> **实现约束：Foundry Hosted Agent 的单个 endpoint 不支持在多个 Agent version 之间做流量切分。Stable v1 和 Candidate v2 必须部署为两个独立 Hosted Agent endpoints，由外部 Router / APIM 负责 95/5 权重和 session affinity。**

```text
                       Release Candidate v2
                               │
                               ▼
                      Azure Event Grid
                               │
                               ▼
                    Azure Durable Functions
                               │
                  ┌────────────┼────────────┐
                  │            │            │
                  ▼            ▼            ▼
               Deploy       Evaluate       Route
                  │            │            │
                  ▼            ▼            ▼
            Endpoint v2   Eval Dataset   Router / APIM
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                                  95%                  5%
                                    │                   │
                                    ▼                   ▼
                              Hosted Endpoint v1    Hosted Endpoint v2
                                    │                   │
                                    └─────────┬─────────┘
                                              │
                                              ▼
                                     Agent Flight Recorder
                                              │
                                              ▼
                                      Reliability Signals
                                              │
                 ┌────────────────────────────┼──────────────────────────┐
                 │                            │                          │
                 ▼                            ▼                          ▼
          loop rate                  retry exhaustion             timeout rate
                 │                            │                          │
                 ├────────────────────────────┼──────────────────────────┤
                 │                            │
                 ▼                            ▼
          unsafe action rate              P95 latency
                 │
                 ▼
          evaluation score
                 │
                 ▼
          Reliability Gate
                 │
             ┌───┴───┐
             │       │
            PASS    FAIL
             │       │
             ▼       ▼
           25%     Rollback
             ↓
           50%
             ↓
          100%
```

---

# 11. Canary Routing

Foundry Hosted Agent 的**单个 endpoint 不支持 version-to-version traffic splitting**。

因此 Canary 必须放在 Hosted Agent endpoint 之外，由：

```text
External Router / Azure API Management
```

作为前置入口，并把 Stable v1 与 Candidate v2 部署为两个独立 Hosted Agent endpoints。

基本模式：

```text
Stable Endpoint v1    Candidate Endpoint v2
      ▲                       ▲
      │                       │
     95%                      5%
      │                       │
      └───────────┬───────────┘
                  │
                  ▼
          Router / APIM
```

必须注意：

# **Session Affinity**

不能出现：

```text
Conversation A

request 1 → v1
request 2 → v2
request 3 → v1
```

正确行为：

```text
conversation_id
      ↓
stable hash / session affinity
      ↓
same conversation
      ↓
same agent version
```

例如：

```text
Conversation A → Canary v2
Conversation B → Stable v1
Conversation C → Stable v1
```

整个 conversation 生命周期保持一致。

---

# 12. Canary Promotion Policy

建议：

```text
5%
↓
25%
↓
50%
↓
100%
```

每一步都必须经过 Reliability Gate。

示例：

```text
Stage 1:
5%
20 sessions minimum

Stage 2:
25%
50 sessions minimum

Stage 3:
50%
100 sessions minimum

Stage 4:
100%
Production
```

Hackathon 不需要跑海量真实用户。

可以通过：

```text
synthetic traffic
+
failure injection
+
evaluation dataset
```

构造可重复的 release benchmark。

---

# 13. Reliability Gate

示例阈值：

```text
reasoning_loop_rate        <= 1%
retry_budget_exhaustion    <= 2%
tool_timeout_rate          <= 3%
unsafe_side_effects        == 0
containment_rate           >= 95%
p95_reliability_overhead   <= 150 ms
evaluation_score           >= stable baseline
```

注意：

> 最终阈值必须根据真实 Benchmark 调整，不提前编数字。

Release Gate 输入：

```text
Runtime Reliability Metrics
+
Evaluation Results
+
Latency
+
Safety Signals
+
Trace Completeness
```

输出：

```text
PROMOTE
or
ROLLBACK
```

---

# 14. Canary Regression Demo

为了让 Demo 非常直观，Candidate v2 故意引入一个 Regression。

例如：

```text
Stable v1:
dns_resolution inconclusive
→ fallback to service_status

Candidate v2:
dns_resolution inconclusive
→ retry dns_resolution
→ retry
→ retry
→ reasoning loop
```

Canary：

```text
5% traffic → Candidate v2
```

结果：

```text
Reasoning Loop Rate spikes
        ↓
Reliability Gate FAIL
        ↓
Promotion blocked
        ↓
Candidate Endpoint v2 removed from routing
        ↓
Router returns 100% traffic to Stable Endpoint v1
```

UI：

```text
╔═══════════════════════════════════════╗
║ AGENT RELEASE FAILED                  ║
║                                       ║
║ Candidate: cloud-ops-agent-v2         ║
║ Canary Traffic: 5%                    ║
║                                       ║
║ Regression                            ║
║ REASONING_LOOP_RATE                   ║
║                                       ║
║ Stable:     0.5%                      ║
║ Candidate:  8.3%                      ║
║                                       ║
║ Evidence                              ║
║ dns_resolution() repeated 3×          ║
║ identical args fingerprint            ║
║                                       ║
║ Action                                ║
║ AUTOMATIC ROLLBACK                    ║
║                                       ║
║ Status                                ║
║ CONTAINED                             ║
╚═══════════════════════════════════════╝
```

---

# 15. Release Orchestration

使用：

```text
Azure Event Grid
+
Azure Durable Functions
+
Azure Functions
```

职责划分：

## Event Grid

表示：

```text
release.created
candidate.deployed
canary.started
gate.completed
release.promoted
release.rolled_back
```

---

## Durable Functions

负责长事务编排：

```text
Deploy Candidate
      ↓
Wait Ready
      ↓
Route 5% sessions to Candidate Endpoint v2
      ↓
Collect Metrics
      ↓
Evaluate Reliability Gate
      ↓
PASS?
 ├─ Yes → Increase Traffic
 └─ No  → Rollback
```

这个流程不交给 LLM 自由决定。

> **Release promotion is deterministic orchestration, not agent reasoning.**

---

## Azure Functions

负责小型 deterministic activities：

```text
deploy_candidate()
set_router_weight()
query_reliability_metrics()
evaluate_release_gate()
promote_candidate()
rollback_candidate()
```

---

# 16. Release State Machine

```text
CREATED
  ↓
DEPLOYING
  ↓
READY
  ↓
CANARY_5
  ↓
GATE_5
  ├── FAIL → ROLLED_BACK
  ↓ PASS
CANARY_25
  ↓
GATE_25
  ├── FAIL → ROLLED_BACK
  ↓ PASS
CANARY_50
  ↓
GATE_50
  ├── FAIL → ROLLED_BACK
  ↓ PASS
PROMOTED
```

必须保证：

```text
Promotion
≠
LLM decision
```

而是：

```text
Deterministic Reliability Gate
+
Durable Orchestration
```

---

# 17. Release Idempotency

Release 操作一样需要幂等。

例如：

```text
release_id
+
candidate_version
+
target_stage
```

生成：

```text
release_idempotency_key
```

保证：

```text
duplicate Event Grid event
duplicate retry
function retry
orchestrator replay
```

不会：

```text
重复 promotion
重复 rollback
重复修改 routing weight
```

---

# 18. Observability

采用：

```text
Foundry Tracing
+
OpenTelemetry
+
Application Insights
+
Azure Monitor
```

再增加自己的 Reliability Telemetry。

Runtime：

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

Release：

```text
afr.release.created
afr.release.canary.started
afr.release.gate.passed
afr.release.gate.failed
afr.release.promoted
afr.release.rolled_back
afr.release.regression.detected
```

每个 span / event 包含：

```text
trace_id
conversation_id
agent_version
agent_endpoint
release_id
tool_name
tool_call_id
args_fingerprint
attempt
duration_ms
failure_type
policy_decision
canary_stage
release_decision
```

---

# 19. Security

只保留真正重要的部分：

```text
Microsoft Entra
+
Managed Identity
+
RBAC Least Privilege
```

核心原则：

```text
LLM:
"I want to restart service."

Deterministic Policy:
"Are you allowed?"

Human:
"Do I approve?"
```

Release 同样：

```text
Agent:
"I think v2 is healthy."

Reliability Gate:
"Metrics must prove it."
```

禁止：

```text
API key hard-coded in source
Agent 自己决定 authorization
Agent 直接执行危险 side effect
Agent 自己决定 release promotion
```

---

# 20. Terraform

基础设施使用 Terraform。

建议管理：

```text
Resource Group
Foundry Resource
Foundry Project
Model Deployment
Application Insights
Azure Monitor resources
Storage
Managed Identity
RBAC
API Management
Event Grid
Azure Functions / Durable Functions dependencies
Connections
```

Hosted Agent 部署不强求全部 Terraform 化。

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

# 21. Repository Structure

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
│   ├── release/
│   │   ├── canary_router.py
│   │   ├── session_affinity.py
│   │   ├── release_state.py
│   │   ├── gate.py
│   │   ├── metrics.py
│   │   ├── promotion.py
│   │   └── rollback.py
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
├── orchestration/
│   ├── durable/
│   │   ├── release_orchestrator.py
│   │   ├── deploy_activity.py
│   │   ├── metrics_activity.py
│   │   ├── gate_activity.py
│   │   └── rollback_activity.py
│   │
│   └── events/
│       └── handlers.py
│
├── ui/
│
├── infra/
│   ├── providers.tf
│   ├── foundry.tf
│   ├── model.tf
│   ├── monitoring.tf
│   ├── apim.tf
│   ├── eventgrid.tf
│   ├── functions.tf
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
│   ├── failure_benchmark/
│   └── release_benchmark/
│
├── benchmark/
│   ├── runtime/
│   └── release/
│
├── docs/
│   ├── architecture.md
│   ├── reliability-model.md
│   ├── threat-model.md
│   ├── release-model.md
│   ├── benchmark.md
│   ├── incident-examples.md
│   └── demo-script.md
│
└── README.md
```

---

# 22. Runtime Benchmark

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

必须真实运行后才能填写。

---

# 23. Release Benchmark

新增一组：

```text
20 × Good Candidate Release
20 × Reasoning Loop Regression
20 × Retry Regression
20 × Timeout Regression
20 × Unsafe Action Regression

= 100 release simulations
```

比较：

```text
Deploy Directly
vs
Reliability-Gated Canary
```

记录：

| Metric | Direct Release | AFR Canary |
|---|---:|---:|
| Bad release detection rate | | |
| Bad release promotion rate | | |
| Rollback success rate | | |
| Regression detection latency P50 | | |
| Regression detection latency P95 | | |
| User sessions exposed before rollback | | |
| Unsafe actions before rollback | | |
| Release decision accuracy | | |
| Release overhead | | |

这会形成非常强的 Impact 数据。

---

# 24. 最终 Dashboard

主页：

```text
┌──────────────── Agent Flight Recorder ────────────────┐

Runtime Incidents                         4
Contained                                 4
Unsafe Actions Prevented                  1
Retries Prevented                         7

Active Release

Candidate                                 agent-v2
Canary                                    5%
Reliability Gate                          FAILED
Action                                    ROLLED BACK

────────────────────────────────────────────────────────

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
Agent repeatedly called dns_resolution with identical
arguments after inconclusive responses.

Action
Execution terminated.

────────────────────────────────────────────────────────

Recent Release

candidate: agent-v2
stage:     5%
decision:  FAILED
reason:    REASONING_LOOP_RATE

Stable     0.5%
Candidate  8.3%

Action:
AUTOMATIC ROLLBACK
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

# 25. 3 分钟 Demo

## 最终版

| 时间 | 内容 |
|---|---|
| 0:00–0:20 | Problem：Production agents fail differently from normal software |
| 0:20–0:35 | Architecture + Agent Flight Recorder |
| 0:35–0:55 | 正常 Cloud Operations Agent |
| 0:55–1:20 | 注入 Reasoning Loop |
| 1:20–1:40 | Detect + Contain |
| 1:40–1:55 | Foundry / Application Insights Trace |
| 1:55–2:10 | Reliability Analyst RCA |
| 2:10–2:25 | Unsafe Action → Human Approval |
| 2:25–2:45 | Deploy Candidate v2 → 5% Canary → Regression |
| 2:45–2:55 | Reliability Gate FAIL → Automatic Rollback |
| 2:55–3:00 | Benchmark + Closing |

重点演示：

```text
Runtime:
Reasoning Loop
→ Detect
→ Kill
→ RCA

Release:
Candidate v2
→ 5% Canary
→ Regression
→ Gate FAIL
→ Rollback
```

Timeout / Retry Storm / Unsafe Action 用 Dashboard 快速展示。

---

# 26. Demo Opening

> **Traditional observability tells you that an application failed. Agent Flight Recorder tells you why the agent behaved incorrectly, stops it before it causes more damage, and prevents unreliable versions from reaching full production.**

---

# 27. Demo Closing

```text
Evaluate agents.
Release safely.
Observe behavior.
Understand failures.
Contain damage.

Agent Flight Recorder.
```

或更短：

> **Protect agents when they change and when they run.**

---

# 28. Microsoft Foundry 学习路线

不要漫无目的刷文档，只学和项目直接相关的主题。

---

## H1 — Foundry 基础 — 2h

搞懂：

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
→ Model
→ Response
```

---

## H2 — Agent — 2h

做最小 Agent：

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

---

## H3 — Hosted Agent — 3h

主线：

```text
Python Agent
→ Local
→ Container
→ Hosted Agent
→ Foundry Endpoint
```

---

## H4 — Observability — 2h

连接：

```text
Application Insights
```

确认：

```text
Foundry
→ Traces
```

能看到：

```text
request
model
tool
latency
tokens
```

---

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

---

## H6 — Terraform — 2h

从：

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

---

## H7 — Durable Functions / Event Grid — 3h

搞懂：

```text
Event Grid
→ Durable Orchestrator
→ Activity Functions
```

完成：

```text
release.created
→ orchestrator
→ deploy
→ evaluate
→ promote / rollback
```

---

## H8 — External Canary Routing — 2h

搞懂：

```text
two independent Hosted Agent endpoints
backend pool / router targets
weighted routing
session affinity
```

完成：

```text
95%
Hosted Endpoint v1

5%
Hosted Endpoint v2
```

总学习预算：

> **约 17–18 小时**

---

# 29. 项目时间预算

目标：

> **60–70 小时**

---

## Phase 0 — Foundry Learning

```text
17h
```

---

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

---

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

---

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

---

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

---

## Phase 5 — Analyst + UI

```text
7h
```

完成：

```text
Reliability Analyst
Chat
Fault Selector
Incident Panel
Approve / Reject
Release Panel
```

---

## Phase 6 — Runtime Benchmark

```text
6h
```

运行：

```text
100 injected incidents
```

输出：

```text
runtime-results.json
runtime-results.csv
runtime-benchmark.md
```

---

## Phase 7 — Reliability-Gated Canary

```text
8h
```

完成：

```text
external router / APIM routing
two Hosted Agent endpoints
Stable Endpoint / Candidate Endpoint
5% canary
session affinity
Reliability Gate
promotion
rollback
```

---

## Phase 8 — Release Orchestration

```text
5h
```

完成：

```text
Event Grid
Durable Functions
Azure Functions activities
release state machine
release idempotency
```

---

## Phase 9 — Competition Polish

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

# 30. P0 — 必须完成

比赛真正主链：

```text
Hosted Agent
3 read tools
1 side-effect tool

Reliability Gateway

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
Runtime Benchmark

Minimal UI
3-minute Demo
```

如果这些没完成：

> 不准做高级 Canary。

---

# 31. P0.5 — 高价值增强

在 P0 完成之后立刻做：

```text
Hosted Endpoint v1
Hosted Endpoint v2
5% Canary
Reliability Gate
Automatic Rollback
Release Panel
```

这部分就是比赛差异化亮点。

最小实现甚至可以只做：

```text
5% sessions → Candidate Endpoint v2
→ evaluate
→ PASS / FAIL
→ promote / rollback
```

不必一开始就完整：

```text
5 → 25 → 50 → 100
```

---

# 32. P1 — 有时间再做

```text
Reliability Analyst Agent
完整 5→25→50→100 Progressive Delivery
Durable Functions 完整状态机
Event Grid Release Events
Release Benchmark 100 次
Azure Monitor Alerts
高级 Dashboard
CI/CD Automation
```

如果时间不够：

```text
Runtime Detect / Diagnose / Contain
>
Canary
>
高级 Release Automation
```

优先级不能反。

---

# 33. 严格禁止 Scope Creep

以下内容不做：

```text
RAG
AI Search
Vector DB
GraphRAG
Cosmos DB（除非真正必要）
Kubernetes
AKS
Service Mesh
Argo Rollouts
复杂 React 前端
MCP 大合集
5 个 Agent
Voice
Image
Web Search
自己造 tracing platform
完整企业级 CI/CD Platform
```

判断标准：

> **它有没有直接增强 Agent Reliability？**

没有：

> **删。**

---

# 34. 比赛策略

目标：

> **Global Top 18**

围绕三个方向反向设计。

---

## Innovation

```text
Agent-specific failure detection
+
deterministic containment
+
reliability-gated release
+
automatic rollback
```

不是普通 observability。

不是普通 canary。

是：

> **Agent-aware reliability control plane.**

---

## Usability

```text
One-click fault injection
+
clear incident timeline
+
approval UI
+
release panel
+
trace evidence
+
one-click canary simulation
```

---

## Impact

```text
100 injected runtime incidents
+
release simulations
+
measured reduction in redundant calls/retries
+
unsafe action prevention
+
bad release detection
+
automatic rollback
```

---

# 35. 最终项目定位

不要再改变题目。

完整演进：

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
      ↓
Reliability-Gated Release
      ↓
Automatic Rollback
```

最终一句话：

> **Agent Flight Recorder is a production reliability layer for Microsoft Foundry agents that detects, diagnoses, and contains runtime failures while using reliability-gated canary releases to prevent bad agent versions from reaching full production.**

---

# 36. 最终产品能力

```text
                     Agent Flight Recorder

               ┌────────────────────────────┐
               │      BEFORE RELEASE        │
               └────────────────────────────┘

Evaluate
   ↓
Deploy Candidate
   ↓
5% Canary
   ↓
Measure Reliability
   ↓
Reliability Gate
   ↓
Promote / Rollback


               ┌────────────────────────────┐
               │      DURING RUNTIME        │
               └────────────────────────────┘

Observe
   ↓
Detect
   ↓
Diagnose
   ↓
Contain
   ↓
Human Control
```

---

# 37. 最终技术栈

```text
Microsoft Foundry
Microsoft Agent Framework
Hosted Agent

OpenTelemetry
Application Insights
Azure Monitor
Foundry Traces

Azure API Management
Azure Event Grid
Azure Durable Functions
Azure Functions

Microsoft Entra
Managed Identity
RBAC

Terraform

Deterministic Tool Policy
Retry Budget
Timeout Budget
Reasoning Loop Detection
Human Approval
Idempotency

Failure Injection
Runtime Benchmark
Release Benchmark

Reliability-Gated Canary
Automatic Rollback
```

---

# 38. 最终执行顺序

当前学习完成后：

```text
Microsoft Foundry Fundamentals
        ↓
Hosted Agent
        ↓
Tracing
        ↓
Identity
        ↓
Terraform
        ↓
Agent Flight Recorder Runtime Core
        ↓
Four Failure Scenarios
        ↓
100 Runtime Incident Benchmark
        ↓
Minimal UI
        ↓
Reliability-Gated Canary
        ↓
5% Candidate Release
        ↓
Automatic Rollback
        ↓
Release Orchestration
        ↓
3-minute Competition Demo
        ↓
Microsoft Agent-a-Thon Submission
```

---

# 39. Final Decision

项目：

# **Agent Flight Recorder**

第一核心卖点：

```text
Detect
Diagnose
Contain
```

第二核心卖点：

```text
Evaluate
Canary
Gate
Rollback
```

合并成：

# **Runtime Reliability + Safe Agent Delivery**

最终不是：

> 一个会调用工具的 Agent。

而是：

> **一个让 Production AI Agent 可观察、可诊断、可控制、可安全发布、可自动回滚的 Reliability Layer。**

最终 Closing：

> **Protect agents when they change and when they run.**
