# SRE / Cloud / Platform / DevOps 岗位要求汇总与提升方案

日期：2026-07-24

依据：今天实际查看的 LinkedIn 与 SEEK 新西兰岗位，包括 ANZ、ASB、TechnologyOne、Pushpay、Datacom、SoftwareOne、XE、Plexure、Foodstuffs、Vector、Trimble、Mantel、Seequent 等公司的 SRE、Cloud、Platform 和 DevOps 职位。

## 一、岗位要求的共同核心

四类岗位的名称不同，但招聘方反复寻找的是同一组能力：

1. 能够构建和维护 AWS/Azure 上的生产系统。
2. 使用 Terraform 等 Infrastructure as Code 管理基础设施。
3. 具备 Kubernetes、容器和云原生平台经验。
4. 能够负责 CI/CD、部署自动化、回滚和发布安全。
5. 能够通过日志、指标和链路追踪定位生产问题。
6. 具备事故响应、根因分析和可靠性改进经验。
7. 理解网络、IAM、安全、数据库和分布式系统。
8. 能够与开发、架构、安全和业务团队协作。
9. Intermediate 通常强调独立交付；Senior 还要求架构决策、指导他人和跨团队影响力。

## 二、SRE 岗位要求

今天查看的 SRE 岗位包括 ANZ、TechnologyOne、Pushpay、ASB，以及 SEEK 上的 Senior SRE 和 SRE 职位。

### 高频要求

- 生产系统监控、告警和可观测性。
- 使用日志、metrics 和 distributed tracing 排查问题。
- CI/CD pipeline 的可靠性和部署故障排查。
- Incident response、service restoration 和 root-cause analysis。
- 自动化重复运维操作，降低人工错误。
- Kubernetes、容器平台、操作系统、网络和脚本能力。
- 与应用团队合作，提高系统可靠性和发布效率。
- 部分岗位包含轮值和工作时间外支持。

### XE 岗位的明确要求

XE 的 Christchurch DevOps Engineer 是今天唯一明确写出 SLO 的职位：

- 3–6 年生产基础设施经验。
- AWS multi-region、IAM、networking、managed services 和 disaster recovery。
- Infrastructure as Code。
- GitHub Actions 或同类 CI/CD pipeline ownership。
- 生产环境 container orchestration。
- Production observability and monitoring stack。
- SLO definition and alerting。
- Production incident response and on-call rotation。
- Release gating、rollback safety 和 24/7 operations。

### SRE 候选人的证据

招聘方通常希望看到：

- 一个实际处理过的生产事故案例。
- 告警出现后如何诊断、缓解、恢复和复盘。
- 具体监控平台，而不只是写“observability”。
- 明确的可靠性指标，例如 availability、error rate、p95 latency、queue age。
- Runbook、incident timeline、RCA 和 corrective actions。
- 如果是正式值班岗位，需要真实 on-call 经验。

## 三、DevOps 岗位要求

今天查看的岗位包括 SoftwareOne、XE、Foodstuffs、Vector、Global Wave 和 SEEK 上的 Senior DevOps。

### 高频要求

- Terraform 或其他 IaC 工具。
- AWS/Azure 生产环境。
- CI/CD pipeline 的设计、维护和故障处理。
- Kubernetes 和容器部署。
- 自动化 build、test、release 和 deployment。
- GitHub Actions、Jenkins、Argo CD 等工具。
- Cloud networking、IAM、secrets 和 security。
- 监控、告警、incident response 和持续改进。

### Intermediate 与 Senior 的区别

Intermediate：

- 可以独立交付 Terraform、pipeline 或 Kubernetes 任务。
- 可以处理常见部署和生产故障。
- 能够参与架构讨论并负责一个技术领域。

Senior：

- 负责平台或 pipeline 的技术方向。
- 制定标准和 reusable patterns。
- 处理复杂生产事故并协调多个团队。
- 指导其他工程师。
- 对安全、成本、可靠性和交付速度做权衡。

## 四、Cloud Engineer 岗位要求

今天查看的岗位包括 Datacom、Trimble、Mantel、AA、Azure Focus 和 Senior Cloud Engineer。

### 高频要求

- AWS 或 Azure 的生产设计和运维。
- VPC/VNet、IAM、networking、compute、storage、database 和 monitoring。
- Terraform 和基础设施自动化。
- CI/CD、containers 和 Kubernetes。
- Security、compliance、backup 和 disaster recovery。
- 成本、性能和容量考虑。

### 岗位差异

- Trimble：云能力符合，但商业开发栈偏 C#/.NET。
- Mantel：除技术外还要求 consulting、client relationship 和 mentoring。
- Azure Focus：认证非常匹配，但需要确认是否偏 Azure platform engineering，还是偏 M365/Windows administration。
- Senior Cloud/Infrastructure：可能包含较多传统网络、企业基础设施和架构治理。

## 五、Platform Engineer 岗位要求

今天查看的岗位包括 Plexure Intermediate Platform Engineer、Datacom Datapay Platform Engineer、Senior Platform Engineer、Seequent Development Platform Engineer 等。

### 高频要求

- 构建开发团队共同使用的 shared platform。
- Terraform、Kubernetes、cloud infrastructure。
- Developer experience 和 self-service tooling。
- CI/CD、GitOps、deployment standards 和 reusable templates。
- 平台监控、可靠性、安全和成本控制。
- 将重复的基础设施操作产品化和自动化。

Platform Engineer 不是简单“维护 Kubernetes”，而是把基础设施能力做成开发团队容易使用的产品。

## 六、你的简历匹配度

### 已有强项

- 4+ 年 Huawei 软件工程经验，符合大部分 intermediate 和部分 senior 岗位年限。
- Java/Spring Boot microservices 和 Kubernetes deployment。
- 分布式系统、跨服务接口和 event-driven architecture。
- 日志、distributed tracing、packet analysis 和 production-oriented troubleshooting。
- AWS Solutions Architect Professional。
- Azure Solutions Architect Expert。
- CKAD 和 AWS Developer Associate。
- Terraform、GitHub Actions、Argo CD、Argo Rollouts 和 canary deployment。
- Cloud Governance X 提供 Azure、Terraform、PostgreSQL、ETL 和工程质量实践。
- 普通话对 Global Wave 等岗位构成直接优势。

### 当前主要缺口

1. CV 没有明确说明是否参加正式 on-call rotation。
2. 没有具体的 production incident、service restoration 和 post-incident review 案例。
3. Observability 只停留在概念，没有突出 Prometheus、Grafana、OpenTelemetry 或具体内部平台。
4. 没有 SLI、SLO、error budget 和 alert design。
5. Terraform 被描述为“正在练习”，弱化了实际能力。
6. 没有量化结果，例如故障定位时间、发布频率、服务数量、吞吐量或可靠性改进。
7. Senior 岗需要的 mentoring、technical leadership 和 architecture ownership 证据不足。
8. CI/CD 和 Kubernetes 经历写成“参与/使用”，缺少 ownership。

## 七、简历立即改写方案

以下内容必须以真实经历为前提。

### Huawei：生产可靠性

可将现有排障描述强化为：

> Investigated production and pre-production incidents across distributed microservices using structured logs, distributed tracing, packet captures, debugging tools, and static analysis, supporting service recovery and root-cause identification.

如果参与过恢复和后续整改：

> Collaborated with cross-functional teams during incident investigation and service restoration, documented root causes and corrective actions, and implemented fixes to reduce recurrence.

如果存在正式轮值：

> Participated in an on-call rotation supporting Kubernetes-based production services, triaging alerts, coordinating incident response, and restoring services through troubleshooting, rollback, and cross-team escalation.

没有正式轮值时，不要使用 `on-call rotation`，改用：

> Provided production-oriented incident investigation and service recovery support during defect resolution and engineering delivery.

### Huawei：平台与交付

> Built and maintained Spring Boot microservices deployed on Kubernetes, contributing to repeatable deployment workflows, release reliability, and long-term platform maintainability.

> Participated in code reviews, technical design, deployment troubleshooting, and cross-team delivery across backend, CLI, and Linux SDK components.

如果实际负责过 pipeline 或发布：

> Improved CI/CD and Kubernetes deployment workflows by automating validation, standardising release steps, and supporting safe rollback during deployment failures.

### Skills

建议新增：

> Observability & Reliability: Azure Monitor, Log Analytics, AWS CloudWatch, distributed tracing, structured logging, packet analysis, incident investigation, root-cause analysis

只有真正完成项目后再添加：

> OpenTelemetry, Prometheus, Grafana, Alertmanager, SLI/SLO design, error budgets, load testing

## 八、Cloud Governance X 提升项目

这是补齐 SRE/Platform 缺口的最佳位置。

### 第一阶段：可观测性

- 在 .NET API、Worker 和 ETL 中加入 OpenTelemetry。
- 采集 traces、metrics 和 structured logs。
- 使用 OpenTelemetry Collector。
- 导出到 Prometheus/Grafana，或 Azure Monitor/Application Insights。

需要监控：

- HTTP request count 和 error rate。
- p50、p95、p99 latency。
- PostgreSQL query latency。
- ETL success/failure rate。
- ETL duration。
- Queue depth 和 oldest message age。
- Worker throughput。
- CPU、memory 和 container restarts。

### 第二阶段：SLO 和告警

定义：

- API availability SLO：99.9%。
- API latency SLO：95% 请求低于 500 ms。
- ETL success SLO：99%。
- Data freshness SLO：95% 的成本数据在 24 小时内同步。
- Queue age SLO：最旧消息低于 5 分钟。

加入：

- 月度 error budget。
- Alertmanager 或 Azure Monitor alerts。
- 高错误率、ETL failure、queue backlog 和 latency alerts。
- 简单 burn-rate alert。

### 第三阶段：事故演练

主动制造并处理一次故障：

- PostgreSQL 暂时不可用。
- Worker 异常退出。
- Queue backlog 持续增长。
- API 新版本导致错误率升高。

留下这些可展示材料：

- Alert screenshot。
- Incident timeline。
- Runbook。
- Root-cause analysis。
- Corrective and preventive actions。
- Rollback 或 recovery 记录。

这不能伪装成真实公司 on-call，但可以证明你理解完整事故流程。

### 第四阶段：容量测试

- 使用 k6 测试并发和吞吐量。
- 记录 p95 latency、error rate 和 saturation point。
- 测试数据库连接池。
- 测试 Worker concurrency 和 queue-drain speed。
- 定义初始扩缩容阈值。

## 九、项目完成后可加入的简历内容

> Instrumented .NET API and worker services with OpenTelemetry, enabling correlated traces, metrics, and structured logs across HTTP requests, ETL execution, PostgreSQL operations, and background processing.

> Built Prometheus metrics and Grafana dashboards for API latency, error rate, ETL reliability, database performance, queue backlog, and worker throughput.

> Defined SLIs and SLOs for API availability, p95 latency, ETL success rate, and data freshness, with alert thresholds and an availability error-budget model.

> Performed load and capacity testing to validate throughput, database connection usage, worker concurrency, and queue recovery behaviour under increasing demand.

> Conducted a controlled incident exercise, producing an operational runbook, incident timeline, root-cause analysis, and corrective actions for service recovery.

## 十、优先级路线

### 现在就可以申请

- Intermediate Platform Engineer。
- Intermediate DevOps Engineer。
- DevOps Engineer。
- Cloud Engineer。
- 非 Senior 的工程型 SRE。

### 完成 observability/SLO 项目后重点申请

- Site Reliability Engineer。
- Platform Engineer。
- Cloud Platform Engineer。
- 对可靠性和可观测性要求较高的 DevOps Engineer。

### 选择性冲刺

- Senior Platform Engineer。
- Senior DevOps Engineer。
- Senior SRE。
- Senior Cloud Engineer。

### 暂不优先

- Principal Engineer。
- SRE Manager。
- 强咨询、售前或组织级架构岗位。

## 十一、最重要的定位调整

简历和 LinkedIn 不要只表达为：

> Software Engineer with cloud certifications

更准确的定位是：

> Backend and platform-oriented engineer with 4+ years of experience building and troubleshooting Kubernetes-based distributed services, supported by AWS, Azure, Terraform, CI/CD, and production reliability skills.

目标不是把自己包装成已经成熟的 Senior SRE，而是证明：

1. 你已经有真实的生产系统和排障基础。
2. 你能够迅速补齐标准 SRE 方法。
3. 你适合 Intermediate Platform/DevOps，并具备向 Senior 发展的能力。
