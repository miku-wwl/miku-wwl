# Pushpay Senior Site Reliability Engineer  
## 岗位中英文对照与解读

> **岗位地点：** Auckland, New Zealand  
> **工作方式：** Hybrid（混合办公）  
> **雇佣类型：** Full-time（全职）  
> **硬性条件：** 必须拥有新西兰合法工作权  
> **岗位定位：** 软件工程型 SRE / 平台可靠性工程 / 高级工程影响力岗位

---

## 一、岗位核心定位

### 原文

> Meaningful technical ownership  
> Broader engineering influence  
> Solve reliability problems properly rather than work around them

### 中文

- 承担有实质意义的技术责任
- 在更大范围内影响工程团队
- 从根本上解决可靠性问题，而不是使用临时方案绕过去

### 解读

这三句话已经说明，这不是一个以“值班、处理告警、维护服务器”为主的传统运维岗位。

Pushpay 想要的是一名能够：

- 独立承担复杂问题；
- 跨应用、基础设施和数据层排障；
- 修改系统设计；
- 用代码和自动化消除重复故障；
- 影响其他工程团队决策；

的高级可靠性工程师。

它更接近：

> **Senior Software Engineer + Senior Platform Engineer + Senior SRE**

---

## 二、Pushpay 对 SRE 的理解

### 原文

> The best Site Reliability Engineers do more than keep systems running.

### 中文

优秀的站点可靠性工程师不仅仅是保证系统持续运行。

---

### 原文

> They improve how software is designed. They remove recurring sources of operational pain. They build tools that make engineering teams faster and safer.

### 中文

他们会改进软件的设计方式，消除反复出现的运维痛点，并开发工具，让工程团队能够更快、更安全地交付软件。

### 解读

Pushpay 明确把 SRE 视为一个**软件工程岗位**。

它希望你不仅能够响应事故，还能够：

- 修改应用设计；
- 改进发布方式；
- 优化监控和告警；
- 编写内部工具；
- 建立自动化流程；
- 提升开发团队的交付效率。

---

### 原文

> They see risks before they become incidents and leave systems easier to understand, operate and change than they found them.

### 中文

他们会在风险演变成生产事故之前发现问题，并让系统变得比接手时更容易理解、运行和修改。

### 解读

这句话要求候选人具备两种能力：

1. **前置风险识别能力**
   - 架构评审
   - 容量分析
   - 安全风险分析
   - 发布风险分析
   - 依赖和故障模式分析

2. **持续改善能力**
   - 不只是修复一次故障；
   - 而是改善系统结构、文档、工具和流程；
   - 让后续维护成本不断降低。

---

## 三、岗位工作范围

### 原文

> This is an opportunity to work across software, cloud infrastructure, reliability, security and engineering practices.

### 中文

这个岗位会横跨软件开发、云基础设施、可靠性、安全以及工程实践等多个领域。

### 解读

该岗位不是单一技术栈岗位。候选人需要能在以下边界之间切换：

- 应用代码
- 云基础设施
- CI/CD
- 数据库
- 网络
- 安全
- 可观测性
- 事故管理
- 架构设计
- 团队工程规范

---

### 原文

> You will take on complex problems that do not always arrive neatly defined, follow them across system boundaries and turn them into durable improvements.

### 中文

你将处理那些定义并不总是清晰的复杂问题，跨越不同系统边界追踪问题，并将其转化为长期、可靠的改进。

### 解读

这是一条非常明显的 Senior 信号。

Pushpay 不会总是给你一个已经拆解好的 Jira Ticket。你需要自己完成：

1. 理解模糊问题；
2. 收集证据；
3. 确定影响范围；
4. 找到真正的根因；
5. 提出解决方案；
6. 拆分实施步骤；
7. 推动相关团队执行；
8. 验证长期效果。

---

## 四、典型工作场景

### 1. 跨层生产事故调查

#### 原文

> Investigating a difficult production issue that crosses application, infrastructure and data boundaries.

#### 中文

调查一个跨越应用、基础设施和数据层边界的复杂生产问题。

#### 可能包含

- 应用超时；
- 数据库连接池耗尽；
- 云资源瓶颈；
- DNS、TLS 或网络故障；
- 消息堆积；
- Kubernetes 调度问题；
- 缓存击穿；
- 第三方服务异常；
- CI/CD 发布导致的回归问题。

---

### 2. 消除重复告警

#### 原文

> Identifying a pattern behind repeated alerts and engineering the underlying problem away.

#### 中文

识别重复告警背后的规律，并通过工程手段彻底消除底层问题。

#### 解读

Pushpay 不满足于：

- 重启服务；
- 调高阈值；
- 静音告警；
- 增加人工 Runbook。

它更希望你：

- 找到重复告警的共同原因；
- 修改代码或架构；
- 自动恢复；
- 增加限流、重试或熔断；
- 优化容量；
- 改进监控信号；
- 删除无价值告警。

---

### 3. 构建自动化和内部工具

#### 原文

> Building automation that removes manual work.

#### 中文

开发能够减少人工操作的自动化工具。

#### 可能涉及

- Terraform 模块；
- 发布自动化；
- 环境初始化；
- 数据修复工具；
- 事故诊断脚本；
- 自动扩缩容；
- 自动回滚；
- 漏洞扫描；
- 合规检查；
- 平台自助服务。

---

### 4. 提升可观测性

#### 原文

> Improving the operational visibility of a service.

#### 中文

提升服务在生产环境中的可见性和可观测性。

#### 可能涉及

- Metrics；
- Logs；
- Traces；
- Dashboards；
- Alerting；
- SLI/SLO；
- Error Budget；
- Dependency Map；
- Deployment Markers；
- Incident Timeline。

---

### 5. 在上线前挑战设计

#### 原文

> Challenging a design before it creates reliability problems in production.

#### 中文

在某项设计引发生产可靠性问题之前，对其提出质疑和改进意见。

#### 解读

你需要能够在架构评审时提出类似问题：

- 这个服务失败后会发生什么？
- 是否存在单点故障？
- 数据一致性如何保证？
- 重试是否可能形成重试风暴？
- 数据库连接数是否可控？
- 是否具备回滚能力？
- 依赖不可用时如何降级？
- 如何观测和诊断？
- RTO 和 RPO 是什么？
- 安全和合规风险是什么？

---

## 五、核心职责中英文对照

| 英文职责 | 中文解释 | 实际含义 |
|---|---|---|
| Take end-to-end ownership of complex reliability and systems work | 对复杂可靠性和系统工作承担端到端责任 | 不只是完成分配任务，而是从发现问题到验证结果全程负责 |
| Recognise operational, architectural and security risks early | 尽早识别运维、架构和安全风险 | 需要具备设计评审和风险分析能力 |
| Turn broad or ambiguous problems into clear, well-scoped improvements | 将宽泛、模糊的问题转化为清晰、范围合理的改进事项 | 自己澄清问题、拆分工作并确定优先级 |
| Build tools and automation that reduce toil | 构建工具和自动化，减少重复性人工劳动 | 通过代码、脚本、平台和 IaC 降低运维成本 |
| Improve observability, operability and resilience | 改善可观测性、可运维性和系统韧性 | 不仅“可用”，还要容易发现问题、恢复和维护 |
| Help teams make stronger technical decisions before software reaches production | 帮助团队在软件进入生产前作出更好的技术决策 | 参与架构、发布和可靠性评审 |
| Communicate trade-offs, risks and progress clearly | 清楚沟通权衡、风险和进度 | 不只汇报技术细节，还要说明业务影响 |
| Mentor engineers | 指导其他工程师 | 需要有知识分享和培养他人的能力 |
| Contribute to technical hiring | 参与技术招聘 | 可能参加技术面试和候选人评估 |
| Participate in on-call support | 按排班参与值班支持 | 需要承担真实生产环境责任 |

---

## 六、On-call 的真正要求

### 原文

> The objective is not simply to respond well when something goes wrong. It is to learn from operational signals, remove recurring failure modes and steadily improve the systems and practices behind them.

### 中文

值班的目标不只是事故发生时能够正确响应，而是从生产信号中学习，消除反复出现的故障模式，并持续改善背后的系统和工程实践。

### 解读

Pushpay 期待的闭环是：

```text
告警或事故
    ↓
快速恢复服务
    ↓
收集证据
    ↓
根因分析
    ↓
Blameless Postmortem
    ↓
确定 Corrective Actions
    ↓
修改代码、架构、监控或流程
    ↓
验证问题不再重复
```

只会处理事故不够，必须能推动长期整改。

---

## 七、跨团队影响力

### 原文

> You will work with Software Engineers, QA, Product, Delivery, Engineering Managers and other parts of the business.

### 中文

你将与软件工程师、QA、产品、交付、工程经理以及其他业务团队合作。

### 解读

该岗位需要较强的沟通能力，因为很多可靠性问题不能由 SRE 团队单独解决。

你可能需要：

- 说服开发团队修改应用；
- 与产品团队讨论可靠性优先级；
- 与 QA 改进故障和回归测试；
- 与管理层说明技术风险；
- 推动多个团队接受统一标准；
- 在可靠性、速度和成本之间作出权衡。

---

## 八、经验要求

### 原文

> Strong experience across site reliability engineering, systems engineering or software engineering.

### 中文

需要在站点可靠性工程、系统工程或软件工程领域具备扎实经验。

### 解读

新版岗位没有硬性要求候选人必须拥有多年的正式 SRE 职位名称。

它接受三类背景：

1. SRE；
2. Systems Engineer；
3. Software Engineer。

这对有后端、分布式系统、云和基础设施经验的候选人比较友好。

---

### 原文

> Ideally gained while developing or operating multi-user web, mobile or cloud software products.

### 中文

相关经验最好来自多用户 Web、移动端或云软件产品的开发或运行维护。

### 解读

Pushpay 更偏好 SaaS 或互联网产品经验。

候选人需要证明自己理解：

- 多用户系统；
- 高并发；
- 数据隔离；
- API；
- 数据库；
- 可用性；
- 安全；
- 发布风险；
- 客户影响。

---

## 九、技术栈要求

### 原文与中文对照

| 技术领域 | 英文 | 中文 |
|---|---|---|
| 云平台 | AWS, GCP or Azure | AWS、GCP 或 Azure |
| 基础设施即代码 | Terraform or CloudFormation | Terraform 或 CloudFormation |
| 编程语言 | Python, Go, Rust, JavaScript or .NET | Python、Go、Rust、JavaScript 或 .NET |
| 命令行环境 | Bash and PowerShell | Bash 和 PowerShell |
| 数据库 | Relational or document-based databases | 关系型或文档型数据库 |
| Web 技术 | HTTP, SSL/TLS, REST APIs or GraphQL | HTTP、SSL/TLS、REST API 或 GraphQL |
| 版本控制 | Git and distributed version control | Git 和分布式版本控制 |
| CI/CD | GitHub Actions, Jenkins or similar tools | GitHub Actions、Jenkins 或其他 CI/CD 工具 |

### 重要解读

原文使用的是：

> Your experience may include several of the following

意思是：

> 你的经验可以包含以下若干项。

这不是要求全部掌握。

真正的核心筛选条件是：

- 能写代码；
- 理解云基础设施；
- 能做自动化；
- 理解端到端系统；
- 有生产可靠性意识；
- 能独立承担复杂问题。

某个具体语言不是关键。即使主要使用 Java，只要能展示成熟的软件工程和系统能力，仍然可以匹配。

---

## 十、沟通与业务理解要求

### 原文

> Reason across an end-to-end system.

### 中文

能够对端到端系统进行整体分析和推理。

### 解读

不能只理解其中一个组件，需要理解完整链路：

```text
用户请求
→ DNS/CDN/WAF
→ Load Balancer
→ API
→ Application
→ Cache
→ Database
→ Queue
→ External Dependency
→ Monitoring and Alerting
```

---

### 原文

> Explain technical work in terms of its wider business impact.

### 中文

能够从更广泛的业务影响角度解释技术工作。

### 解读

不能只说：

> 我优化了数据库连接池。

更好的表达是：

> 通过限制连接并改进超时策略，降低了高峰期服务耗尽风险，减少客户支付失败，并降低了值班团队的事故处理压力。

---

## 十一、公司与工程文化

### 原文

> Blameless postmortems

### 中文

无责事故复盘。

### 解读

复盘重点不是追究某个人，而是分析：

- 系统为什么允许错误发生；
- 监控为什么没有提前发现；
- 流程为什么没有阻止故障；
- 如何通过工程改进避免再次发生。

---

### 原文

> Brainfood learning sessions

### 中文

内部技术学习和知识分享活动。

### 解读

说明公司鼓励内部分享和持续学习。对于希望向 Platform、Staff 或架构方向发展的工程师，这是积极信号。

---

## 十二、福利

| 福利 | 中文说明 |
|---|---|
| $3,000 annual training/conference allowance | 每年 3,000 新西兰元培训或技术会议预算 |
| Hybrid work | 每周最多两天居家办公 |
| KiwiSaver | 新西兰退休储蓄计划 |
| Paid parental leave | 带薪育儿假 |
| 10 days sick leave from start | 入职后即可享有每年 10 天病假 |
| Advanced gear | 提供较好的办公和开发设备 |
| Healthy food and drink | 办公室提供健康食品和饮料 |
| Flu shots | 免费年度流感疫苗 |
| Social events | 公司社交活动 |
| Volunteering options | 志愿服务机会 |

---

## 十三、工作权要求

### 原文

> To be considered for this vacancy you must have working rights for New Zealand.

### 中文

要获得该岗位的考虑，候选人必须拥有新西兰合法工作权。

### 解读

这是硬性条件。

通常意味着候选人在入职时必须能够合法从事全职工作。学生签证的有限工作权是否满足要求，需要根据：

- 入职时间；
- 每周允许工作的小时数；
- 是否已经取得毕业后工作签证；
- 招聘方是否愿意等待签证转换；

具体判断。

---

# 十四、岗位真实画像

这个岗位不是：

- 传统系统管理员；
- 纯云资源运维；
- 只负责 Terraform 的 DevOps；
- 只处理告警的生产支持；
- 只负责 Kubernetes 集群维护。

它真正需要的是：

> **能够使用软件工程方法解决生产可靠性问题，并在组织范围内影响系统设计和工程实践的高级工程师。**

---

# 十五、显性要求与隐藏要求

## 显性要求

- Cloud；
- Terraform 或 CloudFormation；
- 编程能力；
- Bash 或 PowerShell；
- 数据库；
- HTTP、TLS 和 API；
- Git；
- CI/CD；
- On-call；
- Mentoring；
- Technical hiring；
- Architecture review。

## 隐藏要求

- 有真实生产事故经验；
- 能进行系统性根因分析；
- 能在信息不完整时作出判断；
- 能推动其他团队完成整改；
- 能用英文清楚说明复杂技术问题；
- 能平衡可靠性、交付速度、成本和业务价值；
- 能承担 Senior 级别的自主性和责任；
- 不依赖经理持续拆解任务。

---

# 十六、可能的面试考察方向

## 1. 生产事故题

例如：

- 描述一次严重生产事故；
- 如何判断影响范围；
- 如何恢复服务；
- 如何找到根因；
- 如何避免再次发生；
- 哪些地方可以自动化。

## 2. 系统设计题

例如：

- 设计一个高可用 API；
- 如何处理数据库故障；
- 如何设计重试、超时和熔断；
- 如何避免单点故障；
- 如何设计可观测性；
- 如何支持安全回滚。

## 3. 可观测性题

例如：

- 如何定义 SLI 和 SLO；
- 如何减少告警噪声；
- Metrics、Logs 和 Traces 如何配合；
- 如何判断告警是否具备行动价值。

## 4. 自动化题

例如：

- 如何消除重复人工操作；
- 如何设计安全的自动修复；
- 如何使用 Terraform 管理基础设施；
- 如何防止自动化扩大事故影响。

## 5. Senior 行为题

例如：

- 如何处理模糊需求；
- 如何与不同团队发生技术分歧；
- 如何指导 Junior Engineer；
- 如何推动团队采用新的工程标准；
- 如何解释技术工作的业务价值。

---

# 十七、结合你的背景进行匹配

## 已经比较强的部分

- 四年大型企业软件工程经验；
- Java 和后端系统经验；
- Kubernetes；
- AWS Solutions Architect Professional；
- Azure Solutions Architect Expert；
- CKAD；
- Terraform；
- 分布式系统；
- 网络、日志、链路追踪和故障排查；
- CI/CD；
- 跨团队 Agile 协作。

## 仍需强化的证据

- 正式的 SLI/SLO 和 Error Budget；
- 完整的 Incident → RCA → Corrective Action 闭环；
- Blameless Postmortem；
- 多层故障注入和恢复实验；
- 可观测性设计；
- 数据库和应用可靠性；
- 架构评审文档；
- Mentoring 和工程影响力；
- 可靠性改进的量化结果；
- 新西兰合法全职工作权。

---

# 十八、毕业前的目标状态

将该岗位作为北极星时，毕业前应能够展示至少两个完整案例。

## 案例一：复杂生产故障闭环

应包含：

- 系统架构；
- SLI/SLO；
- 故障注入；
- 告警和检测；
- 影响评估；
- 临时恢复；
- 根因分析；
- Postmortem；
- 长期整改；
- 再次验证。

## 案例二：平台或工程效率改进

应包含：

- 原有人工流程；
- Toil 测量；
- 自动化设计；
- Terraform 或平台工具；
- 安全控制；
- CI/CD；
- 使用文档；
- 效率或可靠性改进结果。

---

# 十九、最终评价

## 技术难度

**高。**

它要求候选人同时理解软件、云、基础设施、数据、可靠性和安全。

## Senior 程度

**明确属于 Senior。**

Mentoring、technical hiring、architecture review 和 end-to-end ownership 都是高级工程师信号。

## 对普通 SRE 岗位的覆盖程度

如果能够达到该岗位约 70%–80% 的实质能力，那么通常已经可以覆盖：

- Site Reliability Engineer；
- Platform Engineer；
- Cloud Engineer；
- DevOps Engineer；
- Production Engineer；
- 部分 Senior Software Engineer（Cloud/Platform）岗位。

## 一句话总结

> Pushpay 想招聘的不是“会维护云环境的人”，而是“能够通过代码、架构和组织影响力，持续降低系统故障和工程成本的高级软件型 SRE”。
