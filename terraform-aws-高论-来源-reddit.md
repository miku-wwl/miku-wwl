# Terraform × AWS 社区高论（来源：Reddit）

整理日期：2026-07-27  
主要来源：`r/Terraform`、`r/devops`  
主题范围：Terraform/OpenTofu 在 AWS 上的生产结构、state、多账号、CI/CD、漂移、安全、模块、LocalStack、EKS、Lambda、升级与成本治理。

> 这不是 Reddit 热帖搬运，也不是“点赞数排行榜”。它是一份经过筛选、归并和事实校正的工程笔记。  
> Reddit 搜索和索引不可能保证穷尽所有帖子；本文追求的是覆盖主要决策面和反复出现的生产经验，而不是伪称“全网无遗漏”。

---

## 如何阅读这份报告

本文使用三个证据标签：

- **[共识]**：在多个独立讨论中反复出现，且没有明显依赖某个团队的特殊环境。
- **[争议]**：有经验的使用者给出了相反结论，答案取决于规模、团队和约束。
- **[个案]**：来自真实经历，值得研究，但不能直接推广成普遍最佳实践。

筛选时优先保留：

- 真实生产事故、迁移和恢复复盘；
- 多账号、大规模或长期维护经验；
- 清楚说明代价和失败条件的意见；
- 能转化为验证实验或工程检查项的内容。

主动过滤：

- 证书、求职和纯入门路线；
- 没有技术细节的工具站队；
- 供应商软广和“我刚做了一个工具”的单方陈述；
- 只展示 `terraform apply` 成功、没有维护和失败场景的 Demo；
- 单纯高赞但已过时的事实。

---

## 0. 先吸收这 18 条

1. **Terraform 的核心难题不是写资源，而是确定谁拥有什么、由哪个 state 改、谁有权 apply。**
2. **按 blast radius、权限边界、变更频率和团队所有权切 state；不要按目录美观切。**
3. **一个 state 不应跨越大量 AWS account、region 和独立业务故障域。**
4. **Control Tower 与 Terraform 不是二选一：前者提供 landing zone/guardrail，后者管理可审查的组织配置与工作负载。**
5. **state、saved plan 和 plan JSON 都按敏感资产处理；`sensitive = true` 不是加密。**
6. **CI 到 AWS 优先 OIDC + 短期角色，不要长期 Access Key。**
7. **PR plan 使用只读/受限角色；merge 后 apply 使用独立写角色。**
8. **把 `.terraform.lock.hcl` 提交进 Git；provider 升级应形成普通 PR，而不是某天突然 `init -upgrade`。**
9. **root module 控制 provider 版本；共享 child module 只声明合理兼容范围。**
10. **公开模块不是天然安全或危险；读代码、锁版本、测试升级，必要时用薄 wrapper 固化组织规则。**
11. **只把参数转发一遍的 wrapper 是抽象税；能收窄接口、固化安全默认值、封装完整能力才有价值。**
12. **Terragrunt 能解决重复 backend/provider 和多 stack 编排，也会引入第二套语言、依赖图和调试成本。先量化问题再引入。**
13. **drift detection 应先通知和分流，不要默认自动 apply 修复生产。**
14. **brownfield 不是“全部 import”或“全部重建”：按资源风险、真实漂移和可切流能力逐块决定。**
15. **LocalStack 适合快速反馈、模块契约、幂等性和多服务集成；真实 AWS sandbox 负责验证 IAM、SCP、配额、最终一致性和托管服务行为。**
16. **Terraform 创建 EKS 和 GitOps 引导层即可；集群内不断变化的应用通常交给 Argo CD/Flux/Helm 生命周期。**
17. **Lambda 应 build once、按 digest/hash 推广；不要让 Terraform 每次 plan 都临时打一个不同的 zip。**
18. **成本、策略、安全和测试都应进入 PR，而不是等 apply 之后由人去控制台发现。**

---

## 1. Reddit 真正有价值的，不是目录模板

生产结构讨论里，新手最常问：

- mono-repo 还是 multi-repo；
- `dev/stage/prod` 怎么摆；
- module 放哪里；
- workspace 还是目录；
- Terraform 还是 Terragrunt。

但有经验的回复不断把问题拉回四个更根本的变量：

| 变量 | 必须回答的问题 |
|---|---|
| blast radius | 一次错误 apply 最多能伤到什么？ |
| ownership | 哪个团队审批、部署、值班和恢复？ |
| credentials | pipeline 到底能假扮哪个角色？ |
| change cadence | 网络、数据库、应用服务是否应以同样频率发布？ |

**[共识]** 目录只是这些边界的投影。没有先回答边界问题，再漂亮的 repo tree 也只是把风险排版整齐。

参考：

- [How do you structure Terraform projects for AWS in production? — r/Terraform](https://www.reddit.com/r/Terraform/comments/1s4uh0j/how_do_you_structure_terraform_projects_for_aws/)
- [Terraform for Devops (Real world experience) — r/devops](https://www.reddit.com/r/devops/comments/u9oqp5/terraform_for_devops_real_world_experience/)
- [Should you use the same IaC code for dev/staging/prod or copy it? — r/devops](https://www.reddit.com/r/devops/comments/1bk1nts/)

---

## 2. State 边界：按失败域切，不要按资源种类机械切

### 高论

**[共识]** 巨型单 state 的主要问题不是文件大，而是：

- 每个 plan 都要 refresh 大量无关资源；
- 锁冲突把不相干团队串行化；
- 某个 provider/API 故障阻塞整张 graph；
- 权限只能按 state 的最大能力授予；
- 一次误操作有机会影响所有环境；
- 恢复、导入和升级都变成“大爆炸”。

反过来，切得过碎也会造成：

- 大量 `terraform_remote_state` 或外部参数传递；
- 依赖顺序隐藏到 CI 脚本或 Terragrunt graph；
- 每个小改动要触发很多 pipeline；
- 跨 state 事务不存在，出现中间态；
- 工程师难以知道真正的部署入口。

### 推荐切分维度

优先级通常是：

1. AWS account；
2. 环境或业务故障域；
3. region；
4. 生命周期明显不同的能力；
5. 团队所有权；
6. 变更频率。

一个可用的起点：

```text
organization/
  org-baseline/
  identity-center/
  logging-security/

accounts/
  shared-services/
    network/
    cicd/
    observability/
  product-a-dev/
    foundation/
    data/
    services/
  product-a-prod/
    foundation/
    data/
    services/
```

这不是必须照抄的目录，而是在表达：

- 组织基线不跟应用一起发布；
- prod 不跟 dev 共用 state；
- 网络和数据库不应被每次应用发布顺带 refresh；
- pipeline role 可以按入口收窄权限。

### 一个实用判定

如果两个资源满足以下大多数条件，它们更适合留在同一 state：

- 必须原子地一起创建或销毁；
- 总由同一团队维护；
- 使用同一组部署凭证；
- 变更频率相近；
- 出错时一起恢复；
- 没有独立复用价值。

如果相反，就应考虑拆开。

### 不要把 state 拆分当作微服务宗教

**[共识]** “每个资源一个 state”通常只是把 Terraform graph 的复杂性搬到编排层。  
**[个案]** 有团队用几十或几百个小 stack 成功运行，但前提是已经投资于自动发现、依赖图、并发控制、统一日志和批量升级。

---

## 3. S3 backend：Reddit 旧帖需要时间校正

大量经典帖子把标准答案写成：

```text
S3 state + DynamoDB locking
```

这在帖子发表时是合理经验，但今天不能不加日期直接复读。

### 当前应理解为

- S3 保存 state；
- S3 bucket 开启 versioning；
- backend 开启 S3 lockfile；
- bucket、object 和 KMS/IAM 权限严格收窄；
- DynamoDB 锁是兼容旧版本的迁移路径，不再是新设计的默认答案。

示意：

```hcl
terraform {
  backend "s3" {
    bucket       = "example-tfstate"
    key          = "product-a/prod/network.tfstate"
    region       = "ap-southeast-2"
    use_lockfile = true
  }
}
```

事实校正来源：

- [HashiCorp S3 backend：S3 lockfile 与已弃用的 DynamoDB locking](https://developer.hashicorp.com/terraform/language/backend/s3)

### Backend bootstrap 的鸡生蛋

**[争议]** 社区常见四种做法：

1. 手工或平台脚本只创建一次 state bucket/KMS；
2. 用 CloudFormation/bootstrap 工具创建；
3. 用一个本地 state 的 Terraform root 创建，之后很少再动；
4. 用 Terragrunt 等工具自动生成/初始化 backend。

没有哪个选择能消灭 bootstrap，只能决定把它放在哪里。

### 推荐原则

- 明确标记 bootstrap 是一个特殊生命周期；
- 代码和恢复说明进入版本控制；
- state bucket 禁止随普通 destroy 删除；
- bucket versioning、public access block、加密和审计日志先于业务 state；
- 定期演练从某个旧 object version 恢复；
- 不要为了“100% Terraform”让后端自我删除风险变高。

参考：

- [How are you managing remote-state resources? — r/Terraform](https://www.reddit.com/r/Terraform/comments/13o1rxr/)
- [Terraform state in S3 — r/devops](https://www.reddit.com/r/devops/comments/fwwwib/)

---

## 4. Workspaces：不是错，但不能替代安全边界

### 支持方

**[个案]** workspaces 对“同一套配置、少量环境、完全相同生命周期”非常轻：

- 少量重复目录；
- backend 自动按 workspace 分 key；
- 开发者理解成本低；
- 临时环境方便。

### 反对方

**[共识]** 对强隔离的 AWS 生产环境，workspace 名称本身不会提供：

- 独立 AWS account；
- 独立 IAM role；
- 独立审批规则；
- 独立变量来源；
- 独立故障域；
- 独立代码版本。

如果所有条件分支都写成：

```hcl
local.is_prod ? ... : ...
```

那么“复用”最终可能变成一棵难以推理的条件树。

### 实用结论

- 临时、同构环境：workspace 可用；
- prod/dev 有不同账号、审批和安全策略：独立 root/state 更清楚；
- 不要因为害怕少量重复，就引入大量条件；
- 环境差异应主要表现为数据和规模，不应改变整套拓扑语义。

参考：

- [Workspaces, Terragrunt or something else? — r/devops](https://www.reddit.com/r/devops/comments/1ruxklc/workspaces_terragrunt_or_something_else/)
- [Same IaC code vs copy-paste — r/devops](https://www.reddit.com/r/devops/comments/1bk1nts/)

---

## 5. AWS 多账号：Control Tower 与 Terraform 是上下层

### 反复出现的生产模式

**[共识]**

```text
AWS Organizations / Control Tower
              ↓
OU、guardrail、日志与安全账号
              ↓
共享服务账号中的 CI runner / 平台
              ↓ AssumeRole
每个工作负载账号中的受限部署角色
              ↓
按 account/env/service 切分的 Terraform state
```

Control Tower 擅长：

- landing zone；
- 账号工厂；
- 基础 guardrail；
- 日志和安全账号；
- 组织级基线。

Terraform 擅长：

- OU/SCP/Identity Center 配置的可审查管理；
- 每个账号的 baseline；
- 网络、数据和应用资源；
- 模块化组织规则；
- PR/plan/apply 审计。

### “先手工点几下”并不等于失败

**[个案]** 有生产团队选择手工初始化 Control Tower，因为这是少量、低频而且由 AWS 管理的 bootstrap；后续 SCP、账号、OU、SSO/IAM 和工作负载再由 Terraform 接管。

高论不是“手工最好”，而是：

> 自动化的目标是降低风险和重复劳动，不是让架构图上没有任何手工步骤。

### 多账号要避免的设计

- 一个超级 role 能管理所有账号所有服务；
- 所有账号共用一个巨型 state；
- 本地管理员长期保存跨账号 Access Key；
- provider alias 数量无限增长；
- 一个 plan 同时改 organization、网络和应用；
- 使用 account 名称推断权限，却不设置允许的 account ID；
- shared-services runner 可以无条件进入 prod。

参考：

- [AWS Control Tower with Terraform — r/devops](https://www.reddit.com/r/devops/comments/1466a5f/)
- [AWS multi environments — r/devops](https://www.reddit.com/r/devops/comments/rjjd8b/)
- [Organizations IaC automation — r/devops](https://www.reddit.com/r/devops/comments/r8qkch/)
- [Managing tens of AWS accounts — r/devops](https://www.reddit.com/r/devops/comments/163g1lw/)

---

## 6. CI/CD：plan 是审查对象，apply 是受控能力

### 推荐流水线

```text
Pull request
  ├─ fmt / validate
  ├─ lint
  ├─ security & policy checks
  ├─ module/unit tests
  ├─ speculative plan with read-only role
  ├─ cost delta
  └─ human review

Merge / approved deployment
  ├─ verify exact commit, tool and inputs
  ├─ acquire state lock
  ├─ generate/re-check final plan
  ├─ policy & approval gate
  ├─ apply with restricted write role
  └─ publish audit result
```

### PR plan 与最终 apply 之间不是天然一致

**[共识]** PR 打开之后，真实 AWS、state、provider、变量或其他 PR 都可能变化。因此：

- PR plan 是审查证据，不一定是最终可执行 plan；
- apply 前必须重新确认最终变更；
- 如果使用 saved plan，必须绑定同一 commit、backend、变量和 provider；
- saved plan 本身含完整配置和可能的明文敏感值，应短期保存并限制下载；
- 不要把 plan binary 或 plan JSON 提交到 Git。

### 最小角色分离

| 阶段 | AWS 身份 | 能力 |
|---|---|---|
| PR | plan role | 读取目标资源和 state；不能广泛写生产 |
| apply | deploy role | 只允许该 stack 需要的写操作 |
| break-glass | emergency role | 强审计、短时授权、事后回收 |

**[个案]** Atlantis 被多个团队用于 PR comment 驱动的 plan/apply 和 locking。  
**[争议]** 另一些团队认为 GitHub Actions/GitLab CI 已够用，Atlantis 增加运行和队列限制。工具不是重点，关键是同一套不变量：锁、审查、身份分离、不可篡改输入和日志。

参考：

- [PSA: Love Terraform and CI/CD? You want Atlantis — r/devops](https://www.reddit.com/r/devops/comments/cakyfp/)
- [End-to-End CI/CD Setup Using Jenkins + Terraform — r/Terraform](https://www.reddit.com/r/Terraform/comments/1sz83yx/)
- [HashiCorp：saved plan 的自动化用途与敏感数据警告](https://developer.hashicorp.com/terraform/cli/commands/plan)

---

## 7. OIDC：不要让 GitHub Secret 里躺着永久 AWS 密钥

### 社区共识

**[共识]** GitHub Actions/GitLab 等 CI 访问 AWS 时，首选 OIDC federation：

- 每次运行获得短期凭证；
- 不需要轮换长期 Access Key；
- 可以按 repo、branch、tag、environment 限制 trust；
- CloudTrail 中角色会话更容易追踪；
- PR plan 和 prod apply 可使用不同角色。

### OIDC 也能配错

“使用 OIDC”并不自动等于安全。必须检查：

- trust policy 的 `sub` 是否限制到指定 org/repo/branch/environment；
- 是否允许任意 fork 或 pull request 获取高权限；
- role session duration；
- permissions policy 是否远超 stack 需要；
- GitHub environment 是否有人工审批；
- workflow 文件本身的修改是否受 CODEOWNERS 保护；
- 第三方 Action 是否锁定到可信 commit。

### 推荐结构

```text
repo:infra
  pull_request → tf-plan-prod-readonly
  main merge   → tf-apply-nonprod
  protected environment approval
               → tf-apply-prod
```

参考：

- [GitHub Actions AWS credentials — r/devops](https://www.reddit.com/r/devops/comments/1jrvxgc/)
- [GitHub OIDC security — r/devops](https://www.reddit.com/r/devops/comments/xecjrn/)
- [OIDC use cases: read-only PR, write on main — r/devops](https://www.reddit.com/r/devops/comments/1iir0gv/)
- [Stop putting AWS credentials in GitHub secrets — r/devops](https://www.reddit.com/r/devops/comments/s79vn1/)
- [AWS：GitHub OIDC trust policy 必须约束 subject](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html)

---

## 8. State 和 plan 就是秘密数据库

### 一个经常被误解的点

```hcl
variable "password" {
  sensitive = true
}
```

只会避免部分 CLI/UI 显示，不会让值从 state 或 plan 中消失。

**[共识]** 只要 provider/resource schema 把值保存进 state，它通常仍能被 state 读取者看到。

### 实际控制

- state bucket 独立、private、versioned、encrypted；
- backend role 与业务 deploy role 权限分离；
- 只允许对应 stack 访问对应 object prefix；
- 禁止普通开发者直接下载 prod state；
- CloudTrail/S3 data event 或相应审计覆盖敏感访问；
- plan artifact 设短 TTL；
- CI 日志避免打印 `terraform show -json` 全量内容；
- 尽量让应用运行时从 Secrets Manager/SSM/Vault 获取秘密；
- 优先 IAM role、IRSA/Pod Identity、数据库 IAM auth 等无静态密码模式；
- 把 state 泄露纳入威胁模型和演练。

### “CI 注入 secret”不等于“不进 state”

**[共识]** 不论秘密来自：

- `TF_VAR_*`；
- GitHub Secret；
- Vault data source；
- Secrets Manager data source；
- 本地环境变量；

如果它最终作为资源属性进入 provider state，秘密仍可能出现在 state 中。秘密来源和 state 持久化是两个问题。

参考：

- [Terraform state sensitive info — r/Terraform](https://www.reddit.com/r/Terraform/comments/1gxxhna/)
- [IaC and secrets — r/devops](https://www.reddit.com/r/devops/comments/nhuhdx/)
- [Manually created secrets and Terraform state — r/devops](https://www.reddit.com/r/devops/comments/1e31fld/)
- [External values: SSM/Vault vs CI TF_VAR — r/devops](https://www.reddit.com/r/devops/comments/1tpnxfq/)
- [HashiCorp：sensitive 值仍保存在 state 和 plan](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)

---

## 9. Provider 升级：冻结不动和永远 latest 都会出事

### Reddit 的事故味道

社区里同时存在两类痛苦：

- 没锁版本，AWS provider 大版本发布后 pipeline 突然变化；
- 锁死多年，需要新功能时一次跨多个 major，技术债集中爆炸。

**[共识]** 更稳的做法是“受控持续升级”。

### 推荐策略

1. root module 声明清晰的 Terraform/provider 兼容范围；
2. 提交 `.terraform.lock.hcl`；
3. CI 使用固定 Terraform CLI 版本，不使用模糊的 `latest`；
4. Renovate/Dependabot 定期发升级 PR；
5. PR 展示 lockfile diff、release note 和 plan；
6. 先在 dev/test/sandbox apply；
7. 对 provider major 阅读官方 upgrade guide；
8. 分批推广，避免 Terraform Core、AWS provider、模块大版本同一 PR 全升级；
9. 维护回滚路径和 state version；
10. 定期升级，避免一次跨越几年。

### Root 与 child module 的约束不同

共享模块通常不应精确钉死 provider patch，因为多个模块的精确约束可能互相冲突。常见模式：

```hcl
# child module：声明已知兼容下界/范围
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.80, < 7.0"
    }
  }
}
```

```hcl
# root：控制部署实际允许范围，lockfile 记录精确选择
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

### 一个 2026 年的鲜活教训

**[个案]** Terraform 1.15 发布讨论中，有用户报告 `validate` 行为变化打破现有 CI；临时处理是将 reusable workflow 固定回已知版本，而不是跟随 latest。这类事件说明：

> `required_version` 是兼容声明，不是 CI 实际下载版本的锁。

参考：

- [How do you keep provider versions up to date? — r/Terraform](https://www.reddit.com/r/Terraform/comments/10wr2l1/)
- [Provider upgrade workflow in CI — r/Terraform](https://www.reddit.com/r/Terraform/comments/1ct809z/)
- [Terraform/provider upgrade questions — r/Terraform](https://www.reddit.com/r/Terraform/comments/tzv4b3/)
- [AWS Provider 5.0 heads-up — r/Terraform](https://www.reddit.com/r/Terraform/comments/13ru7f9/)
- [Terraform 1.15 release discussion and CI regression — r/devops](https://www.reddit.com/r/devops/comments/1szc32o/)
- [HashiCorp dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)

---

## 10. 模块：公共、私有和 wrapper 之间没有宗教答案

### 公共模块支持方

**[个案]**

- 维护者和使用者更多；
- 常见 AWS 边缘条件已经踩过；
- 锁定版本后可控；
- 可以阅读源码并向 upstream 贡献；
- VPC、EKS 等复杂服务无需重复造完整轮子。

### 自建模块支持方

**[个案]**

- 大型公共模块为覆盖所有用户暴露几百个变量；
- 默认值可能不符合组织安全基线；
- 调试高度通用的条件逻辑比读几个直接资源更难；
- major upgrade 由外部维护节奏驱动；
- 团队实际只需要小部分能力。

### 真正的判断标准

模块应该提供一个**有观点的能力**，而不是把 provider resource 的每个参数换个名字再暴露一遍。

有价值的组织模块例如：

```text
secure_s3_data_bucket
  - 强制加密
  - public access block
  - versioning
  - 生命周期规则
  - 审计标签
  - 可选复制
  - 只暴露组织允许改变的参数
```

价值较低的 wrapper：

```text
aws_s3_bucket 的 45 个参数
→ 原样变成 45 个 module variables
→ 没有安全默认值、策略或组合能力
```

### 推荐决策流程

1. 读公共模块代码、issues、release cadence；
2. 评估暴露面和组织默认值；
3. 用最小真实场景做 create/no-op/update/destroy；
4. 锁定 module version；
5. 如果只需 10% 能力且公共模块复杂度很高，考虑自建；
6. 如果公共模块能力成熟，用薄 wrapper 固化组织规则；
7. 不要复制代码后永不跟踪上游安全修复；
8. 给模块写升级和弃用策略。

### 重要事实：lockfile 不锁远程 module

`.terraform.lock.hcl` 当前主要锁 provider，不会替你固定远程 module。生产调用应明确 module version/tag/commit。

参考：

- [Does your organization use provided Terraform modules? — r/devops](https://www.reddit.com/r/devops/comments/ytg7o1/)
- [Public Terraform modules in production — r/Terraform](https://www.reddit.com/r/Terraform/comments/bv50ez/)
- [Terraform modules everywhere? — r/devops](https://www.reddit.com/r/devops/comments/11hxk59/)
- [Using open source Terraform modules vs writing your own — r/Terraform](https://www.reddit.com/r/Terraform/comments/1n7jlax/)

---

## 11. Terragrunt：它解决真实问题，也会成为真实问题

### 它确实能解决什么

- backend/provider 配置重复；
- account/region/environment 矩阵；
- 多 root module 的依赖顺序；
- 批量 plan/apply；
- 统一输入和组织约定；
- 用目录结构表达环境实例。

### 它会增加什么

- Terraform HCL 之外的第二层语义；
- `run-all` 的依赖图和锁；
- 更复杂的错误堆栈；
- 本地与 CI 行为差异；
- onboarding 成本；
- 升级 Terragrunt 自身的风险；
- 全量 plan 时间可能大幅增加；
- 容易把 child module 直接当 deployable root 使用；
- 团队可能为了 DRY 生成难以追踪的配置。

### Reddit 中最有价值的冲突

**[争议]**

- 一方在账号和环境增长后，从 vanilla Terraform 迁到 Terragrunt，认为重复和编排明显改善；
- 另一方正在迁走，因为 `run-all plan` 约 35 分钟，而拆回 vanilla 后约 7 分钟；
- 另有长期使用者用 Terragrunt 管理约 45 个 AWS accounts，认为问题多来自错误使用方式，而不是工具本身。

这不是谁更懂，而是工作负载图、CI 实现和团队心智模型不同。

### 引入前必须有指标

- 当前重复代码量；
- stack 数量；
- account × region × env 组合数；
- PR 到 plan 的 P50/P95 时间；
- 同时修改的 stack 数；
- 失败后定位时间；
- 新成员独立部署所需时间；
- 不使用 Terragrunt 时编排代码有多少。

没有指标时，Terragrunt 讨论很容易退化为审美战争。

参考：

- [Terragrunt: What It Solves, What It Costs — r/Terraform](https://www.reddit.com/r/Terraform/comments/1rhmfjl/)
- [Am I the only one who doesn't like Terragrunt? — r/Terraform](https://www.reddit.com/r/Terraform/comments/1ow9i1r/)
- [Terraform/Terragrunt/Terratest — r/devops](https://www.reddit.com/r/devops/comments/1ppqsxv/)
- [Use cases for Terragrunt — r/Terraform](https://www.reddit.com/r/Terraform/comments/yosx1n/)
- [Terragrunt basics and dependency outputs — r/Terraform](https://www.reddit.com/r/Terraform/comments/svlexs/)

---

## 12. Drift：检测、分流、恢复，不要盲目自愈

### 漂移的来源不只是不守规矩

- 事故期间手工修复；
- 自动扩缩或服务控制面改变属性；
- AWS 默认值/provider schema 演化；
- 另一个 Terraform state 管了同一资源；
- console 权限过宽；
- import 不完整；
- CI 失败在部分 apply；
- 外部系统管理同一对象；
- 生命周期忽略规则掩盖了真实差异。

### 推荐机制

定时运行：

```powershell
terraform plan -detailed-exitcode
```

处理约定：

- `0`：无变化；
- `1`：执行错误，单独告警；
- `2`：有 diff，创建 ticket/通知并分类。

### 为什么不默认自动 apply

**[共识]** 生产 drift 可能是：

- 正在救火的合理临时修改；
- Terraform code 错了；
- provider bug；
- AWS service 自动调整；
- 尚未完成的迁移。

自动 apply 会在不了解意图时把现实强推回代码，可能撤销救火措施或扩大故障。

### 建议分流

| 漂移类型 | 处理 |
|---|---|
| 未授权 console 改动 | 调查、回滚或回写代码，并收紧权限 |
| 应急修改 | 创建后续 PR，把现实纳入声明或撤销临时改动 |
| provider/default 噪音 | 升级、显式配置或合理 `ignore_changes` |
| 资源被外部工具拥有 | 明确单一 owner，移出其中一个 state |
| 丢失 state 绑定 | import |
| 架构已偏离代码 | 进入 brownfield 迁移计划 |

参考：

- [Dealing with Terraform Drift — r/devops](https://www.reddit.com/r/devops/comments/1lh1ufl/)
- [Terraform drift — r/devops](https://www.reddit.com/r/devops/comments/q7ecej/)
- [Terraform drift detection — r/devops](https://www.reddit.com/r/devops/comments/10tmm83/)

---

## 13. Brownfield：Import 还是重建，先看可控切换能力

### 社区里最成熟的答案

**[共识]** 不要把已有 AWS 环境当成一次性“全量 Terraform 化”项目。先盘点，再按资源组迁移。

### 更适合 import

- 资源承载不可轻易迁移的数据；
- 当前配置与目标模型接近；
- 依赖清楚；
- API 能完整读取关键属性；
- resource address 能稳定设计；
- 删除/重建代价高；
- 能在维护窗口校验 no-op。

### 更适合重建并切流

- 多年 console 修改，配置极不一致；
- 原有设计本身需要重构；
- import 后会产生巨大且难解释的 diff；
- 资源支持蓝绿或 DNS/ALB 切换；
- 新旧环境可并行验证；
- 重建比长期携带历史包袱风险更低。

### 推荐迁移顺序

1. 资产和 owner 盘点；
2. 标记有状态/无状态和删除风险；
3. 冻结无计划的 console 改动；
4. 建立目标 root/state/address 设计；
5. 用 import block 批量、可审查地导入；
6. 第一个目标是 no-op plan，不是马上重构；
7. 再逐步重命名，使用 `moved` block；
8. 对可重建服务建立平行环境并切流；
9. 为每批资源定义回滚；
10. 加 drift 检查，避免迁移期间继续分叉。

### 重要警告

- 不要在活跃 pipeline 同时运行时手工编辑 state；
- 不要看到 diff 就 `ignore_changes`；
- 不要为了“代码看起来干净”重建关键数据库；
- 不要把 import 成功误认为配置已准确；
- 先备份和验证 state version，再做地址移动。

参考：

- [Best approach for existing AWS infra: import or rebuild? — r/Terraform](https://www.reddit.com/r/Terraform/comments/1net5yl/)
- [Two-year-old Terraform, production drifted — r/Terraform](https://www.reddit.com/r/Terraform/comments/16bq3kd/)
- [Converting an existing deployment to Terraform — r/devops](https://www.reddit.com/r/devops/comments/13wr3fp/)
- [Systematically Terraforming a brownfield — r/devops](https://www.reddit.com/r/devops/comments/1j3i93y/)

---

## 14. LocalStack：你的 Ultimate 版应该放在哪一层

### Reddit 的分裂，其实是职责差异

**[争议]**

- 应用开发团队：LocalStack 对本地多服务集成、快速反馈和 PR 测试很有价值；
- 平台/基础设施团队：最终仍偏好真实 AWS sandbox，因为模拟器不可能完全复制控制面。

这两者可以同时成立。

### 最适合 LocalStack Ultimate 的测试

1. Terraform 配置能完成 `init/plan/apply`；
2. module 输入输出契约；
3. create → second plan no-op → update → destroy；
4. S3/SQS/SNS/Lambda/DynamoDB 等多服务连接；
5. Lambda event source 和失败路径；
6. 测试数据与资源的快速隔离；
7. provider endpoint 注入是否只停留在 root；
8. CI 中的快速回归；
9. brownfield import/moved block 的无成本演练；
10. state 拆分和依赖输出的实验。

### 不应只靠 LocalStack 证明

- IAM/SCP 的完整授权语义；
- Organizations/Control Tower；
- AWS quota 和 account-level limit；
- 最终一致性与真实节流；
- EKS 控制面和 addon 兼容性；
- AMI、region 和可用区差异；
- 托管数据库升级；
- AWS 特定网络边缘行为；
- 真实费用；
- provider 与新 AWS API 的当天兼容性。

### 推荐测试金字塔

```text
             少量真实 AWS 生产前验证
          真实 AWS sandbox / ephemeral account
       LocalStack Ultimate 多服务集成与失败注入
    terraform test / module tests / policy / static scan
 fmt / validate / lint / provider schema / unit-level checks
```

LocalStack 不是“假 AWS，所以没用”，也不是“本地过了，所以生产等价”。它的价值是缩短反馈环，而不是提供最终真实性。

参考：

- [As DevOps, do you use LocalStack? — r/devops](https://www.reddit.com/r/devops/comments/1jpzxgk/)
- [Anyone tried testing Terraform against LocalStack? — r/Terraform](https://www.reddit.com/r/Terraform/comments/h057k0/)
- [DoorDash-style local AWS environment with LocalStack and Terraform — r/devops](https://www.reddit.com/r/devops/comments/11lahst/)
- [Lambda local testing: SAM, LocalStack or ephemeral AWS — r/devops](https://www.reddit.com/r/devops/comments/1jocutu/)
- [Secure, testable Terraform pipeline with Terratest, LocalStack, Checkov, Conftest and Nix — r/Terraform](https://www.reddit.com/r/Terraform/comments/1rg1jer/)

---

## 15. 一套适合 LocalStack Ultimate 的高强度实验

### 实验 1：模块幂等性

目标：

```text
apply → plan(no-op) → 改一个安全输入 → plan → apply → plan(no-op) → destroy
```

失败即记录：

- provider 默认值造成的永久 diff；
- unordered set/list；
- archive hash 不稳定；
- 时间戳或随机值未使用 keepers；
- data source 在 apply 前未知；
- `ignore_changes` 掩盖真实问题。

### 实验 2：跨服务事件链

创建：

```text
S3 event → SQS → Lambda → DynamoDB
```

验证：

- resource policy；
- dead-letter queue；
- retry；
- event source mapping；
- 加密；
- 删除顺序；
- 第二次 plan 是否 no-op。

真实 AWS 补测：

- IAM effective permissions；
- delivery latency；
- KMS key policy；
- service quota；
- CloudWatch 指标。

### 实验 3：State 边界

把网络和应用先放一个 state，再拆为：

```text
foundation state → SSM/output contract → service state
```

测量：

- plan 时间；
- lock 冲突；
- 输出耦合；
- foundation 变更如何触发 service；
- foundation 不可用时 service 能否 plan；
- 删除保护。

### 实验 4：Drift 注入

apply 后手动修改：

- tag；
- bucket policy；
- Lambda env；
- security group rule。

然后运行 scheduled plan，并实现：

- exit code 分类；
- diff 摘要；
- ticket/Markdown 报告；
- 人工选择“回滚现实”或“回写代码”。

### 实验 5：Brownfield

1. 先用 AWS CLI 或另一个 root 创建资源；
2. 为资源写目标配置；
3. 使用 import block；
4. 修到 no-op；
5. 用 moved block 改 address；
6. 验证 state backup 和恢复。

### 实验 6：Provider 升级

建立两条 CI：

```text
current lockfile
candidate provider PR
```

对同一套 fixture 执行：

- create；
- no-op；
- update；
- import；
- destroy；
- plan JSON 结构差异。

只比较 plan 成败不够，还要看 replacement 和默认值变化。

### 实验 7：OIDC 权限模型的替身测试

LocalStack 可用于验证 pipeline 逻辑和角色选择，但 trust policy 最终必须在真实 sandbox 测。

构造：

- PR role 只能读；
- nonprod role 可写指定前缀；
- prod role 需要 environment approval；
- 错误 repo/branch 的 token 必须失败。

### 实验 8：失败中的部分 apply

故意让事件链中间资源失败，观察：

- state 写入了什么；
- rerun 能否收敛；
- 哪些资源需要 import；
- provider timeout/retry；
- destroy 是否安全。

### 实验 9：Secrets 泄露审计

放入假的 canary secret，检查它是否出现在：

- `terraform.tfstate`；
- saved plan；
- `terraform show -json`；
- CI log；
- crash log；
- test artifact。

不要使用任何真实秘密。

### 实验 10：工具链可复现

锁定：

- Terraform/OpenTofu；
- AWS provider；
- tflint；
- checkov/tfsec；
- conftest；
- LocalStack image；
- Terratest dependencies。

在干净 runner 上验证结果一致，避免“我本机可以”。

---

## 16. EKS：Terraform 管基础，GitOps 管集群内高速变化

### Reddit 中较稳定的边界

**[共识]**

Terraform 适合：

- VPC/subnet；
- EKS control plane；
- node group；
- IAM/OIDC/Pod Identity；
- 安全组；
- KMS/logging；
- 引导 Argo CD 或 Flux；
- 少量必须先于 GitOps 的 addon。

GitOps/Helm 更适合：

- 业务 Deployment/Service；
- 高频发布；
- 大量 Kubernetes manifest；
- 持续 reconciliation；
- 应用团队自服务；
- 与镜像版本一起变更的配置。

### 为什么不建议 Terraform 管所有 Kubernetes workload

- Kubernetes 自己已有期望状态和 reconciliation；
- Terraform state 与 cluster state 容易出现双重所有权；
- Helm provider 错误常难定位；
- cluster 尚未创建时 provider 初始化有鸡生蛋；
- cluster 销毁后 refresh/plan 容易失败；
- 应用发布频率远高于基础设施。

### EKS 升级不是改一行版本号

社区生产经验反复提到：

1. 查 deprecated API/CRD；
2. 查 EKS addon、CNI、CoreDNS、kube-proxy、autoscaler 和 ingress 兼容性；
3. 先在低环境升级；
4. 一次处理一个 minor；
5. 升 control plane；
6. 测试；
7. 滚动升级 node group/AMI；
8. drain、容量和 PDB 都要验证；
9. 再验证 addon、日志、网络和 autoscaling；
10. 多集群按批次推进，不要同时升 50 个。

### 建议 state

```text
eks-foundation state
  └─ cluster, node groups, IAM, logging

gitops-bootstrap state
  └─ Argo/Flux bootstrap and repository registration

application repositories
  └─ Helm/Kustomize/manifests reconciled by GitOps
```

参考：

- [Upgrading EKS cluster programmatically — r/devops](https://www.reddit.com/r/devops/comments/1l7c9bh/)
- [How do you handle Kubernetes updates? — r/devops](https://www.reddit.com/r/devops/comments/1avc1as/)
- [After Terraform deploys EKS, how do you deploy services? — r/devops](https://www.reddit.com/r/devops/comments/16syzqt/)
- [Argo CD setup with Terraform on EKS — r/devops](https://www.reddit.com/r/devops/comments/1kj52hu/)

---

## 17. Lambda：基础设施和应用制品不要混成一次临时构建

### 常见反模式

```hcl
data "archive_file" "lambda" {
  # 每次 pipeline 临时从工作区打包
}
```

如果文件顺序、mtime、依赖安装或构建环境不稳定，zip hash 会不断变化，Terraform 每次都认为 Lambda 需要更新。

### 更稳的生命周期

```text
application CI
  ├─ test
  ├─ build once
  ├─ create immutable zip/image
  ├─ publish to S3/ECR
  └─ output digest/version

infrastructure/deployment
  ├─ consume exact digest/version
  ├─ update Lambda version
  ├─ integration test
  └─ promote alias between environments
```

### 高论

**[共识]**

- 代码构建应可复现；
- dev/stage/prod 推广同一制品，不要每个环境重建；
- Terraform 管 function configuration、IAM、trigger、version/alias；
- 应用团队拥有代码测试和制品；
- 独立 Lambda/服务应能独立发布，不必把所有函数绑成一个发布单体；
- `source_code_hash` 应来自不可变制品，而不是不稳定的临时目录。

参考：

- [How do you do CI/CD with Terraform and AWS Lambda? — r/devops](https://www.reddit.com/r/devops/comments/vdm4wd/)
- [Lambda functions in Terraform and unstable archive hash — r/Terraform](https://www.reddit.com/r/Terraform/comments/yrra62/)
- [Lambda local testing strategies — r/devops](https://www.reddit.com/r/devops/comments/1jocutu/)

---

## 18. 成本治理：在 PR 看变化，在 AWS 看真实账单

### Infracost 适合做什么

- 在 PR 显示预估月成本变化；
- 捕获数量级 typo，例如 2 写成 22；
- 比较 instance type、region、容量；
- 给 reviewer 一个成本信号；
- 配合 policy 限制异常增幅。

### 它不能替代什么

- AWS Cost Explorer；
- Budgets/Anomaly Detection；
- CUR 和分摊；
- 实际数据传输、请求量和 autoscaling；
- Savings Plans/RI 的真实折扣；
- 运行时闲置检测；
- 完整 FinOps owner/tagging 机制。

**[共识]** 最好的反馈位置是工程师已有的 PR，而不是再建一个没人看的 dashboard。

### Terraform 层应固化

- owner、service、environment、cost-center tags；
- log retention；
- dev 资源关停策略；
- storage lifecycle；
- gp3 等组织默认；
- NAT Gateway、EIP、snapshot 等高频浪费项的 policy；
- AWS Budget/Anomaly Detection 基础告警。

参考：

- [VS Code Terraform cost estimates — r/devops](https://www.reddit.com/r/devops/comments/vs5tj5/)
- [Monitoring infrastructure cost — r/devops](https://www.reddit.com/r/devops/comments/10999w4/)
- [AWS cost optimization patterns across many accounts — r/devops](https://www.reddit.com/r/devops/comments/1r9effj/)
- [Cost visibility must enter existing workflow — r/devops](https://www.reddit.com/r/devops/comments/1pnufax/)

---

## 19. OpenTofu：迁移常常很轻，但“轻”不等于“必须迁”

### 社区经验

**[个案]** 多位使用者描述从较早 Terraform 版本切到 OpenTofu 时，主要工作只是：

- 替换二进制；
- 更新 CI image/command；
- 检查 registry/provider 签名；
- 少量清理 pipeline。

也有人利用 OpenTofu 的 backend 变量、state encryption 或 provider 迭代能力简化多账号结构。

### 反方

**[争议]**

- 如果现有 Terraform 稳定，迁移未必产生足够业务价值；
- Terraform 和 OpenTofu 已开始分别演化，不能永远假设完全一致；
- 执行平台、商业能力、provider 发布和组织支持要一起评估；
- “一行命令能换”不代表恢复和生产验证可以省略。

### 安全迁移清单

1. 备份并验证 state；
2. 选择非生产、代表性高的 stack；
3. 同一 commit 分别运行 Terraform/OpenTofu plan；
4. 比较 resource actions 和 provider selection；
5. 检查 registry、provider checksum 和私有模块；
6. 先迁 CI，不让每个人本地自由切换；
7. create/no-op/update/import/destroy 全测；
8. 分批迁移 state，不重建稳定 VPC 只为换 CLI；
9. 定义回退版本；
10. 记录哪些功能开始依赖某一实现。

参考：

- [For everyone who migrated to OpenTofu, how was it? — r/Terraform](https://www.reddit.com/r/Terraform/comments/1gagheu/)
- [Is it wise to move from Terraform to OpenTofu? — r/Terraform](https://www.reddit.com/r/Terraform/comments/1rzg1ck/)
- [Which IaC tools are actually used in production? — r/devops](https://www.reddit.com/r/devops/comments/1ps5058/)

---

## 20. 高频危险动作

### `-target`

适合：

- 故障恢复；
- 打破特殊鸡生蛋；
- HashiCorp/错误信息明确指导的临时操作。

不适合：

- 日常部署模式；
- 永久跳过 graph 中其他变化；
- 用来掩盖 state 边界设计问题。

### `ignore_changes`

适合：

- 某个属性明确由另一个 controller 拥有；
- provider/AWS 自动维护且团队接受；
- owner 和原因被注释及测试。

不适合：

- 看不懂 diff；
- 想让 plan 变绿；
- 避免修复错误建模；
- 隐藏安全策略漂移。

### 手工 `terraform state` 命令

适合：

- import/move/恢复；
- 有备份、有锁、有变更窗口；
- 明确知道 address 与远端对象的关系。

不适合：

- 活跃 CI 同时运行；
- 没有 state version；
- 用删除 state 的方式“解决”真实资源问题；
- 不了解后续 plan 会做什么。

### `terraform apply -auto-approve`

自动化中不是原罪，前提是：

- 审批已经发生在 PR/environment gate；
- apply 的是经过确认的输入或最终 plan；
- 身份受限；
- state lock；
- 日志和回滚机制存在。

---

## 21. 争议地图：不要把偏好伪装成定律

| 争议 | A 方 | B 方 | 更好的决策问题 |
|---|---|---|---|
| Workspaces vs directories | 少重复、同构环境轻 | 隔离和审查不清 | 环境是否真同构？是否不同账号/角色？ |
| Mono-repo vs multi-repo | 原子改动、统一搜索 | 权限/发布解耦 | ownership 和 change cadence 是否一致？ |
| Terragrunt vs vanilla | 编排与 DRY | 第二层复杂度 | 你有多少 stack，当前痛点可量化吗？ |
| 公共模块 vs 自建 | 成熟、少造轮子 | 复杂、外部生命周期 | 组织需要多少定制？谁负责升级？ |
| Import vs rebuild | 保留稳定资源 | 清除历史漂移 | 数据风险与平行切流能力如何？ |
| LocalStack vs sandbox | 快、便宜、可本地化 | 真实 AWS 才有 fidelity | 哪些性质需要真实控制面证明？ |
| 集中 state vs 每账号 state | 统一和简单 | blast radius 大 | 账号是否是权限/故障边界？ |
| Terraform 管 K8s workload vs GitOps | 一种工具 | 双 state/频率不匹配 | 资源由谁持续 reconcile？ |
| 自动 drift remediation vs 告警 | 快速收敛 | 可能撤销救火 | 能否可靠判断意图？ |
| Terraform vs OpenTofu | 既有生态/平台 | 开放治理/新能力 | 迁移解决了什么具体问题？ |

读 Reddit 时，如果一个回复没有交代：

- 团队人数；
- 账号和 stack 数；
- 是否受监管；
- pipeline 形态；
- 变更频率；
- 故障恢复要求；

那么它更像偏好，不是架构结论。

---

## 22. 推荐的生产基线

### Repo/root

- root module 是 deployable unit；
- child module 不藏 provider credentials/region；
- 环境入口明确；
- module 与 provider 版本可追踪；
- `.terraform.lock.hcl` 提交；
- README 写 owner、state、role、apply 入口和恢复方式。

### Backend

- S3 versioning；
- `use_lockfile = true`；
- encryption；
- public access block；
- object prefix 最小权限；
- 删除保护；
- state 恢复演练。

### CI

- `fmt -check`；
- `validate`；
- lint；
- security scanner；
- policy-as-code；
- module tests；
- LocalStack integration；
-真实 AWS sandbox smoke；
- PR plan；
- cost delta；
- protected apply；
- drift schedule。

### AWS access

- OIDC；
- repo/branch/environment 限定；
- plan/apply role 分离；
- prod approval；
- break-glass 审计；
- 无长期 Access Key。

### Change safety

- exact Terraform runner version；
- provider lockfile；
- module version pin；
- major upgrade guide；
- dev → stage → prod；
- saved plan 视为 secret；
- 并发和 state lock；
- rollback/runbook。

---

## 23. 90 天吸收路线

### 第 1–2 周：State 与身份

- 建 S3 lockfile backend；
- 做 state version 恢复；
- 用 OIDC 分 plan/apply role；
- 对 prod prefix 做最小权限；
- 给 state 放 canary secret，确认泄露面。

### 第 3–4 周：模块与 LocalStack

- 建一个 opinionated S3 模块；
- 对公共模块与自建模块做同场测试；
- 完成 create/no-op/update/destroy；
- 加 Checkov/Conftest；
- 记录每个测试能证明什么、不能证明什么。

### 第 5–6 周：多账号

- 建 shared/dev/prod 三账号模型；
- 角色只能进入指定账号；
- state 按账号和故障域拆；
- 用 SSM 或显式 pipeline input 传少量跨 state contract；
- 演练一个账号不可达。

### 第 7–8 周：Brownfield 与 Drift

- 用 CLI 创建遗留资源；
- import 到 Terraform；
- 修到 no-op；
- 注入 drift；
- scheduled plan 自动建报告；
- 模拟应急 console change 后回写代码。

### 第 9–10 周：EKS 或 Serverless

二选一：

- EKS：Terraform 建 cluster + Argo/Flux bootstrap，GitOps 部署应用；
- Serverless：LocalStack 建 S3/SQS/Lambda/DynamoDB 链，真实 AWS 做 IAM/延迟补测。

### 第 11–12 周：升级与恢复

- Renovate/Dependabot provider PR；
- Terraform Core 固定版本升级；
- 比较 plan；
- 从 state version 恢复；
- 测一次失败的部分 apply；
- 写生产 runbook。

---

## 24. Reddit 高论压缩成代码评审问题

每个 Terraform PR 至少问：

1. 这个 root/state 的 owner 是谁？
2. 此 change 的最大 blast radius 是什么？
3. 是否有意外 replacement？
4. 是否跨 account/region？
5. provider 和 module 版本是否变化？
6. lockfile 是否合理变化？
7. plan 用了哪个角色，apply 又用哪个？
8. saved plan/JSON 是否含秘密？
9. 新资源是否符合 tags、encryption、logging、retention？
10. 成本变化是什么？
11. LocalStack 证明了哪些性质？
12. 哪些仍需真实 AWS sandbox？
13. 是否改变 state boundary 或 remote-state contract？
14. 是否引入 `ignore_changes`/`target`，为什么？
15. 是否可能与外部 controller 双重拥有？
16. rollback 是重新 apply、切流、恢复 state，还是人工操作？
17. provider/API 最终一致性会不会让一次 apply 暂时失败？
18. 这次改动能否先在低环境验证？
19. drift 检测会如何看待它？
20. runbook 和 owner 是否更新？

---

## 25. 精选阅读：r/Terraform

### 生产结构与 state

- [How do you structure Terraform projects for AWS in production?](https://www.reddit.com/r/Terraform/comments/1s4uh0j/how_do_you_structure_terraform_projects_for_aws/)  
  看点：问题面很完整；不要把评论中的单个目录当标准答案。

- [How are you managing remote-state resources?](https://www.reddit.com/r/Terraform/comments/13o1rxr/)  
  看点：backend bootstrap 的四类做法与鸡生蛋。

- [Two-year-old Terraform, production has drifted](https://www.reddit.com/r/Terraform/comments/16bq3kd/)  
  看点：长期失管后的恢复顺序、import、moved block 和平行环境。

- [Best approach for existing AWS infrastructure: import or rebuild?](https://www.reddit.com/r/Terraform/comments/1net5yl/)  
  看点：按 drift 程度与切流能力做 brownfield 决策。

### Terragrunt 与编排

- [Terragrunt: What It Solves, What It Costs](https://www.reddit.com/r/Terraform/comments/1rhmfjl/)  
  看点：两个方向的真实迁移；最值得读的是 CI 时间数据。

- [Am I the only one who doesn't like Terragrunt?](https://www.reddit.com/r/Terraform/comments/1ow9i1r/)  
  看点：错误使用 root/child module 会制造大量“Terragrunt 问题”。

- [Use cases for Terragrunt](https://www.reddit.com/r/Terraform/comments/yosx1n/)  
  看点：多账号 baseline 和依赖链。

- [Terragrunt basics](https://www.reddit.com/r/Terraform/comments/svlexs/)  
  看点：backend/provider generation 和 dependency outputs。

### Module

- [Public Terraform modules in production](https://www.reddit.com/r/Terraform/comments/bv50ez/)  
  看点：公共模块可用于生产，但必须锁版本和理解代码。

- [Using open source Terraform modules vs writing your own](https://www.reddit.com/r/Terraform/comments/1n7jlax/)  
  看点：VPC 等核心公共模块与小型自建模块的边界。

- [Custom Terraform wrappers](https://www.reddit.com/r/Terraform/comments/1kve26k/)  
  看点：内部 JS → tfjson → Make graph 这种抽象何时变成维护负担。

### Provider 与工具链

- [How do you keep provider versions up to date?](https://www.reddit.com/r/Terraform/comments/10wr2l1/)  
  看点：Renovate、maintenance cadence、root/child constraint、跨平台 lock hash。

- [Provider version upgrade workflow in CI](https://www.reddit.com/r/Terraform/comments/1ct809z/)  
  看点：升级 PR 的规范流程。

- [Terraform/provider version upgrade questions](https://www.reddit.com/r/Terraform/comments/tzv4b3/)  
  看点：lockfile 如何固定精确选择，为什么仍要读 changelog。

- [AWS Provider 5.0 heads-up](https://www.reddit.com/r/Terraform/comments/13ru7f9/)  
  看点：不锁/不维护 provider 的两种相反风险。

- [How do you manage provider major version upgrades?](https://www.reddit.com/r/Terraform/comments/1rbf9jj/)  
  看点：按 resource type 阅读 AWS provider upgrade guide。

### Testing、LocalStack 与安全

- [Anyone tried testing Terraform against LocalStack?](https://www.reddit.com/r/Terraform/comments/h057k0/)  
  看点：简单 S3 成功不代表复杂网络/provider credentials 等价。

- [Secure, testable pipeline with Terratest, LocalStack, Checkov, Conftest and Nix](https://www.reddit.com/r/Terraform/comments/1rg1jer/)  
  看点：`apply` 不是测试，测试要分静态策略、集成和可复现工具链。

- [Terraform state sensitive info](https://www.reddit.com/r/Terraform/comments/1gxxhna/)  
  看点：state 是高敏感资产，秘密来源不消除 state 持久化。

- [Terraform, GitHub Actions, many secrets: is Vault worth it?](https://www.reddit.com/r/Terraform/comments/1tc8n09/)  
  看点：Vault 可能解决问题，也可能只是增加平台复杂度；先分部署身份与应用秘密。

### Lambda 与 OpenTofu

- [Lambda functions in Terraform](https://www.reddit.com/r/Terraform/comments/yrra62/)  
  看点：不稳定 archive/hash 导致永久 diff。

- [For everyone who migrated to OpenTofu, how was it?](https://www.reddit.com/r/Terraform/comments/1gagheu/)  
  看点：迁移常很轻，但 provider registry、CI 和长期分化仍要验证。

- [Is it wise to move from Terraform to OpenTofu?](https://www.reddit.com/r/Terraform/comments/1rzg1ck/)  
  看点：不要为了换 CLI 重建稳定基础设施。

---

## 26. 精选阅读：r/devops

### 多账号与生产结构

- [Terraform for Devops: real-world experience](https://www.reddit.com/r/devops/comments/u9oqp5/terraform_for_devops_real_world_experience/)  
  看点：多账号、远程 state、Vault/assume role 和 CI。

- [AWS Control Tower with Terraform](https://www.reddit.com/r/devops/comments/1466a5f/)  
  看点：landing zone bootstrap 与 Terraform 管理层的边界。

- [AWS multi environments](https://www.reddit.com/r/devops/comments/rjjd8b/)  
  看点：Control Tower、shared services、guardrail 和账号复杂度。

- [Organizations IaC automation](https://www.reddit.com/r/devops/comments/r8qkch/)  
  看点：shared-services runner + per-account deployment role。

- [Managing tens of AWS accounts](https://www.reddit.com/r/devops/comments/163g1lw/)  
  看点：Organizations、Identity Center、集中 CloudTrail 和账号 baseline。

- [Same IaC for environments vs copy-paste](https://www.reddit.com/r/devops/comments/1bk1nts/)  
  看点：共用代码的升级竞态与环境条件爆炸。

### Modules、repo 与 self-service

- [Does your organization use provided Terraform modules?](https://www.reddit.com/r/devops/comments/ytg7o1/)  
  看点：公共模块支持和反对两方都有长期实践。

- [Terraform modules everywhere?](https://www.reddit.com/r/devops/comments/11hxk59/)  
  看点：参数透传 wrapper 为什么可能只是企业仪式。

- [Terraform library and self-service](https://www.reddit.com/r/devops/comments/tn2um5/)  
  看点：golden module、审批和内部平台。

- [Terraform monorepo pipeline](https://www.reddit.com/r/devops/comments/tk7yse/)  
  看点：Atlantis/Terragrunt 与 module repo 的组合。

- [Terraform module monorepo management tools](https://www.reddit.com/r/devops/comments/16h8rrg/)  
  看点：tag/version 与 orchestrator 的不同取舍。

### Drift 与 brownfield

- [Dealing with Terraform Drift](https://www.reddit.com/r/devops/comments/1lh1ufl/)  
  看点：禁止 console 与 scheduled detection 的现实折中。

- [Terraform drift](https://www.reddit.com/r/devops/comments/q7ecej/)  
  看点：`-detailed-exitcode`、紧急手工变更和 prod/sandbox 差异。

- [Terraform drift detection](https://www.reddit.com/r/devops/comments/10tmm83/)  
  看点：通知优于盲目自动 remediation。

- [Converting an existing deployment to Terraform](https://www.reddit.com/r/devops/comments/13wr3fp/)  
  看点：按服务逐步 import 的可行性。

- [Systematically Terraforming a brownfield](https://www.reddit.com/r/devops/comments/1j3i93y/)  
  看点：把 brownfield 当持续迁移，而不是一次性导入。

### CI、身份与 secrets

- [PSA: Love Terraform and CI/CD? You want Atlantis](https://www.reddit.com/r/devops/comments/cakyfp/)  
  看点：PR plan/apply、locking，以及使用者对并发限制的反方经验。

- [GitHub Actions AWS credentials](https://www.reddit.com/r/devops/comments/1jrvxgc/)  
  看点：OIDC 相比长期 IAM user keys。

- [GitHub OIDC security](https://www.reddit.com/r/devops/comments/xecjrn/)  
  看点：必须限制 org/repo/branch/environment。

- [OIDC use cases](https://www.reddit.com/r/devops/comments/1iir0gv/)  
  看点：PR 只读 role、main apply role。

- [IaC and secrets](https://www.reddit.com/r/devops/comments/nhuhdx/)  
  看点：即使 Git 中加密，Terraform 写资源后秘密仍可能进入 state。

- [External values: SSM/Vault or CI injection?](https://www.reddit.com/r/devops/comments/1tpnxfq/)  
  看点：非秘密配置、秘密来源、state 持久化要分开讨论。

### LocalStack、EKS 与 Lambda

- [As DevOps, do you use LocalStack?](https://www.reddit.com/r/devops/comments/1jpzxgk/)  
  看点：应用团队和平台团队对 fidelity 的不同需求。

- [Local AWS environment with LocalStack and Terraform](https://www.reddit.com/r/devops/comments/11lahst/)  
  看点：Docker Compose + LocalStack + Terraform + 本地数据库的开发环境。

- [Lambda local testing](https://www.reddit.com/r/devops/comments/1jocutu/)  
  看点：SAM、LocalStack hot reload、PR ephemeral AWS 三层。

- [Upgrading EKS programmatically](https://www.reddit.com/r/devops/comments/1l7c9bh/)  
  看点：addon 与 cluster version 必须协调。

- [How do you handle Kubernetes updates?](https://www.reddit.com/r/devops/comments/1avc1as/)  
  看点：API deprecation、control plane、node group 和低环境先行。

- [After Terraform deploys EKS, how do you deploy services?](https://www.reddit.com/r/devops/comments/16syzqt/)  
  看点：Terraform bootstrap + Argo/Flux 的明显社区倾向。

- [CI/CD with Terraform and AWS Lambda](https://www.reddit.com/r/devops/comments/vdm4wd/)  
  看点：避免把独立 Lambda 服务重新绑成发布单体。

### 成本与工具方向

- [Monitoring infrastructure cost](https://www.reddit.com/r/devops/comments/10999w4/)  
  看点：Infracost + AWS Budgets/报告是互补，不是替代。

- [AWS cost optimization across accounts](https://www.reddit.com/r/devops/comments/1r9effj/)  
  看点：EBS、snapshot、RDS、EIP、gp2、NAT、log retention 等重复浪费模式。

- [Which IaC tools are actually used in production?](https://www.reddit.com/r/devops/comments/1ps5058/)  
  看点：Terraform/OpenTofu 仍是主流，但 wrapper/orchestrator 选择高度分散。

---

## 27. 最后的社区能量

Reddit 最值得吸收的不是：

> “我们用 Terragrunt，所以你也该用。”

而是：

> “我们在什么规模、什么约束下，用它解决了什么问题，又付出了什么代价。”

把 r/Terraform 和 r/devops 的高质量经验压成一句话：

> Terraform 在 AWS 上的成熟度，不由 HCL 写得多漂亮决定，而由 state 边界、身份边界、变更证据、失败恢复和持续升级共同决定。

真正的生产级，不是一次 `apply` 成功，而是：

- 第二次 plan 能 no-op；
- 变更能被理解和审批；
- drift 能被发现和解释；
- provider 能小步升级；
- state 泄露面被控制；
- 失败的部分 apply 能恢复；
- 一个账号或服务出问题不会拖垮全图；
- LocalStack 和真实 AWS 各自证明正确的性质；
- 三个月后另一个工程师仍知道如何安全修改。

