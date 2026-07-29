# Terraform × AWS 社区高论（来源：公司技术博客）

整理日期：2026-07-27  
主要来源：互联网公司、SaaS、金融科技与基础设施公司的工程博客；少量公司工程师在 HashiCorp/AWS 官方站点发布的案例演讲。  
筛选目标：寻找 Terraform × AWS 在生产规模、组织协作、故障恢复、权限边界、版本迁移、平台抽象和成本治理上的真实经验；排除入门教程、SEO 拼贴、只展示 `terraform apply` 成功截图以及没有失败条件的产品宣传。

> 公司博客不是中立论文。它会省略组织政治、失败总量、成本和仍未解决的问题，也经常承担招聘或产品营销任务。本文把同一模式在多家公司的重复出现视为强信号，把单家公司成功故事视为有条件个案。  
> 本文中的 **[公司自述]** 是工程团队公开复盘；**[厂商自述]** 是工具或服务提供商讲自己的 Terraform 集成；**[联合案例]** 是 AWS/HashiCorp 与客户共同发布，事实可用，但产品取向需折价；**[反例]** 是公司选择了 Terraform 之外的控制面。  
> “搜完所有公司博客”在数学意义上不可能。本文做的是高信号策展：覆盖十余家不同规模和行业的生产团队，并用当前 Terraform 与 LocalStack 官方资料校正旧文章中的版本事实。

---

## 0. 先吸收这 40 条

1. **大规模 Terraform 的核心控制面不是 HCL，而是 `代码路径 → state → AWS 账号/角色 → 所有者` 的可信映射。**
2. **backend 路径不是普通字符串，它是资源身份、写权限和故障域的一部分。**
3. **Grab 真实遇到过复制 Terraform 项目却忘改远端 state，结果原地替换既有资源；Pinterest 后来把 backend bucket/key/KMS 与 workspace 映射做成强制校验。**
4. **只靠 code review 不能证明一个目录有权修改某个 state；机器应在 plan 前验证路径、state 和执行角色。**
5. **Slack 的约 1,400 个 state 说明拆分不是洁癖，而是账号、区域、服务所有权、plan 延迟和 blast radius 共同作用的结果。**
6. **state 应按所有权、生命周期、权限边界和故障域切，不应只按 `dev/staging/prod` 或目录美观切。**
7. **判断是否拆 state 的好问题是：如果这份 state 的所有资源被误删，受影响的是不是同一团队、同一系统、同一恢复优先级？**
8. **区域资源与 IAM/CloudFront 等全局资源通常应分开；大型服务还可能需要独立 state。**
9. **拆 state 会把一个大图变成分布式系统：依赖发布、执行顺序、兼容性、失败重试和所有权转移必须被显式设计。**
10. **不要让下游随意读取上游整份 remote state；优先发布窄而稳定的接口，如 SSM 参数、DNS、标签查询或受控输出。**
11. **生产 apply 应由中央执行器完成，作者本机最多拥有 plan/read 权限。**
12. **短期 OIDC 身份、中央入口角色和降权后的 workspace 角色，比静态 AWS key 更适合 CI。**
13. **“PR 已批准”与“明确授权这次 apply”是两件事；Pinterest 使用代码审批加显式 apply 触发形成双重控制。**
14. **plan 必须贴回 PR，但仅展示 plan 不等于 apply 了同一份计划；还要绑定 commit、依赖锁、变量、身份和 plan artifact。**
15. **Slack 的 Smart Planner 会找出 module 变更间接影响的 state；平台必须理解消费者图，而不只检查本目录。**
16. **DoorDash 用 Conftest/OPA 自动批准低风险 Security Group 规则，把安全团队从机械审查移到政策设计和例外处理。**
17. **策略门禁的目标不是“让所有 PR 都找安全团队”，而是让符合已编码政策的改动自动通过。**
18. **Terraform 通过 API 执行时可能看不到控制台警告；Slack 的 DNSSEC 事故说明危险操作需要服务专属 precondition、runbook 和外部探针。**
19. **`terraform apply` 成功只证明 provider/API 接受了配置，不证明 DNS、路由、复制、权限或业务 SLO 已经健康。**
20. **平台模块应该交付业务能力，而不是机械包一层 AWS resource。**
21. **Deliveroo 的“应用”模块一次建立 ECR、CI、IAM、CloudWatch、ALB、ECS、DNS 和监控，展示了 capability module 的正确抽象层。**
22. **Figma 用模块把 ALB、Cognito、Okta SAML 和授权规则组合成安全内部应用入口，说明 Terraform 的优势常在跨 provider 事务图。**
23. **Raw Terraform 自助不等于平台自助；Grab 最终在 Terraform 上方增加 UI、API、元数据和受控 module。**
24. **好的平台保留 Terraform 作为可恢复的执行后端，但不要求每个使用者掌握 provider、backend 和 state 细节。**
25. **平台应保留高级入口或 escape hatch；否则批量变更和非标准场景会绕过平台。**
26. **Terraform module 是组织 API；需要 owner、版本、升级说明、消费者清单、兼容窗口和弃用机制。**
27. **Slack 的 0.11→0.12 升级耗费一个季度，说明 CLI/provider 升级必须当作平台产品和迁移项目。**
28. **Datadog provider v4 把四个 AWS integration resource 合成一个，说明 provider schema 会改变“谁拥有同一底层对象”；升级前必须解决 state address 和双重所有权。**
29. **Indeed 的 AWS 紧急迁移同时让数百名不熟 AWS、也不熟 Terraform 的工程师上手，教训是工具迁移与云技能迁移不能被当成同一件小事。**
30. **Guild/社区、模板、支持频道和可发现的标准，与 CI 技术同样属于 Terraform 平台。**
31. **过度中心化会让平台团队成为人工 reviewer 和单点故障；应中心化规则、执行和可观测性，分散资源所有权。**
32. **Terraform 代码和 state 可以成为治理数据源，但不能替代真实 AWS inventory；Airbnb 同时解析 Terraform 与 S3 Inventory。**
33. **所有权、成本中心、数据分类和服务身份应进入机器可读元数据，不能只存在于 README 或团队记忆。**
34. **漂移不会因为宣称 GitOps 就消失；Cloudflare 用持续 drift detector 自动开工单，并为修复设置 SLA。**
35. **高频应用发布不该拖着低频基础设施图一起跑；Figma 将服务定义和部署从 Terraform 生命周期移到 Kubernetes/内部部署系统。**
36. **当需求是全量运行时枚举、break-glass 即时修改和自动批量改写配置时，Terraform/HCL 可能不是合适的主数据模型；Spotify 明确选择 Kubernetes CR。**
37. **AWS-only 团队也不必天然选择 Terraform；Nubank 用 CloudFormation 加自研 Nimbus 做 3,000+ stack 的不可变蓝绿平台。**
38. **LocalStack 很适合验证 provider 调用、模块组合、state 隔离、策略和失败注入；它不能证明真实 IAM、配额、延迟、DNS 缓存和服务控制面完全一致。**
39. **Pinterest 已公开说部分团队把 LocalStack 放进 RPP 流水线测试复杂架构，这证明本地云模拟可以是企业控制面的一层，而不只是个人开发工具。**
40. **成熟路线不是“Terraform 管一切”，而是为每类资源选择最合适的 owner，并让边界、身份、审计和恢复路径都可见。**

---

## 1. 我怎么筛公司技术博客

### 1.1 高信号判据

一篇文章至少满足两项才进入正文：

- 披露资源量、账号数、workspace/state 数、团队数或迁移时长；
- 描述真实事故、错误操作、性能瓶颈或恢复过程；
- 给出组织边界：谁写、谁审、谁 apply、谁值班、谁升级；
- 展示 state、权限、CI、模块或控制面的具体架构；
- 解释为什么旧方案失败，而不只是展示新方案；
- 给出 Terraform 不适用的反例；
- 能从另一家公司找到独立印证；
- 旧文章虽版本过时，但失败模型仍然长期有效。

### 1.2 降权内容

- 工具厂商未披露客户条件的“最佳实践”；
- 没有规模和失败路径的 hello-world；
- 把一个成功 apply 当成生产验证；
- 用“多云”暗示同一份资源代码可跨云；
- 只谈 DRY，不谈模块兼容与消费者升级；
- 只说“GitOps”，不说 break-glass 和 drift 回收；
- 只说“least privilege”，不展示身份链和授权边界；
- 把旧版 DynamoDB 锁、旧 HCL 语法或 provider 行为写成当前事实。

### 1.3 证据标签

- **[跨公司共识]**：三家以上独立工程团队出现同一模式。
- **[强个案]**：有规模、事故或清晰因果链，但环境特定。
- **[反例]**：公司明确不用 Terraform 解决该类问题。
- **[历史]**：文章的技术版本已老，但经验仍有价值。
- **[当前校正]**：用 2026-07-27 可访问的官方资料更新做法。

---

## 2. 高信号公司地图

| 公司 / 团队 | 公开规模或情境 | 最值得吸收的东西 | 证据偏差 |
|---|---|---|---|
| [Slack：How We Use Terraform](https://slack.engineering/how-we-use-terraform-at-slack/) | 数万 EC2、约 1,400 state、多 AWS 子账号 | state 演化、版本升级、module catalog、影响面 plan | 公司自述 |
| [Slack：DNSSEC 事故](https://slack.engineering/what-happened-during-slacks-dnssec-rollout/) | Route53 DNSSEC 回滚、递归解析器缓存 | API 路径漏掉控制台警告；外部探针的重要性 | 公司事故复盘 |
| [Pinterest：RPP](https://medium.com/pinterest-engineering/securing-infrastructure-at-scale-introducing-pinterests-resource-provisioner-pipeline-rpp-8283bb12cbe5) | 数百 workspace、数万资源、多 repo | OIDC、角色链、backend 强校验、双重控制、LocalStack | 公司自述 |
| [Pinterest：多账号转型](https://aws.amazon.com/blogs/mt/from-monolith-to-multi-account-pinterests-aws-organization-transformation-journey/) | 既有 AWS estate 向多账号迁移 | 现成 Landing Zone 产品不一定兼容遗留结构 | AWS 联合案例 |
| [Grab：An elegant platform](https://engineering.grab.com/an-elegant-platform) | 数据流平台、多团队自助 | 错 state 事故、UI/API/Git/Terraform 三层控制面 | 公司自述 |
| [Grab：Airflow on EKS](https://engineering.grab.com/the-journey-of-deploying-apache-airflow-at-Grab) | KOPS→EKS；每 namespace 配 RDS/Redis | Terraform、Helm、Vault 与应用平台的责任分层 | 公司自述 |
| [Deliveroo：Application Deployment](https://deliveroo.engineering/2018/02/21/application-deployment.html) | 单体拆到 50+ 服务，每周新增服务 | capability module 和高频发布边界 | 公司自述、历史 |
| [Indeed：AWS 迁移](https://www.hashicorp.com/en/resources/how-indeed-used-terraform-in-its-move-to-aws) | 数十团队、数百工程师；数年计划压缩到约一年 | Guild、自助 workspace、版本债、避免巨型 workspace | HashiCorp 托管公司演讲 |
| [Segment：Terraform abstractions](https://www.hashicorp.com/resources/terraform-abstractions-safety-power/) | 349 服务、峰值 14,000 容器、约 30–50 次 infra apply/日（演讲时） | 模块、state 和“默认容易”的工程平台 | HashiCorp 托管公司演讲、历史 |
| [Segment：100 AWS accounts](https://segment.com/blog/secure-access-to-100-aws-accounts/) | 100 个 AWS 账号 | hub role、团队角色、账号访问体验 | 公司自述、历史 |
| [Airbnb：Data Protection Platform](https://medium.com/airbnb-engineering/automating-data-protection-at-scale-part-1-c74909328e08) | PB 级数据、多 AWS 账号、生产 S3 | Terraform 元数据 + S3 Inventory 的治理组合 | 公司自述 |
| [Figma：内部应用安全](https://www.figma.com/blog/inside-figma-securing-internal-web-apps/) | AWS + Okta；内部应用入口 | 跨 provider 安全模块与权限语义 | 公司自述 |
| [Figma：迁移 Kubernetes](https://www.figma.com/blog/migrating-onto-kubernetes/) | 12 个月内迁移大部分核心服务 | 把高频服务定义移出 Terraform；考虑 ACK | 公司自述 |
| [Spotify：Declarative Infrastructure](https://engineering.atspotify.com/2023/05/fleet-management-at-spotify-part-2-the-path-to-declarative-infrastructure/) | 数十万云资源、数万服务 | Terraform 不满足 runtime introspection 和配置自动改写 | 公司反例、GCP 为主 |
| [Spotify：Fleet-wide Refactoring](https://engineering.atspotify.com/2023/5/fleet-management-at-spotify-part-3-fleet-wide-refactoring) | 2022 年生成 27 万+ PR | 平台必须支持全舰队变更，而不只逐 repo review | 公司反例 |
| [Cloudflare：企业 IaC](https://blog.cloudflare.com/shift-left-enterprise-scale/) | 数百账号、约 30 MR/日 | state 独立密钥、OPA、漂移工单、provider dogfood | 厂商自述、非 AWS |
| [Datadog：Provider v4](https://www.datadoghq.com/blog/datadog-terraform-provider-v4/) | 2026 provider 大版本 | AWS integration 资源合并、权限和 secret 行为变化 | 厂商自述 |
| [GitLab：Runway](https://about.gitlab.com/blog/building-gitlab-with-gitlab-a-multi-region-service-to-deliver-ai-features/) | 多区域内部应用平台 | manifest→独立 state→健康门禁→自动回滚 | 厂商兼内部平台、GCP |
| [DoorDash：Conftest](https://www.hashicorp.com/resources/terraform-code-reviews-supercharged-with-conftest) | Security Group PR 审查瓶颈 | OPA 自动化低风险安全审批 | HashiCorp 托管公司演讲、历史 |
| [Canva：Infrastructure is Distribution](https://www.canva.dev/blog/engineering/infrastructure-is-distribution/) | 当时 Infra 约 200 人 | 私有 PaaS 是多供应商整合层，不只是 module 仓库 | 公司自述 |
| [Monzo：安全平台](https://monzo.com/blog/2022/03/31/how-we-secure-monzos-banking-platform) | 银行平台；AWS 变更需同行评审 | 生产审批、staging 自动应用、审计与自助 guardrail | 公司自述 |
| [Nubank：Immutable Infrastructure](https://aws.amazon.com/blogs/mt/leveraging-immutable-infrastructure-nubank/) | 3,000+ CloudFormation stack（文章时） | Terraform 不是 AWS-only 唯一答案；蓝绿不可变平台 | AWS 联合反例 |
| [Nubank：21k databases](https://building.nubank.com/how-nubank-distributes-infrastructure-ownership-to-operate-more-than-21-thousand-databases-with-a-lean-team/) | 21,000+ 数据库 | 声明式平台、分散所有权和自动治理 | 公司自述、当前 |

不要按“公司名气”排序阅读。对 Terraform × AWS 最具可操作性的四篇主轴是 Slack、Pinterest RPP、Grab 和 Indeed；Spotify/Figma/Nubank 用来防止把 Terraform 变成宗教。

---

## 3. State 与账号边界：从文件管理升级成资源身份系统

### 3.1 Slack 的 1,400 个 state 是怎么长出来的

Slack 早期只有一个 AWS 账号，结构很直观：

```text
environment
├── global        # IAM、CloudFront 等
├── us-east-1
├── us-west-2
└── ...
```

增长后出现几个现实约束：

- 数万 EC2 让单账号和 EC2 控制台变得难以操作；
- AWS API rate limit 变成平台级约束；
- 多团队在一个账号里难以做清晰 access control；
- 单 state 资源越多，refresh/plan 越慢，误操作影响越大；
- 团队需要拥有自己的部署流水线，而中央 Ops 无法人工代理所有改动。

Slack 最终形成：

```text
AWS organization
└── child account (team/service boundary)
    ├── global state
    ├── region A state
    ├── region B state
    └── large service isolated state
```

约 1,400 个 state 不是目标数字，而是组织和云边界自然展开后的结果。Cloud Foundations 团队管理 Terraform CLI/provider 版本、基础 module 和工具；服务团队拥有具体 state 与 pipeline。

### 3.2 一个实用的拆 state 判断矩阵

| 维度 | 应拆的信号 | 不必拆的信号 |
|---|---|---|
| 所有权 | 不同团队值班、审批、预算 | 同一团队完整拥有 |
| 权限 | 需要不同 AWS role/SCP/KMS | 完全相同权限 |
| 生命周期 | VPC 几月一改，应用每天改 | 同步创建和销毁 |
| 故障域 | 任一误删会波及无关系统 | 资源必须一起恢复 |
| 性能 | plan/refresh 已拖慢反馈 | 图小且稳定 |
| 配额 | 账号/区域配额互相影响 | 不存在配额冲突 |
| 数据敏感度 | state 读取边界不同 | state 可见性相同 |
| 依赖密度 | 接口少、可稳定发布 | 强循环依赖，硬拆只会造脆弱编排 |

**拆分目标不是 state 越多越好，而是让一次锁、一次 plan、一次权限授予和一次灾难恢复都落在同一责任域。**

### 3.3 Grab 的事故：backend key 是生产身份

Grab 早期让使用者复制一个现有 Terraform 项目作为起点。有人忘记修改 remote state 地址，CI 便把新代码解释成“修改旧项目”，导致既有资源被原地替换。

这个事故揭示：

```text
目录名 ≠ workspace 身份
repo 权限 ≠ state 权限
code owner ≠ AWS owner
```

如果 backend 只靠工程师复制和修改，它本质上是一个可伪造的资源身份声明。

### 3.4 Pinterest 的修正：把映射放进中央目录

Pinterest RPP 维护中央 source of truth，把每个 workspace 映射到：

- repo；
- root path；
- owner/team；
- S3 backend bucket；
- state key；
- KMS key；
- execution role。

流水线在 `terraform init/plan` 前验证 root module 的 backend 是否与目录完全一致。复制了错误 backend、改了 key 或引用另一 workspace 的 KMS 都会立即失败。

建议把 workspace catalog 看成授权数据库：

```yaml
workspaces:
  payments-prod-us-east-1:
    repo: company/infra-payments
    path: terraform/prod/us-east-1
    owner: payments-platform
    account_id: "111122223333"
    backend:
      bucket: org-tfstate-payments-prod
      key: payments/us-east-1/terraform.tfstate
      kms_key_arn: arn:aws:kms:us-east-1:111122223333:key/...
    execution_role_arn: arn:aws:iam::111122223333:role/tf-payments-prod
```

机器验证至少要回答：

1. 当前 repo/path 是否注册？
2. 触发者是否属于允许团队？
3. backend bucket/key/KMS 是否精确匹配？
4. 目标 AWS account ID 是否匹配 role？
5. plan 使用的 provider identity 是否与 catalog 一致？
6. apply 是否仍对应同一 commit、同一 catalog revision？

### 3.5 当前 S3 backend 校正

Slack、Gruntwork 和不少历史文章使用 S3 + DynamoDB 锁。当前 Terraform 官方文档已经把 DynamoDB locking 标为 deprecated；新配置应优先用 S3 lockfile：

```hcl
terraform {
  backend "s3" {
    bucket       = "org-tfstate-prod"
    key          = "payments/us-east-1/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
    kms_key_id   = "arn:aws:kms:us-east-1:111122223333:key/..."
  }
}
```

同时：

- 开启 S3 bucket versioning；
- state object 与 `.tflock` 分别做最小权限；
- 读取 state 也需要授权，不只是写；
- state 与 workload 最好不共用同一个宽权限角色；
- backend 凭据不要硬编码在 HCL 或 `-backend-config` 日志里；
- 做恢复演练，不要把“有版本”误当成“会恢复”。

官方依据：[Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)。

### 3.6 CLI workspace 的边界

Terraform CLI workspace 是同一 working directory 和同一 backend 下的多份 state。官方明确说它不适合需要不同凭据和 access control 的强隔离。

适合：

- 临时 feature environment；
- 同一权限边界内、相同拓扑的平行实例；
- 可快速销毁的测试副本。

不适合：

- prod 与 dev 的强权限隔离；
- 两个业务团队的所有权隔离；
- 网络、数据库、应用等系统分解；
- 不同 AWS 账号与 KMS 边界。

官方依据：[Terraform CLI workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)。

---

## 4. CI/CD：真正的安全边界在执行器

### 4.1 Pinterest RPP 的身份链

Pinterest 的思路可以抽象为：

```text
GitHub PR
   │ OIDC（短期、可验证 repo/ref/workflow）
   ▼
central RPP actions role
   │ catalog 验证 repo/path/workspace
   ▼
down-scoped team/workspace execution role
   │
   ├── terraform plan → PR
   └── explicit apply authorization → terraform apply
```

价值不只是“没有静态 key”：

- 中央入口可以统一记录谁、哪次 workflow、哪个 commit；
- role chaining 让每个 workspace 只拿所需权限；
- repo/path 不能任意选择高权限 role；
- runner 漏洞与 workload 权限不必绑定成组织管理员；
- 所有策略、backend 校验和 plan 发布集中升级。

### 4.2 Plan 与 Apply 的完整性

很多团队做到“PR 里能看到 plan”，但还缺以下闭环：

| 要素 | 为什么必须绑定 |
|---|---|
| Git commit SHA | 防止批准 A、应用 B |
| `.terraform.lock.hcl` | 防止 provider 版本变化 |
| module artifact/version | 防止 module source 漂移 |
| 变量与环境 | 防止 plan/apply 输入不同 |
| workspace catalog revision | 防止审批后角色或 backend 被改 |
| AWS caller identity | 防止计划和应用进不同账号 |
| plan artifact hash | 防止重新 plan 得到不同动作 |
| 审批与 apply 触发者 | 满足双重控制和审计 |

高风险环境的理想动作：

```text
validate → static policy → local integration → remote plan
→ human review → explicit apply authorization
→ apply saved plan → service health probes → evidence archive
```

### 4.3 只读本地入口

Slack 的 Ops box 允许工程师运行 plan，但只有只读 AWS 权限，不能直接 apply。这个设计承认两件事：

- 工程师需要快速调查和理解；
- 生产写权限不应因此分发到每台电脑。

本地 `terraform plan` 仍可能读取敏感 data source/state，因此“只读”不等于“无敏感权限”。state read、Secrets Manager/KMS decrypt 和参数读取要另外约束。

### 4.4 Smart Planner：检查消费者，不只是改动目录

共享 module 改一行，真正受影响的是所有消费者。Slack 的 Smart Planner 会：

- 识别 module/state 改动；
- 找出直接和间接受影响的 state；
- 对这些 state 运行 plan；
- 把结果贴回 PR；
- plan 失败时阻止 merge。

这比“每个 root 跑 `terraform validate`”高一个数量级，因为 `validate` 不知道：

- 真实 provider schema 与远端对象；
- 某 module 的全部调用组合；
- 跨 state 输出兼容性；
- replace 动作对业务的影响；
- 使用旧 provider/CLI 的消费者。

平台至少应维护：

```text
module version → consumer roots
state output    → downstream consumers
provider range  → affected roots
owner           → notification/escalation
```

### 4.5 DoorDash：把审查意见变成政策

DoorDash 的典型瓶颈是 AWS Security Group PR：

1. 开发者提交；
2. 等安全团队；
3. 分支落后后重排；
4. 再等一次安全审批。

他们用 Conftest/OPA 编码预定义安全政策。符合政策的变更无需人工安全审批；只有违反政策或例外才升级给安全团队。

高价值原则：

- 先自动化重复、确定、可解释的 reviewer 意见；
- policy 输出要指出资源地址、字段和修复方向；
- 例外必须有 owner、理由和到期时间；
- policy 本身也需要测试、版本和灰度；
- 不要把“扫描器通过”写成“系统安全”。

### 4.6 Monzo：审批与环境推进

Monzo 公开描述 AWS 变更需要同行评审，Concourse 自动应用 staging，production 需要 owning team 审批和 PR merge。这里的关键不是工具，而是：

- 审批者与资源 owner 对齐；
- staging 是自动、重复执行的门；
- production 不是工程师本机的下一条命令；
- 变更记录、身份和运行结果进入审计链。

---

## 5. `apply` 成功不等于系统正确：Slack DNSSEC 的尖锐教训

### 5.1 事故链

Slack 的 DNS 都由 Terraform 管理。在 DNSSEC 回滚时：

- Route53 控制台会显示一个关键警告；
- Terraform/API 执行路径没有把该风险以同等强度呈现给操作者；
- `.com` 上层区域的 DS record 已被递归解析器缓存，TTL 最长约 24 小时；
- Slack 停止 zone signing 后，仍缓存 DS 的 validating resolver 开始返回 `SERVFAIL`；
- 外部 DNS/DNSSEC 探针立即告警；
- 团队联系公共 resolver/operator 清缓存。

### 5.2 可迁移到所有 AWS 服务的结论

```text
API 接受变更
     ≠ 控制台所有警告被看到
     ≠ 分布式缓存已收敛
     ≠ 数据面工作正常
     ≠ 回滚回到了原状态
```

类似风险还包括：

- Route53/CloudFront 的缓存和传播；
- IAM policy 生效延迟；
- KMS key policy 把自己锁出；
- RDS 参数或 engine upgrade 的不可逆行为；
- S3 Object Lock、versioning、replication 的语义；
- EKS addon/controller 与 Terraform 的所有权冲突；
- Security Group/NACL 路径组合后的实际连通性；
- 删除保护或 retained resource 的恢复假象。

### 5.3 对流水线的具体要求

危险资源应额外具有：

- `precondition` / policy-as-code 防护；
- 变更前外部状态快照；
- 服务专属 runbook；
- 带 TTL/传播窗口的等待条件；
- 从多个外部网络执行的 synthetic probe；
- 失败时的前向修复方案，而不只 `git revert`；
- break-glass 联系人和第三方依赖清单。

`git revert` 只撤销期望配置，不能撤销已经传播到外部世界的缓存、邮件、证书、数据删除或客户端行为。

---

## 6. Module 不是 DRY 工具，而是内部产品 API

### 6.1 Deliveroo：模块应交付“可运行应用”

Deliveroo 的应用 module 通过少量参数建立：

- ECR；
- CircleCI build/push；
- S3、IAM、CloudWatch；
- 内部 release manager 注册；
- ALB；
- worker autoscaling；
- New Relic dashboard；
- ECS placeholder task；
- Route53 与 Cloudflare DNS。

数据库 module 则内置：

- 加密；
- backup retention；
- staging/prod 不同实例规格；
- production replica；
- secret 注入。

这类 module 的 API 是：

```hcl
module "application" {
  source    = "..."
  name      = "payments"
  repo_name = "payments-service"
}
```

而不是：

```hcl
module "thin_ec2_wrapper" {
  # 把 aws_instance 的 50 个字段原样暴露一次
}
```

真正价值是把组织决策封装进去：日志、监控、加密、备份、命名、owner、成本标签和发布系统。

### 6.2 Figma：跨 provider 组合是 Terraform 的强项

Figma 为内部应用构建了组合 module：

```text
Okta SAML app
      │
AWS Cognito user pool/client
      │
AWS ALB authentication action
      │
internal application
```

Terraform 同时管理 AWS 与 Okta，module 让基础设施工程师快速获得：

- 禁止自注册的 Cognito；
- SAML attribute mapping；
- ALB HTTPS authentication；
- Okta group 到应用权限的映射；
- 统一 session/security 默认值。

这是 Terraform 比单云模板更有吸引力的地方：它能把 AWS、SaaS 身份、DNS 和监控连接成一个依赖图。

但跨 provider 也放大失败面：

- 两个 API 的一致性和 rate limit 不同；
- 一边成功、一边失败时没有真正分布式事务；
- provider 升级节奏不同；
- destroy 可能先删掉仍被另一侧依赖的对象；
- state 同时包含多个系统的敏感元数据。

### 6.3 Grab：Raw Terraform 自助为什么失败

Grab 初始方案：

```text
使用者写 Terraform MR
→ Coban 团队逐个 review
→ Atlantis plan/apply
```

出现四类问题：

- **稳定性**：错误复制 backend 导致资源替换；
- **扩展性**：平台团队成为所有 MR 和失败支持的瓶颈；
- **安全**：没有资源级 IAM/所有权，code review 是唯一 guardrail；
- **采用率**：只服务懂 Terraform 的人。

Grab 建立三层平台：

```text
Coban UI
   │ ClickOps / self-service
Heimdall API
   │ auth、validation、request lifecycle、MR
Khone Git repo
   │ metadata + vetted Terraform modules + CI
Terraform/providers
   │
AWS / Kubernetes / Kafka / ...
```

关键设计：

- Git/Khone 是唯一持久 source of truth；
- UI 和 API 不直接悄悄改基础设施；
- production 需要资源 code owner 审批；
- staging 可自动；
- backend/provider 环境由 CI 生成，使用者不能随意填；
- 只允许经过审核的 module；
- 路径推导资源名、环境和 cluster；
- metadata 保存 owner 与成本归属；
- 高级用户仍可直接走 API/Git 做批量变更。

### 6.4 Terraform 做后端的两个恢复优势

Grab 保留 Terraform 而没有让平台直接调用各类 API，给出的理由很强：

1. **灾难恢复**：即使 UI/API 控制面坏了，有权限的人仍可从 Git 里的 Terraform 重建资源，降低控制面本身对 RTO 的绑架。
2. **生态适配**：AWS/Kubernetes/Kafka API 变化主要由 provider 吸收，平台不用为每个 API 自建完整 lifecycle engine。

但这只有在以下条件成立时才是真的：

- Git 中代码足以重建；
- state/backend 可恢复；
- secret 和外部依赖有恢复路径；
- provider/module 版本可取得；
- 关键 artifact 没有消失；
- 团队定期演练绕过 UI 的恢复流程。

### 6.5 Canva：基础设施平台是“分发系统”

Canva 把成长阶段分成：

- **Outsourced**：主要依赖单个云/PaaS；
- **Integrated**：整合 AWS、GitHub、Buildkite、Datadog、Snowflake 等；
- **Bespoke**：规模大到自研系统更经济。

Integrated 阶段的平台团队不应把 Terraform module 数量当成产出；其真正产品是一个私有 PaaS，把多供应商的：

- 身份；
- 网络；
- 可观测性；
- 安全；
- 成本；
- 生命周期；
- 失败恢复

组合成一致的开发者体验。

---

## 7. 组织与采用：Indeed 的紧急迁移课

### 7.1 同时学 AWS 和 Terraform 会放大混乱

Indeed 原计划用三到五年迁 AWS。2021 年 Texas 冬季风暴后，目标压缩成约六到八个月的执行窗口。数十团队、数百工程师/SRE 同时迁移，很多人既不熟 AWS，也几乎没听过 Terraform。

这说明培训要拆成两个维度：

| 能力 | 典型问题 |
|---|---|
| Terraform 模型 | state、plan、unknown、replace、module、provider |
| AWS 服务语义 | IAM、网络、DNS、数据持久性、配额、传播、成本 |

会写 HCL 不代表理解 `aws_iam_policy` 的权限；会点 AWS Console 也不代表理解 state address 和 module upgrade。

### 7.2 Guild 不是中央审批委员会

Indeed 的 Terraform Guild 负责：

- 语言和 module 标准；
- 文档与培训；
- 支持频道；
- 代表内部使用者向平台团队反馈；
- 版本与迁移沟通；
- 与安全团队共同建立治理。

他们强调 Guild 不应与 Terraform 运行平台团队完全重叠。原因是：

- 平台团队负责服务可靠性和规则执行；
- Guild 代表使用者体验和实践演化；
- 两者完全重合会让“服务提供者”替“客户”定义全部需求。

### 7.3 自助 workspace 也要自助治理

Indeed 使用 TFE provider + YAML 让工程师自助创建 workspace 和分配团队，Git 保留审计。自助不等于无限：

- 谁能创建；
- 命名和 owner；
- 默认 state 可见性；
- 何时拆/合；
- 过期 workspace；
- 成本；
- provider/CLI 版本；
- module 采用率

都需要指标和回收机制。

### 7.4 避免“一条真理路径”

Indeed 的 cloud desktop 场景在一个 workspace 里资源越来越多：

- refresh 花数分钟；
- 内存持续增长；
- targeted run 只是缓解；
- 每实例一个 TFE workspace 又受行政/许可成本限制。

他们因此探索 Terragrunt + GitLab CI 等替代 paved road。高论是：

> 平台应有少数明确支持的路径，但不能为了统一而把不匹配的生命周期塞进同一执行模型。

### 7.5 避免巨型代码库与知识单点

Indeed 曾有一份能一次建立完整 datacenter 基础设施的巨大代码。核心架构师离职后：

- 机构知识丢失；
- blast radius 让新维护者不敢改；
- workspace 变得脆弱、难重构。

他们的反思是拆成独立/紧密相关组件，并通过稳定 module/标签发现等方式发布接口，而不是把下游直接绑到上游内部输出。

### 7.6 中心化什么，分散什么

| 中心化 | 分散给资源团队 |
|---|---|
| 身份入口与 runner | 业务资源定义 |
| workspace catalog | 资源 SLO 与值班 |
| backend 规则 | 变更内容和节奏 |
| provider/CLI 支持窗口 | 业务特定测试 |
| 基础 module/policy | 成本取舍 |
| audit/metrics/drift | 例外的业务理由 |
| 升级协调 | 采用和反馈 |

中心团队逐行 review 所有 HCL，是最昂贵也最难扩展的控制。

---

## 8. Terraform 代码也是治理数据，但不是全部现实

### 8.1 Airbnb：代码元数据 + 真实 inventory

Airbnb 的 Data Protection Platform 需要理解多 AWS 账号里的 S3：

- Terraform repo 中保存账号和 bucket 配置；
- 每个 bucket 带 project/owner 等元数据；
- crawler 解析 Terraform，得到账号与 bucket；
- 所有生产 bucket 通过 Terraform 启用 S3 Inventory；
- crawler 读取每日/每周 Inventory，而不是高成本遍历 List API；
- 再与数据分类、MySQL、Hive 等元数据汇合。

这套设计没有犯“Git 就是现实”的错误：

```text
Terraform code        → intended ownership/configuration
Terraform state       → managed identity/history
AWS S3 Inventory      → actual objects
classification system → data sensitivity
org metadata          → current team/service owner
```

### 8.2 元数据要进入强制路径

推荐每个 root/resource 至少有：

```yaml
owner_team: payments-platform
service_id: payments-api
environment: production
cost_center: cc-1234
data_classification: confidential
pagerduty_service: payments-prod
repository: company/payments-infra
expires_at: null
```

这些字段应：

- 由 schema 校验；
- 映射成 AWS tags；
- 进入 catalog 和成本平台；
- 与组织目录定期对账；
- 人员/团队重组时自动迁移；
- 缺失或失效时阻止新的生产变更。

### 8.3 Drift 是工作流，不是一次检测

Cloudflare 的生产 IaC 模式值得借鉴：

- 关键生产修改只走代码；
- 持续 drift detector 比较 Terraform 与实际 API；
- drift 自动创建内部工单；
- 按严重度设置修复 SLA；
- 工单分配给 owner。

一个完整 drift 系统要区分：

| 漂移类型 | 处理 |
|---|---|
| 紧急 break-glass | 先恢复服务，再自动生成回写 PR |
| 合法外部 controller | 明确把字段/resource 移出 Terraform owner |
| 未授权 ClickOps | 告警、回滚或工单 |
| provider read 归一化差异 | 修 provider/忽略特定字段 |
| AWS 默认值变化 | 评估后显式声明或接受 |
| 已废弃 orphan | owner 确认后 import/delete |

“每晚跑 `terraform plan`”只是检测器，不是 drift 管理系统。

---

## 9. 版本与 provider 迁移：一次组织级依赖升级

### 9.1 Slack：从季度级痛苦到批量升级工具

Slack 的 Terraform 0.11→0.12：

- 花了一个季度；
- module 复制成 `-v2`；
- wrapper 根据 `versions.tf` 选择 Terraform binary；
- 旧、新版本并存；
- 最后再清理旧 binary 与 wrapper 逻辑。

后续升级时，他们建立 Go 工具：

- 检查当前 CLI/provider 版本；
- 检查是否有未 apply 变更；
- 找出 remote-state 依赖；
- 升级后运行 plan；
- 解析 HCL 与 module dependency tree；
- 可按百分比批量升级 state。

这把升级从手工项目变成可重复平台能力。

### 9.2 升级波次

```text
0. inventory
   └─ CLI/provider/module/state/output consumers
1. compatibility lab
   └─ LocalStack + provider schema + static tests
2. canary roots
   └─ sandbox、低风险、代表性资源
3. low-risk batch
4. production cohort
5. long-tail / exceptions
6. old version removal
```

每波应观察：

- plan error rate；
- unexpected replace；
- refresh 时间；
- apply failure；
- provider crash；
- API rate limit；
- drift 增量；
- 回滚或 state surgery 次数。

### 9.3 Datadog v4：资源所有权重构

2026 年 Datadog provider v4：

- 用 `datadog_integration_aws_account` 替代四个 v3 AWS integration resources；
- 统一 AWS account integration 的 source of truth；
- 调整 monitor permission 语义；
- 移除读取/导入既有 application key 的部分路径；
- 迁移到 Terraform protocol v6。

最危险的不是 HCL 改名，而是：

```text
legacy resources ─┐
                  ├─ 同一个 Datadog AWS integration 对象
new unified one  ─┘
```

如果新旧 resource 同时宣称所有权，可能互相覆盖。迁移检查：

1. 列出旧 resource address 与远端对象；
2. 阅读 provider 的 import/migration 指南；
3. 决定使用 `moved`、import block 还是 state move/remove；
4. 在无写入或 canary 账号验证 plan；
5. 确认 plan 不是 destroy/recreate integration；
6. 检查 monitor permission 变化；
7. 检查 state 中既有 secret；
8. 分批推进，不把 100+ AWS accounts 一次切完。

### 9.4 Provider 是供应链和 API 适配层

Cloudflare 的公开经验补充了 provider 维护现实：

- 手工 provider 很容易落后产品 API；
- 大规模 dogfood 才暴露边缘问题；
- OpenAPI 生成能降低 feature parity 滞后；
- provider major version 仍会带来迁移摩擦；
- import 遗留 ClickOps 是长期工程，不是一条命令。

因此，选择 AWS/SaaS provider 时要看：

- schema 与 API 覆盖；
- issue/PR 响应；
- major upgrade 迁移工具；
- import 能力；
- read-after-write 和 eventual consistency 处理；
- state upgrader；
- 敏感字段行为；
- 维护权和 release cadence。

---

## 10. AWS 专属高论

### 10.1 多账号首先是故障和配额边界

Slack 从单账号迁出，不只是 IAM 整洁：

- EC2 console 与 API 可操作性；
- AWS rate limits；
- 团队 access control；
- blast radius；
- 成本与 quota 归属

都在推动多账号。

Pinterest 的迁移又提醒：AWS Control Tower/AFT 等现成方案未必能无痛接管历史 estate。账户工厂设计前先 inventory：

- 既有 Organizations/OU/SCP；
- 共享 VPC/Transit Gateway；
- IAM Identity Center；
- DNS；
- 日志与 Security Hub/GuardDuty；
- Terraform 与非 Terraform 资源；
- 账号创建后的人工/脚本 side effects。

### 10.2 Global resource 要独立思考

IAM、CloudFront、Route53、Organizations、SCP 等：

- 不完全按普通 AWS region 生命周期工作；
- 影响面常跨所有区域；
- 更新频率和审批更低；
- 权限更高；
- 错误恢复更慢。

因此不要因为 repo tree 简单，就把它们和每个 region 的应用资源塞进同一 state。

### 10.3 IAM：身份链比 policy 文本更重要

审查 IAM 不能只看最终 policy JSON，还要看：

- 谁能触发 workflow；
- OIDC trust 限制了哪些 repo/ref/workflow/environment；
- 中央 role 能 Assume 哪些下游 role；
- 下游 role 是否能横向 Assume；
- session tags/source identity 是否保留；
- state/KMS 权限是否比 workload 写权限更宽；
- fork PR 是否能获取凭据；
- reusable workflow 是否固定到不可变版本；
- runner 是否可能执行不受信代码。

### 10.4 S3：资源创建只是治理起点

Airbnb 的经验说明 S3 module 应考虑：

- owner/project tags；
- data classification；
- inventory；
- encryption/KMS；
- public access block；
- versioning/lifecycle；
- replication；
- access logging/CloudTrail data events；
- object ownership；
- retention/Object Lock；
- 成本和过期回收。

一个只包 `aws_s3_bucket` 名字的 module 几乎没有平台价值。

### 10.5 Terraform 与 Kubernetes/应用发布的生命周期

Figma 在 ECS 时，改一个环境变量要：

1. 先改 Terraform；
2. apply 一个零实例模板 task set；
3. 再用部署系统替换 image 并上线。

顺序容易被忘，产生 bug。迁 EKS 后，服务定义进入一处 Bazel 配置并生成 Kubernetes YAML，由内部部署系统应用；他们还考虑用 AWS Controllers for Kubernetes 管更多 AWS resource。

通用边界：

| Terraform 更适合 | 应用/控制器更适合 |
|---|---|
| AWS account、VPC、KMS、基础 IAM | Deployment、Pod、HPA |
| EKS cluster/control-plane 基础 | 高频 service config |
| 低频 RDS/S3/SQS 基座 | image/version/rollout |
| 跨 provider 身份与监控接线 | 与应用 release 同步的资源 |
| 需要显式审批的长期资源 | 持续 reconciliation 的对象 |

不要让两个控制面同时拥有同一字段。若 ACK/controller 管一个 AWS object，就应明确 Terraform 只负责 controller/permission 或完全移交该 resource。

---

## 11. 什么时候不该继续加 Terraform

### 11.1 Spotify：需要把配置当“可批量修改的数据”

Spotify 管理数十万云资源、数万服务，提出三个要求：

1. GitOps、评审和审计；
2. runtime introspection：枚举所有数据库、政策违规、owner，并支持 break-glass；
3. 配置必须是 JSON/YAML 这类数据，以便程序安全地做全舰队自动修改。

他们明确认为 Terraform 不满足第 2、3 项：

- 分散 state/HCL 不适合全量运行时查询；
- 自动修改“求值后得到某结果的代码”比修改结构化数据难；
- 希望最终用高阶 custom resource 替代大量 raw cloud resource。

最终选择 Kubernetes API/custom resources 作为声明式基础设施控制面。

这不是说 Kubernetes CR 永远更好，而是说明：

```text
逐 workspace 计划与批准    → Terraform 强
持续 reconciliation       → controller 强
全舰队枚举与策略查询       → 统一 API/control plane 强
自动批量改写声明           → data model 强
跨任意 SaaS provider       → Terraform 生态强
```

### 11.2 Figma：高频服务定义移出 Terraform

如果一个变更必须与应用 image 一起发布、回滚和观察，单独走 Terraform pipeline 会制造双重时序。Figma 的迁移说明：资源 owner 应贴近其实际 lifecycle，而不是因为“以前都在 Terraform”就永远留下。

### 11.3 Nubank：AWS-only 可以选 CloudFormation

Nubank 用自研 Clojure 工具 Nimbus 生成/操作 CloudFormation：

- 3,000+ stack；
- 声明式 source of truth；
- 不可变基础设施；
- 蓝绿创建新资源；
- 健康检查后删除旧资源；
- 失败时保留/回退旧版本。

其意义不是证明 CloudFormation 胜过 Terraform，而是：

- AWS 托管 stack/rollback 是一种不同执行语义；
- 高阶内部平台比底层模板语言更决定开发体验；
- 纯 AWS 团队可用 native control plane 换取更紧的服务集成；
- 迁移成本、团队能力和既有平台资产往往比语法偏好更重要。

### 11.4 一个停止扩张的检查表

遇到以下情况，先别再加 Terraform resource：

- 同一对象已有 Kubernetes operator/controller；
- 资源每次应用发布都变化；
- 必须持续 reconciliation 而不是按次 apply；
- 需要毫秒/秒级自动扩缩或故障反应；
- 必须全舰队结构化查询和自动修改；
- provider 长期落后 API，团队频繁用 `local-exec` 补洞；
- import/read 语义不稳定；
- 删除/回滚有强服务专属状态机；
- 另一个 AWS 托管控制面已提供更可靠的事务/回滚；
- 为了统一而引入的 pipeline 比资源本身更复杂。

---

## 12. 反复出现的失败模式

### 12.1 身份与 state

- 复制 root module 忘改 backend；
- 两个目录指向同一 state；
- repo/path 可以任意选择执行 role；
- dev/prod 用 CLI workspace 假装强隔离；
- state bucket 读权限过宽；
- 只有 state versioning，没有恢复演练；
- 手工 state surgery 无审计。

### 12.2 Module

- 共享 module 直接引用相对路径，merge 即影响全部消费者；
- module 无版本、owner 和 changelog；
- 版本锁死，安全修复无法全舰队推进；
- mega-module 条件分支比 AWS API 更难理解；
- wrapper 原样暴露所有参数，却制造额外升级层；
- 下游直接读上游 remote state 内部结构；
- module 改动只测一个 happy path。

### 12.3 CI 和权限

- 静态 AWS key 长期保存在 CI secret；
- plan 与 apply 不是同一 commit/输入；
- PR 来自 fork 也能拿高权限；
- runner 执行不受信脚本后再取得生产 role；
- 安全团队人工 review 每条普通规则；
- apply 后没有数据面 probe；
- break-glass 后没有回写 Git/state。

### 12.4 组织

- 平台团队逐行审全部 HCL；
- 工程师同时学习 AWS 和 Terraform，却没有分层培训；
- 只有文档，没有支持社区和 owner；
- workspace/state 没有目录和指标；
- 巨型 workspace 的关键作者离职；
- 只允许“一条真理路径”，使用者最终绕过；
- 成本、owner 和到期信息不进入机器数据。

### 12.5 工具边界

- Terraform 与 operator 同时改同一对象；
- 把应用 image/env 高频发布塞进低频 infra apply；
- 把 console warning 缺失当成 API 安全；
- 认为 `git revert` 可以撤销外部传播；
- 把 LocalStack 通过当成真实 AWS 全等；
- provider major upgrade 只做文本替换；
- 把所有 ClickOps 漂移一律自动回滚，破坏紧急修复。

---

## 13. 一套可落地的目标架构

```text
                     ┌─────────────────────────┐
                     │ Workspace Catalog        │
                     │ path/state/role/owner    │
                     └───────────┬─────────────┘
                                 │ validate
Developer ──PR──► CI Orchestrator/RPP
                      │
                      ├─ fmt/validate/lint
                      ├─ OPA/Semgrep/custom policy
                      ├─ LocalStack integration tests
                      ├─ impacted-consumer discovery
                      └─ OIDC → central role → scoped role
                                           │
                              remote plan ─┴─► PR evidence
                                                   │
                                 owner/security approval
                                                   │
                                      explicit apply trigger
                                                   │
                                      apply saved plan
                                                   │
                      ┌────────────────────────────┼──────────────────────┐
                      ▼                            ▼                      ▼
                AWS control plane           external probes       audit/evidence
                      │                            │                      │
                      └──────── actual inventory/drift ──────────────────┘
                                                   │
                                            owner ticket / PR
```

### 13.1 四个不可缺的数据库

1. **Git**：期望配置和审批历史；
2. **Terraform state**：逻辑地址与远端对象身份；
3. **Workspace catalog**：组织授权与 owner；
4. **AWS inventory/telemetry**：真实资源和数据面健康。

任何一个都不能完全替代其他三个。

### 13.2 最小平台 API

平台至少应能回答：

```text
Who owns this resource?
Which state owns it?
Which repo/path can change that state?
Which AWS role can apply it?
Which module/provider versions produced it?
What plan/approval created the last change?
What depends on it?
Is it drifting?
How is it recovered?
What does it cost?
```

如果要靠某位老员工的记忆回答，平台还没有完成。

---

## 14. LocalStack Ultimate：把公司高论变成实验

LocalStack 官方支持通过 `tflocal` 把 AWS provider endpoint 指向本地服务，也支持 Terraform init hooks + Testcontainers。Pinterest 公开提到部分团队在 RPP 里用 LocalStack 预演复杂架构。

官方入口：

- [Terraform integration](https://docs.localstack.cloud/aws/connecting/infrastructure-as-code/terraform/)
- [Terraform init hooks + Testcontainers](https://docs.localstack.cloud/aws/tutorials/using-terraform-with-testcontainers-and-localstack/)

### 14.1 实验分层

| 层 | LocalStack 能力 | 仍需真实 AWS |
|---|---|---|
| HCL/provider schema | 很强 | 新 provider/边缘字段复核 |
| 资源依赖图 | 很强 | 服务真实异步行为 |
| module 组合 | 很强 | 配额、性能、区域差异 |
| state/backend 错配 | 很强 | S3/KMS/IAM 真实策略 |
| policy-as-code | 很强 | 组织 SCP/permission boundary |
| app integration | 强，视服务覆盖 | 控制面与数据面差异 |
| DNS/缓存传播 | 有限 | 真实 resolver/TTL/父区 |
| IAM eventual consistency | 有限 | 真实 STS/IAM |
| 成本 | 不产生真实账单 | CUR/Pricing/实际 usage |
| 灾难恢复 | 可验证流程 | 真实数据、备份和跨区恢复 |

### 14.2 实验 1：复现 Grab 的错 backend 事故

目标：证明“代码看起来是新项目”不代表 Terraform 认为它是新资源。

步骤：

1. 建 `service-a`，使用本地 S3 backend key `services/a.tfstate`；
2. apply 一个 SQS/S3 资源；
3. 复制目录为 `service-b`，修改资源名/参数，但故意保留 `a.tfstate`；
4. 运行 plan；
5. 观察 update/replace/delete；
6. 在 CI 前置 catalog validator 后重复，确认在 `terraform init` 前失败。

断言：

- validator 报出 expected/actual bucket、key、account；
- 不能只靠资源 tag 检测；
- 错配不产生任何 apply 权限；
- 日志不泄露 backend secret。

### 14.3 实验 2：Pinterest workspace catalog

实现一个最小 `workspace-catalog.yaml` 和验证器：

```text
changed path
   → exact workspace lookup
   → backend parse
   → KMS/key/bucket compare
   → expected account/role compare
   → owner/codeowner compare
```

负面测试：

- 路径穿越；
- 相似前缀欺骗；
- 大小写/URL encoding；
- symlink；
- 同 bucket 不同 key；
- 同 key 不同 bucket；
- prod path 指向 dev role；
- catalog 在 plan 后变化。

### 14.4 实验 3：Slack 式多 state 与依赖图

模拟：

```text
global-iam
network-us-east-1
network-us-west-2
service-a-us-east-1
service-a-us-west-2
observability
```

要求：

- 每个 root 独立 state；
- 输出只发布必要字段；
- module 改动能找出受影响 roots；
- 并行执行不跨越依赖；
- 上游输出 schema 变化使 CI 明确失败；
- 删除任一 state 时恢复顺序可执行。

然后比较：

- 一个大 state 的 plan 时间/影响面；
- 六个 state 的锁竞争/编排成本；
- remote state 直读与 SSM/DNS/标签发布接口。

### 14.5 实验 4：S3 lockfile 与恢复

用当前 backend 模式验证：

```hcl
use_lockfile = true
```

测试：

- 两个并发 apply，第二个获得锁失败；
- 人工中断后锁处理；
- state object 版本恢复；
- `.tflock` 权限与 state 权限分离；
- 恢复旧 state 后 plan 是否真的安全；
- state 中敏感字段扫描。

LocalStack 可验证流程；真实 AWS 还要验证 KMS key policy、bucket policy、S3 versioning/MFA Delete 或 Object Lock 设计。

### 14.6 实验 5：DoorDash 式风险分级

给 Security Group 写 OPA/Conftest 规则：

- 禁止 `0.0.0.0/0` 到管理端口；
- ingress 必须有 owner/service tag；
- production 只允许批准的端口和来源；
- 例外必须含 ticket、owner、到期时间；
- expired waiver 阻止 plan。

再做三类 PR：

1. 标准 Web ingress，自动通过；
2. 高风险管理端口，升级安全审批；
3. 合法临时例外，到期后自动失败。

衡量：

- 自动通过率；
- false positive；
- 人工 review 等待时间；
- 例外逾期数；
- 规则版本造成的批量影响。

### 14.7 实验 6：Deliveroo 式 capability module

做一个 `service` module，组合 LocalStack 中可支持的：

- ECR；
- ECS/Lambda 任选一条应用路径；
- ALB/API Gateway；
- IAM role；
- CloudWatch logs/alarms；
- Route53；
- S3/SQS；
- owner/cost/data tags。

只暴露业务选择：

```hcl
module "service" {
  name           = "orders"
  exposure       = "internal"
  data_class     = "confidential"
  scaling_profile = "burstable"
}
```

测试 module 默认是否真正形成：

- 最小权限；
- 加密；
- 日志与监控；
- 清晰 outputs；
- 可销毁临时环境；
- 不需要调用者理解 30 个底层 resource。

### 14.8 实验 7：Figma 式跨 provider 安全模块

LocalStack 负责 AWS 侧 ALB/Cognito/Lambda；Okta 侧用 mock HTTP provider、测试替身或只验证 module 接口。

重点不是复刻完整 Okta，而是测试：

- AWS/SaaS 一边失败时 Terraform 行为；
- import 既有 IdP；
- module destroy 顺序；
- JWT issuer/audience 配置；
- group→capability，而不是 group→org chart；
- secret/state 暴露。

真实 AWS/Okta 必须补 WebAuthn、SAML、JWT 签名和 ALB 真实行为测试。

### 14.9 实验 8：Airbnb 式 S3 治理

构建：

```text
Terraform bucket metadata
       +
LocalStack S3 actual buckets/objects
       +
inventory-like export
       ↓
governance reconciler
```

注入：

- Terraform 里有、AWS 里没有；
- AWS 里有、Terraform 里没有；
- owner tag 失效；
- bucket 有对象但已过期；
- public access 配置漂移；
- data classification 与 KMS 不匹配。

输出 owner 工单，而不是只打印终端警告。

### 14.10 实验 9：Slack DNSSEC 教训的通用化

LocalStack 不适合证明真实父区 DS 缓存，但可以验证流水线结构：

- 对危险资源识别 replace/delete；
- 要求显式 risk acknowledgment；
- apply 前执行 precondition；
- apply 后从 Terraform 之外执行 probe；
- probe 失败阻止环境推进；
- runbook/evidence URL 必填。

真实 AWS 用专用测试域名、小 TTL、多公共 resolver 做小规模演练，绝不能用生产主域首次验证。

### 14.11 实验 10：Provider 升级 canary

选择两个 AWS provider 版本：

1. 锁定旧版；
2. 对代表性 roots 保存 plan JSON；
3. 升级 provider；
4. 比较 action、replace、unknown、sensitive 和 schema；
5. LocalStack apply；
6. 小型真实 AWS sandbox canary；
7. 才扩大到生产 cohort。

结果应产生机器指标，而不是“肉眼看起来一样”。

### 14.12 实验 11：Terraform owner 与 controller owner 冲突

做一个简化控制器持续修改某个资源 tag/字段，同时 Terraform 也声明该字段：

- 观察每次 plan 都想改回；
- 给 Terraform `ignore_changes`，观察 drift 被隐藏；
- 最终明确字段 owner 或把整个 resource 移交。

目标是理解：`ignore_changes` 不是 ownership 设计，只是暂时停止争夺。

### 14.13 实验 12：平台控制面故障恢复

模拟 Grab 的设计目标：

1. 停掉 UI/API；
2. 只保留 Git、state、provider cache/module artifact；
3. 用受控 break-glass runner plan/apply；
4. 重建平台本身；
5. 对账 state、Git、实际资源；
6. 记录 RTO/RPO 和缺失依赖。

如果离开平台 UI 就无法恢复，Terraform 作为可移植 source of truth 的优势没有兑现。

---

## 15. 建议的本地实验仓库结构

```text
terraform-aws-company-blog-lab/
├── catalog/
│   ├── workspaces.yaml
│   └── schema.json
├── platform/
│   ├── validate-workspace/
│   ├── discover-consumers/
│   ├── policy/
│   └── drift-reconciler/
├── modules/
│   ├── service/
│   ├── governed-s3/
│   └── internal-app-gateway/
├── roots/
│   ├── global/
│   ├── network/
│   ├── services/
│   └── observability/
├── tests/
│   ├── backend-collision/
│   ├── policy/
│   ├── provider-upgrade/
│   ├── recovery/
│   └── ownership-conflict/
├── docker-compose.yml
├── Makefile
└── README.md
```

### 15.1 每个 root 的最小契约

```text
owner metadata
backend identity
required Terraform/provider range
dependency inputs
published outputs
risk classification
test command
recovery runbook
deprecation date
```

### 15.2 每次 PR 的证据包

```json
{
  "commit": "...",
  "workspace": "...",
  "catalog_revision": "...",
  "terraform_version": "...",
  "provider_locks_hash": "...",
  "caller_identity": "...",
  "plan_hash": "...",
  "policy_result": "...",
  "localstack_test_result": "...",
  "approvals": [],
  "apply_result": "...",
  "post_apply_probes": []
}
```

---

## 16. 一份生产成熟度评分表

每项 0–2 分：

| 维度 | 0 | 1 | 2 |
|---|---|---|---|
| Workspace catalog | 没有 | 手工表 | CI 强制、可查询 |
| Backend identity | 人工复制 | convention | path/state/role 强校验 |
| State recovery | 无 | 有版本 | 定期恢复演练 |
| Credentials | 静态 key | 部分 OIDC | OIDC + scoped role chain |
| Plan integrity | 终端输出 | PR plan | saved plan + commit/input 绑定 |
| Policy | 人工 review | lint/checklist | 测试化 policy + exception lifecycle |
| Module | 无 owner | 有版本 | consumer graph + rollout/弃用 |
| Drift | 不测 | 定时 plan | owner 工单/SLA/break-glass 回写 |
| Inventory | 只看 Git | Git+state | Git+state+AWS actual |
| Metadata | README | tags | schema+组织对账+成本/数据治理 |
| Testing | validate | LocalStack | LocalStack + 真实 AWS canary |
| Apply 后验证 | 无 | resource read | 数据面/外部 probe/SLO |
| 升级 | 临时手工 | 文档 | inventory、canary、批量波次、指标 |
| 平台恢复 | 依赖 UI | 有 runbook | 演练过的 Git/state break-glass |
| Ownership | 平台团队全包 | 混合 | 中央规则、分散业务责任 |

解释：

- **0–9**：脚本化阶段；
- **10–18**：团队 IaC；
- **19–24**：有平台雏形；
- **25–30**：组织级 Terraform 控制面。

分数高不代表 Terraform 管得越多。能清楚把高频/controller 资源移出 Terraform，同样是成熟。

---

## 17. 30 天吸收路线

### 第 1 周：State、身份与错误边界

精读：

- Slack Terraform；
- Grab elegant platform；
- Pinterest RPP；
- Terraform S3 backend 当前文档。

动手：

- 实验 1、2、4；
- 画出你自己的 `path → state → role → account → owner`；
- 故意制造 wrong backend 和并发锁；
- 做一次 state 版本恢复。

产出：

- workspace catalog v0；
- backend validation；
- state recovery runbook。

### 第 2 周：Module 与自助平台

精读：

- Deliveroo；
- Figma internal apps；
- Indeed；
- Canva。

动手：

- 实验 6、7；
- 把一个 resource wrapper 重构成 capability module；
- 给 module 加 owner、version、consumer inventory；
- 明确 UI/API/Git escape hatch。

产出：

- 一个 opinionated service module；
- module API 与弃用标准；
- 平台自助边界图。

### 第 3 周：安全、验证与治理

精读：

- DoorDash Conftest；
- Slack DNSSEC；
- Airbnb DPP；
- Cloudflare shift-left；
- Monzo security。

动手：

- 实验 5、8、9；
- policy 分级与 waiver expiry；
- apply 后外部 probe；
- drift 自动工单模型。

产出：

- policy test suite；
- risk tier；
- post-apply evidence schema；
- drift SLA。

### 第 4 周：迁移与工具边界

精读：

- Slack upgrade；
- Datadog v4；
- Spotify declarative infrastructure；
- Figma Kubernetes；
- Nubank immutable infrastructure。

动手：

- 实验 10、11、12；
- provider upgrade canary；
- 列出所有双重 owner；
- 控制面故障恢复演练。

产出：

- provider/CLI support matrix；
- Terraform vs controller ownership table；
- 下一季度平台路线图。

---

## 18. 把博客经验用于设计评审的 20 个问题

1. 这份 root 的 owner 是谁，on-call 是谁？
2. 为什么这些资源属于同一 state？
3. 如果整份 state 被误删，影响是否同一故障域？
4. repo/path 如何被证明有权使用这个 backend？
5. plan/apply 如何绑定同一 commit、变量和 provider？
6. CI 用什么身份，凭据有效多久？
7. runner 执行不受信代码前还是后取得生产 role？
8. module 改动的所有消费者怎样被发现？
9. module/provider 升级如何 canary 和回滚？
10. 资源 replace 是否会丢数据或中断服务？
11. apply 后用什么数据面 probe 证明系统正常？
12. 控制台是否有 API 路径看不到的警告？
13. break-glass 修改怎样回写 Git/state？
14. Terraform code/state 与 AWS actual 怎样对账？
15. owner、成本和数据分类怎样保持当前？
16. 哪些字段被其他 controller 修改？
17. 这个资源的变化频率是否与 Terraform pipeline 匹配？
18. LocalStack 能验证什么，哪些必须去真实 AWS？
19. 如果平台 UI/runner 挂了，怎样恢复？
20. 如果今天重选 owner，这个资源还应该归 Terraform 吗？

---

## 19. 当前版本校正

### 19.1 S3 锁

历史博客中的 DynamoDB locking 已不再是新项目首选。当前官方文档：

- S3 backend 支持 `use_lockfile = true`；
- DynamoDB-based locking 已 deprecated；
- 推荐 bucket versioning；
- `.tflock` 需要单独的 Get/Put/Delete 权限。

来源：[Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)。

### 19.2 CLI workspace

CLI workspace 共享同一 backend，不适合强权限隔离或系统分解。来源：[Manage workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)。

### 19.3 Provider lock

`.terraform.lock.hcl` 应提交版本控制，以锁定 provider selection。它不自动解决 module 版本与所有消费者追踪。来源：[Provider requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)。

### 19.4 LocalStack

`tflocal` 通过 Terraform override 注入 AWS service endpoints，可让未修改的 AWS Terraform root 指向 LocalStack；复杂 CI 可结合 init hooks/Testcontainers。它是兼容层和测试环境，不是 AWS 行为的形式证明。

### 19.5 旧 HCL 与旧资源

Deliveroo、Slack、Segment 等历史代码中的插值语法、resource 名称、provider 版本和参数不可直接复制。应吸收架构和失败模型，再查当前 registry/docs。

---

## 20. 综合结论：真正值得偷师的不是 Terraform 代码

从这些公司博客里反复出现的成熟形态是：

```text
Terraform
  不是平台本身
  而是平台的一台执行引擎

平台
  = 身份 + 目录 + 状态 + 政策 + 测试
  + 审批 + 执行 + 探针 + 漂移 + 恢复
  + 所有权 + 成本 + 迁移机制
```

Slack 教你拆 state 和做升级；Pinterest 教你保护 path/state/role 映射；Grab 教你 Raw Terraform 自助为什么会失败；Deliveroo 与 Figma 教你把 module 做成业务能力；Indeed 教你组织采用；Airbnb 教你把 IaC 接进治理；DoorDash 教你自动化审批；Slack DNSSEC 教你不迷信 apply；Spotify、Figma 和 Nubank则提醒你：**成熟的 Terraform 平台最重要的能力之一，是知道什么时候不再让 Terraform 拥有那个资源。**

---

## 21. 来源索引

### 21.1 独立公司工程博客

- Slack — [How We Use Terraform At Slack](https://slack.engineering/how-we-use-terraform-at-slack/)
- Slack — [The Case of the Recursive Resolvers](https://slack.engineering/what-happened-during-slacks-dnssec-rollout/)
- Pinterest — [Securing Infrastructure at Scale: RPP](https://medium.com/pinterest-engineering/securing-infrastructure-at-scale-introducing-pinterests-resource-provisioner-pipeline-rpp-8283bb12cbe5)
- Grab — [An elegant platform](https://engineering.grab.com/an-elegant-platform)
- Grab — [The Journey of Deploying Apache Airflow at Grab](https://engineering.grab.com/the-journey-of-deploying-apache-airflow-at-Grab)
- Deliveroo — [Application Deployment at Deliveroo](https://deliveroo.engineering/2018/02/21/application-deployment.html)
- Segment — [Automating Our Infrastructure to Empower Engineers](https://segment.com/blog/automating-our-infrastructure/)
- Segment — [Secure access to 100 AWS accounts](https://segment.com/blog/secure-access-to-100-aws-accounts/)
- Segment — [Revamping Segment’s Flink real-time compute platform](https://segment.com/blog/revamping-segments-flink-real-time-compute-platform/)
- Airbnb — [Automating data protection at scale — Part 1](https://medium.com/airbnb-engineering/automating-data-protection-at-scale-part-1-c74909328e08)
- Figma — [Securing internal web apps](https://www.figma.com/blog/inside-figma-securing-internal-web-apps/)
- Figma — [How we migrated onto K8s in less than 12 months](https://www.figma.com/blog/migrating-onto-kubernetes/)
- Spotify — [The Path to Declarative Infrastructure](https://engineering.atspotify.com/2023/05/fleet-management-at-spotify-part-2-the-path-to-declarative-infrastructure/)
- Spotify — [Fleet-wide Refactoring](https://engineering.atspotify.com/2023/5/fleet-management-at-spotify-part-3-fleet-wide-refactoring)
- Canva — [Infrastructure is Distribution](https://www.canva.dev/blog/engineering/infrastructure-is-distribution/)
- Monzo — [How we secure Monzo’s banking platform](https://monzo.com/blog/2022/03/31/how-we-secure-monzos-banking-platform)
- Nubank — [21 thousand databases with a lean team](https://building.nubank.com/how-nubank-distributes-infrastructure-ownership-to-operate-more-than-21-thousand-databases-with-a-lean-team/)

### 21.2 公司工程师演讲 / 联合案例

- Indeed / HashiCorp — [How Indeed used Terraform in its move to AWS](https://www.hashicorp.com/en/resources/how-indeed-used-terraform-in-its-move-to-aws)
- Segment / HashiCorp — [Terraform abstractions for safety and power at Segment](https://www.hashicorp.com/resources/terraform-abstractions-safety-power/)
- DoorDash / HashiCorp — [Terraform Code Reviews: Supercharged with Conftest](https://www.hashicorp.com/resources/terraform-code-reviews-supercharged-with-conftest)
- Pinterest / AWS — [From Monolith to Multi-Account](https://aws.amazon.com/blogs/mt/from-monolith-to-multi-account-pinterests-aws-organization-transformation-journey/)
- Slack / AWS — [How Slack adopted Karpenter](https://aws.amazon.com/blogs/containers/how-slack-adopted-karpenter-to-increase-operational-and-cost-efficiency/)
- Nubank / AWS — [Immutable infrastructure with CloudFormation](https://aws.amazon.com/blogs/mt/leveraging-immutable-infrastructure-nubank/)

### 21.3 厂商自己的生产/Provider 经验

- Cloudflare — [Shifting left at enterprise scale](https://blog.cloudflare.com/shift-left-enterprise-scale/)
- Cloudflare — [Automatically generating the Terraform provider](https://blog.cloudflare.com/automatically-generating-cloudflares-terraform-provider/)
- Datadog — [Terraform provider v4.0.0](https://www.datadoghq.com/blog/datadog-terraform-provider-v4/)
- Datadog — [Managing Datadog with Terraform](https://www.datadoghq.com/blog/managing-datadog-with-terraform/)
- GitLab — [A multi-region service to deliver AI features](https://about.gitlab.com/blog/building-gitlab-with-gitlab-a-multi-region-service-to-deliver-ai-features/)

### 21.4 当前事实校正

- HashiCorp — [S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)
- HashiCorp — [CLI workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)
- HashiCorp — [Provider requirements and lock file](https://developer.hashicorp.com/terraform/language/providers/requirements)
- LocalStack — [Terraform integration](https://docs.localstack.cloud/aws/connecting/infrastructure-as-code/terraform/)
- LocalStack — [Terraform init hooks + Testcontainers](https://docs.localstack.cloud/aws/tutorials/using-terraform-with-testcontainers-and-localstack/)
