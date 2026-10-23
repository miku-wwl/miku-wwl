# 6周 Harness Engineering → AI Platform Engineering 学习计划

## 一、学习目标

- 学习周期：6周
- 每周投入：8–12小时
- 总学习时间：48–72小时
- 学习方向：Agent Engineering → Harness Engineering → AI Platform Engineering
- 毕业项目：`cloud-native-agent-harness`

### 目标岗位

- AI Platform Engineer
- Agent Infrastructure Engineer
- Agent Runtime Engineer
- AI Platform SRE

---

## 二、核心学习资源

### 1. Learn Claude Code（17讲）

- GitHub：https://github.com/shareAI-lab/learn-claude-code
- 学习时间：Week 1–2
- 目标：从零理解并实现 Agent Harness

核心内容：

- Agent Loop
- Tool Calling
- Permission System
- Hooks
- Subagents
- Skills
- Context Compaction
- Memory
- Task Management
- MCP
- Agent Teams
- Workflow Runtime

实践目标：

从零实现 Mini Harness，掌握：

Model → Tool Call → Execution → Tool Result → Model

### 2. WalkingLabs — Learn Harness Engineering（14讲 + 8项目）

- GitHub：https://github.com/walkinglabs/learn-harness-engineering
- 学习时间：Week 3
- 目标：掌握 Agent Harness 可靠性工程

核心内容：

- Harness Architecture
- State Management
- Session Lifecycle
- Task Recovery
- Persistent Memory
- Verification Mechanisms
- Context Engineering
- Multi-session Continuity
- Graph Engineering

实践目标：

让 Agent 能够可靠地完成长时间、多步骤任务，并在中断后恢复执行。

### 3. LangChain Academy — Deep Agents

- 官方课程：https://academy.langchain.com/courses/foundation-introduction-to-deepagents
- GitHub：https://github.com/langchain-ai/lca-deepagents
- 学习时间：Week 4
- 目标：理解生产级 Agent Runtime

核心内容：

- Deep Agents Architecture
- MCP Integration
- Threads & Checkpointers
- Human-in-the-Loop
- Sandbox Execution
- Filesystem Backends
- Context Management
- Memory
- Subagents
- Agent Deployment

实践目标：

理解成熟 Agent Framework 如何组织 Runtime、State、Checkpoint、Sandbox 和 Context。

### 4. E2B + Firecracker + Kubernetes

学习时间：Week 5

官方学习资源：

E2B：
- https://docs.e2b.dev/
- https://docs.e2b.dev/sandbox

Firecracker：
- https://github.com/firecracker-microvm/firecracker

Kubernetes：
- https://kubernetes.io/docs/concepts/workloads/
- https://kubernetes.io/docs/concepts/security/

核心内容：

- Sandbox Isolation
- MicroVM Architecture
- Sandbox Lifecycle
- Kubernetes Pods & Jobs
- Resource Limits
- Security Boundaries
- Sandbox Provisioning
- Sandbox Cleanup

实践目标：

实现 Harness 对隔离 Sandbox 的生命周期管理：

Create → Execute → Monitor → Pause/Resume → Destroy

### 5. OpenTelemetry + Langfuse

学习时间：Week 6

官方学习资源：

OpenTelemetry：
- https://opentelemetry.io/docs/languages/python/

Langfuse：
- https://langfuse.com/docs/observability/overview
- https://langfuse.com/docs/evaluation/overview

核心内容：

- Distributed Tracing
- Structured Logging
- Metrics
- Agent Execution Traces
- Token Usage Monitoring
- Cost Monitoring
- Failure Injection
- Agent Evaluations
- CI Evaluation Pipeline

实践目标：

建立完整的 Agent Observability 与 Reliability Engineering 能力。

---

## 三、6周学习安排

| 周次 | 学习内容 | 交付成果 |
|------|----------|----------|
| Week 1 | Learn Claude Code S01–S08 | 基础 Agent Loop、Tools、Permissions、Context |
| Week 2 | Learn Claude Code S09–S17 | Mini Harness，支持 Memory、MCP、Subagents |
| Week 3 | WalkingLabs 14讲 + 8项目 | State、Checkpoint、Resume、Verification |
| Week 4 | LangChain Academy Deep Agents | 理解并集成生产级 Agent Runtime |
| Week 5 | E2B + Firecracker + Kubernetes | Sandbox Lifecycle Management |
| Week 6 | OpenTelemetry + Langfuse | Tracing、Metrics、Evals、Failure Injection |

---

## 四、毕业项目：cloud-native-agent-harness

### 项目定位

一个云原生 Agent 执行平台。

不是简单的 LLM Chatbot，而是具备执行控制、安全隔离、状态恢复与可观测性的 Agent Infrastructure。

### 技术栈

| 层次 | 技术 |
|------|------|
| Language | Python |
| API | FastAPI |
| Agent Runtime | Custom Harness / Deep Agents |
| Tool Integration | MCP |
| Sandbox | E2B |
| Isolation Technology | Firecracker |
| Container Orchestration | Kubernetes |
| Observability | OpenTelemetry |
| LLM Monitoring | Langfuse |
| Testing | Pytest |
| CI/CD | GitHub Actions |

### 核心功能

1. Agent Loop
2. Context Management
3. Tool Calling & MCP
4. Policy & Approval
5. Sandbox Isolation
6. Checkpoint & Resume
7. Session Lifecycle Management
8. Observability
9. Token & Cost Monitoring
10. Failure Injection
11. Automated Evaluations
12. CI Evals

### 最终交付

- [ ] GitHub 开源仓库
- [ ] README.md
- [ ] Architecture Diagram
- [ ] Agent Runtime
- [ ] MCP Integration
- [ ] Policy & Approval System
- [ ] Sandbox Lifecycle Management
- [ ] Checkpoint & Resume
- [ ] OpenTelemetry Tracing
- [ ] Langfuse Integration
- [ ] Automated Unit Tests
- [ ] Integration Tests
- [ ] End-to-End Tests
- [ ] Failure Injection Tests
- [ ] CI Evaluation Pipeline
- [ ] Kubernetes Deployment Documentation
- [ ] E2E Demo Video

---

## 五、需要掌握的三个工程层次

### Layer 1 — Agent Engineering

负责：

- Goals
- Planning
- Reasoning
- Tool Selection

核心问题：

Agent 如何决定下一步应该做什么？

### Layer 2 — Harness Engineering

负责：

- Agent Loop
- Context
- Tools
- Permissions
- Checkpoints
- Recovery
- Verification

核心问题：

如何确保 Agent 能够可靠、安全、持续地执行任务？

### Layer 3 — Infrastructure Engineering

负责：

- Sandbox
- Kubernetes
- Database
- Queue
- Telemetry
- Resource Management

核心问题：

如何让大量 Agent 在生产环境中安全、可靠地运行？

---

## 六、与现有项目的关系

现有主要项目：

1. kube-platform-control-plane（KPCP）
2. agentic-intent-gate（AIG）

本学习计划主要用于提升 Agent Runtime 和 Harness Engineering 能力。

学习成果可用于：

- AIG：增强 Agent Loop、Context、MCP、Checkpoint 与 Recovery。
- KPCP：深化 Kubernetes Controller、执行编排、安全审批和平台可靠性。

### 2026-10-16 调整后的实践顺序

1. Agent Development Playground
2. Harness Engineering Playground
3. KPCP Internals Playground（Kubebuilder / Controller）
4. AWS Portability & Approval Execution
5. Cloud-native Agent Infrastructure

注意：

原始6周计划仍然保留。

但实际执行时，应先完成 Agent/Harness 基础，再结合 KPCP 学习 Kubernetes Controller，最后进入 E2B 与云原生 Sandbox 集成。

---

## 七、最终职业目标

利用已有技术背景：

- 4年华为分布式系统开发经验
- AWS Solutions Architect Professional
- Azure Solutions Architect Expert
- Terraform Professional
- CKAD
- Kubernetes / Cloud / Microservices

补齐以下能力：

- Agent Runtime Engineering
- Harness Engineering
- Agent Infrastructure
- Sandbox Security
- Agent Observability
- Reliability Engineering
- Agent Evaluations

最终形成：

Cloud Platform Engineering
        +
Agent Runtime / Harness Engineering
        +
AI Reliability Engineering

目标方向：

**AI Platform Engineer / Agent Infrastructure Engineer / AI Platform SRE**

---

## 八、资料确认说明

- Learn Claude Code：已核实17讲对应的 GitHub 仓库。
- WalkingLabs：历史记录明确确认了仓库地址。
- LangChain Academy：已核实官方课程与配套代码仓库。
- E2B / Firecracker / Kubernetes：当前核实的官方文档入口。
- OpenTelemetry / Langfuse：当前核实的官方文档入口。

其中，Week 5–6 的具体文档 URL 属于本次补全，
并非全部都能从2026年10月15日的原始讨论中逐一确认。
