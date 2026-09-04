**Huawei + OKX 工作经验康复专项**

14 天 × 每晚 2 小时｜基于最新量化版简历恢复工程细节、白板能力与面试表达

目标：不是简单复习 Java / Kubernetes，而是把 2021–2025 的 Huawei 与 OKX
工作经历重新恢复到“能讲、能画、能扛追问”的状态，并确保最新简历中的每个数字、规模和
impact 都能在面试中解释清楚。

# 1. 为什么要做这个专项

- 当前最大的风险不是“技术不会”，而是工作细节逐渐模糊。最新简历已经明确写出
  2 个核心模块、5-service 平台、5–6 套 staging/test 环境、客户
  production、hundreds of Thrift contracts、约 20 个 high-severity
  defects/year 等具体信息，面试官很可能围绕这些数字连续追问。

- 因此康复重点不是继续堆新技术，而是恢复“当时真实做过什么、系统如何工作、这些数字从哪里来、出了什么问题、你如何判断和解决”的工程记忆。

- 核心原则：不要把 2026 年后来学会的 SRE / Kubernetes / Cloud 能力倒灌成
  2021–2025 年的工作经历。所有数字与 impact
  必须能解释来源；所有面试表达都要区分 Confirmed / Approximate but
  Defensible / Uncertain / Learned Later。

# 2. 当前已确认的量化锚点

下面这些不是“为了好看硬编的 KPI”，而是当前简历中需要在面试里守得住的真实
scope / cadence / impact 锚点。

| 量化锚点              | 当前表述 / 面试恢复重点                                                                                                |
|-----------------------|------------------------------------------------------------------------------------------------------------------------|
| Ownership             | 负责 2 个核心功能模块，所在 cluster network management platform 共 5 个 service。                                      |
| Environments          | 日常涉及约 5–6 套 staging / test 小环境，并参与客户 production deployment 的问题定位与交付。                           |
| Release cadence       | 约每季度一次 showcase；major product release 约 18 个月一次。                                                          |
| Distributed contracts | 系统核心开发流程涉及 hundreds of Thrift IDL-based service contracts；需要区分“平台总规模”与“本人实际设计/演进的接口”。 |
| Reliability workload  | 每年约处理 20 个 high-severity production / pre-production defects 或严重问题单。                                      |
| Cross-team scope      | 典型工作会涉及 peer service / downstream release / platform / CLI / Linux SDK / architecture 等团队边界。              |

# 3. 每晚固定 2 小时模板

| 时间        | 动作              | 目标                                                              |
|-------------|-------------------|-------------------------------------------------------------------|
| 0–25 min    | 记忆恢复          | 不看答案，凭记忆写系统、上下游、职责、故障、解决过程。            |
| 25–80 min   | 技术重建          | 查旧资料、画架构、做最小 demo，把模糊机制补回来。                 |
| 80–110 min  | 面试表达          | 同一主题练 30 秒 / 2 分钟 / 5–10 分钟三个版本。                   |
| 110–120 min | Experience Ledger | 记录 Confirmed / Uncertain / Need Review / Good Interview Story。 |

# 4. 14 天康复计划

| Day | 主题                                     | 2 小时重点                                                                                                                                                                 | 当晚必须产出                                                        |
|-----|------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| 1   | Huawei 全局架构 + Ownership              | 重建 five-service cluster network management platform；明确你负责的 2 个核心模块、服务位置、上下游、数据流、控制面/数据面。                                                | 1 张系统架构图 + 1 张 ownership / team-boundary 图 + 3 分钟整体介绍 |
| 2   | Java / Spring Boot 两个核心模块          | 针对 2 个核心模块恢复 Controller/Service/DAO、配置、异常处理、并发、缓存、接口调用链；挑出你真正 ownership 最强的功能。                                                    | 每个模块各画 1 条 request → service → downstream 调用链             |
| 3   | Kubernetes + 5–6 套环境                  | 恢复 5–6 套 staging/test 环境与 customer production deployment 的差异；Pod/Deployment/Service/ConfigMap、配置注入、release validation、故障排查。                          | 画 environment + deployment lifecycle，并列出你亲自做过的动作       |
| 4   | Distributed Systems / Service Boundaries | 梳理 five-service 平台、peer service、downstream release、platform / architecture 等边界；同步/异步调用、失败传播、状态归属与一致性。                                      | 回答“一个功能为什么会跨多个 service / team？”                       |
| 5   | Thrift IDL / Hundreds of Contracts       | 恢复 Why Thrift、IDL、serialization、generated client/server、interface evolution、timeout/error handling；解释 hundreds of contracts 是平台规模还是本人直接修改范围。     | 手写最小 Thrift interface + 准备“hundreds”数字来源解释              |
| 6   | Operational State / Consistency          | 恢复 runtime operational state 的来源、同步路径、in-memory data store 角色、stale state / race / failure；解释“maintain consistency across service boundaries”具体指什么。 | 重建 1 条真实 operational state flow + 明确 consistency semantics   |
| 7   | ~20 High-Severity Issues / Year          | 回忆每年约 20 个严重单的来源与口径；日志、trace、packet、debug、static analysis 分别在哪类问题中使用。                                                                     | 列出至少 5 个问题标题，其中选 2 个作为核心 incident story           |
| 8   | Incident 深挖 + Impact                   | 对 2 个真实案例执行 Symptom → Hypothesis → Evidence → RCA → Fix → Validation → Customer Impact；明确哪些发生在 production、哪些在 pre-production。                         | 2 个完整 STAR / incident narrative + 一句 impact 总结               |
| 9   | Release / Showcase / Production          | 恢复 quarterly showcase、major release ≈ 18 months 的真实流程；test→staging→customer production、release validation、版本交付、rollback/恢复边界。                         | 回答“你实际参与 release 和 production 到什么程度？”                 |
| 10  | Cross-team Engineering                   | 恢复 downstream release、platform、CLI、Linux SDK、architecture 等团队的职责边界；典型 issue 谁分析、谁出版本、谁对客户交付。                                              | 画 1 张 team responsibility map + 准备 1 个跨团队问题案例           |
| 11  | OKX 架构恢复                             | matching system 周边、event stream、Aeron Archive、record/replay 的位置，以及你本人负责的 Java backend scope。                                                             | 画 OKX replay architecture                                          |
| 12  | Replay / Recovery                        | 为什么 replay 能恢复 state；event ordering；checkpoint/snapshot；故障恢复边界；哪些机制是本人参与，哪些属于团队/平台。                                                     | 5 分钟讲清 replay-based recovery                                    |
| 13  | Resume Bullet Attack                     | 逐条攻击最新 Huawei/OKX bullet，尤其追问 2 modules、5 services、5–6 envs、hundreds of Thrift contracts、~20 severe issues/year 的来源和 scope。                            | 每个 bullet 准备 5 个 follow-up answers                             |
| 14  | Full Mock                                | 45min Huawei + 20min OKX + distributed systems / production / SRE follow-up；重点检查数字是否可守、impact 是否过度。                                                       | 完整录音 1 次并根据回答结果修正简历措辞                             |

# 5. Huawei 五条简历 Bullet 的恢复标准

Bullet 1 — Backend / Ownership & Scale：必须能解释“2
个核心功能模块”和“5-service platform”分别是什么；你 personally owned
哪些功能；configuration、monitoring、traffic shaping、congestion
management、host security、device-port lifecycle
中哪些是你直接做过、哪些是平台整体范围。

Bullet 2 — Kubernetes / Environments & Releases：必须能解释 5–6 套
staging/test 环境是什么、customer production deployment
与内部环境有什么不同；你实际做过哪些 deployment / config / validation /
troubleshooting；quarterly showcase 和 major release（约 18
个月）的流程与个人参与边界。

Bullet 3 — Thrift / Distributed Contracts：必须能解释 hundreds of Thrift
IDL-based contracts
的数字来源，并明确这代表平台/开发流程的规模，不等于几百个接口都由你本人设计。至少恢复
Why Thrift、IDL、serialization、generated client/server、interface
evolution、timeout/error handling，以及 runtime operational state
如何跨服务同步。

Bullet 4 — Production / Reliability & Impact：必须守住“约 20 个
high-severity defects/year”的口径：严重单如何定义、production 与
pre-production 如何区分、哪些是本人直接定位解决。至少准备 2
个真实案例：Symptom → Hypothesis → Logs/Trace/Packet Evidence → Root
Cause → Fix → Validation → Customer Impact。

Bullet 5 — Cross-team Engineering：必须能讲清 downstream
release、platform、CLI、Linux SDK、architecture
等团队的职责边界，以及典型 feature / issue 为什么需要跨团队。准备 1
个真实案例，说明你负责分析/接口/代码/验证中的哪一段，下游团队如何出版本或完成后续交付。

# 6. Day 1 白板架构恢复模板

User / CLI / API  
\|  
v  
Cluster Network Management Platform (5 services)  
\|  
+------ Core Module A (your ownership)  
+------ Core Module B (your ownership)  
+------ Peer Service(s)  
\|  
v  
Cross-service RPC / Thrift IDL Contracts  
\|  
+------ Downstream Release / Integration Service  
+------ Platform / CLI / Linux SDK boundaries  
\|  
v  
Runtime Operational State / In-memory Data Store  
\|  
v  
Customer Cluster / Network / Device Environment

注意：上图只是恢复模板。Day 1 的任务是把“5 services、2 core
modules、下游 release team、Thrift、runtime state、customer
production”替换成你真实系统中的名称和调用关系；不能把示意结构直接当事实。

# 7. OKX：只打穿一个 Replay Story

OKX 时间较短，不需要准备很多故事。目标是把 Aeron Archive + matching
system + replay-based recovery 这一条讲透。

Incoming Events  
\|  
v  
Matching System  
\|  
+----\> Live Processing  
\|  
+----\> Aeron Archive  
\|  
record  
\|  
failure / recovery  
\|  
replay  
\|  
v  
reconstruct state

重点准备以下追问：

- 为什么不能简单从数据库恢复？

- Replay 从哪里开始？如何判断恢复完成？

- Event ordering 为什么重要？

- Duplicate replay 会怎样？

- Replay 时 downstream 如何处理？

- 哪些机制是你亲自参与的，哪些不在你的 scope？

# 8. 最终验收标准

**Level 1 — 30 秒**：一句话讲清“我做了什么”。

**Level 2 — 2 分钟**：系统背景 + 我的职责 + 技术方案 + 结果。

**Level 3 — 5 分钟**：白板 architecture + data flow + failure modes。

**Level 4 — 追问**：能回答 Why? Why not X? What failed? How did you
debug it? What was your exact contribution? What would you change today?

# 9. 优先级结论

**这 28 小时的 ROI 高于继续学习一批新工具。**

你当前已经拥有足够多的新技术和云平台知识。真正能显著提升面试转化率的，是把
Huawei 四年的“量化版简历”恢复成随时可调用的工程经验库：2 modules / 5
services / 5–6 environments / hundreds of Thrift contracts / ~20 severe
issues per year，每一个数字都能解释，每一个 impact
都有真实案例支撑；同时让 OKX replay story 成为一个可被深挖的高质量案例。

# 10. Experience Ledger 模板

| 主题                 | Confirmed                                                                                                                   | Uncertain | Need Review | Good Interview Story |
|----------------------|-----------------------------------------------------------------------------------------------------------------------------|-----------|-------------|----------------------|
| Huawei Quant Anchors | 2 modules; 5 services; 5–6 envs; quarterly showcase; ~18mo major release; hundreds Thrift contracts; ~20 severe issues/year |           |             |                      |
