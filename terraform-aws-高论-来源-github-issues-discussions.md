# Terraform × AWS 社区高论（来源：GitHub Issues / Discussions）

整理日期：2026-07-27  
主要来源：Terraform Core、Terraform AWS Provider、terraform-aws-modules、Terragrunt、AWS Control Tower Account Factory、LocalStack 与 OpenTofu 的 GitHub Issues / Discussions。  
筛选目标：提炼可复现故障、maintainer 解释、API 设计约束、重大升级经验和生产编排问题；排除机器人升级单、无复现问题、纯功能投票和没有后续证据的猜测。

> GitHub 搜索不可能数学意义上穷尽所有 Issue、Comment 和 Discussion。本文覆盖的是 Terraform × AWS 最主要的生产决策面，并优先选择有复现、维护者回复或修复轨迹的高信号线程。  
> Issue 的状态会随时间变化。本文保留“历史事故 / 已修复 / 仍开放 / 设计限制”的区别，不把旧 bug 写成今天仍必现。

---

## 如何阅读证据

本文使用四个标签：

- **[设计限制]**：由 Terraform graph、AWS API 或 state 模型决定，通常不能用 `depends_on` 或重试真正解决。
- **[仍开放]**：截至整理时仍是开放 Issue/Discussion，表示需求或问题存在，不表示一定能复现于所有版本。
- **[已修复/历史]**：对理解风险很有价值，但当前版本可能已经修复。
- **[生产教训]**：线程包含真实升级、故障、性能或恢复经验，具有可迁移的工程价值。

每条结论还会区分：

```text
Issue body           = 报告者的观察
maintainer comment   = 项目对原因、边界或预期行为的解释
merged/release note  = 已实现或已发布的证据
workaround comment   = 社区暂时绕行，不等于官方保证
```

---

## 0. 先吸收这 22 条

1. **AWS API 如果只提供“整份替换”，Terraform 就只能有一个 owner；拆成多个 resource/state 会互相覆盖。**
2. **远端操作成功而 provider 返回失败时，真实 AWS、state 和 CLI 结果可能三者不一致；失败后先查远端和 state，不能直接重跑。**
3. **`depends_on` 只表达顺序，不会修复 AWS IAM eventual consistency。**
4. **永久 diff 不是一种问题：可能来自 AWS 标准化、provider flatten/expand、无序集合、托管版本、默认值或双重所有权。**
5. **Provider 的大版本升级可能改变 state schema；升级后的 state 不保证能被旧 provider 解码。**
6. **“回滚 provider 版本”不是可靠 rollback；真正的回滚还需要升级前 state 版本和兼容 module。**
7. **AWS Provider v4 的 S3 refactor 说明：即使 major release 有升级指南，真实社区反馈仍可能迫使行为在后续 minor 回调。**
8. **锁 Terraform CLI、provider、module 和 LocalStack image；只锁其中一个不够。**
9. **Terraform Core 的 backend 初始化早于普通变量求值；Terraform 中 backend variables 仍是长期开放需求，使用 partial configuration。**
10. **S3 native lockfile 是当前 Terraform backend 方向；DynamoDB locking 已弃用，但旧 Issue 必须按发表时间阅读。**
11. **State 持久化失败可以留下 orphan resource；remote state versioning 和失败恢复是核心能力，不是备选项。**
12. **Saved plan 必须绑定 commit、绝对/相对路径、provider、module、变量、workspace 和 backend 上下文。**
13. **State secrets 已有 ephemeral/write-only 改善，但只有 provider 暴露相应能力时才生效。**
14. **Terraform 的动态 provider 实例仍是核心限制；多账号规模大时应拆 root/state 或评估 OpenTofu `provider for_each`。**
15. **大型 state 的主要成本既有 AWS API refresh，也有 graph、provider 内存和高连接度；拆 state 是架构选择，不只是性能 hack。**
16. **`moved` block 很安全但仍偏静态；大规模 `for_each` 地址迁移要生成、审查并分批。**
17. **公共 EKS module 的维护者自己也承认复杂度有代价；“功能最多”不是唯一质量指标。**
18. **EKS major module upgrade 可能重命名资源地址和节点组，造成 ASG replacement；必须逐条阅读 upgrade guide 和 plan。**
19. **EKS access entry 不应绑定到每次执行可能变化的 caller identity；CI 应使用稳定部署角色。**
20. **Terragrunt 的 DAG 不会消除 Terraform unknown value 和 saved-plan 限制；多 unit CI 是独立的编排产品问题。**
21. **AFT 的 account vending 很强，但 plan/approval 与持续 drift detection 仍有公开 feature gap；不能因为用了 AFT 就假设治理闭环。**
22. **LocalStack 与 AWS Provider 必须做兼容矩阵；provider 新增 API 读取可能让旧 LocalStack 卡死或失败。**

---

## 1. 本次高信号仓库地图

| 仓库 | 主要价值 | 典型问题 |
|---|---|---|
| [`hashicorp/terraform`](https://github.com/hashicorp/terraform/issues) | Terraform Core 语义与 state/backend 限制 | provider graph、backend、saved plan、moved/import、state secrets |
| [`hashicorp/terraform-provider-aws`](https://github.com/hashicorp/terraform-provider-aws/issues) | AWS API 映射与 provider bug | eventual consistency、perpetual diff、升级、API owner |
| [`terraform-aws-modules/terraform-aws-eks`](https://github.com/terraform-aws-modules/terraform-aws-eks/issues) | 大型公共 module 的真实维护压力 | EKS 升级、node group、access entry、addon |
| [`terraform-aws-modules/terraform-aws-vpc`](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues) | VPC/AZ/NAT 的地址稳定性 | 加 AZ、NAT replacement、route ownership |
| [`gruntwork-io/terragrunt`](https://github.com/gruntwork-io/terragrunt/issues) | 多 stack 编排与 CI 边界 | dependency graph、saved plan、partial destroy、性能 |
| [`aws-ia/terraform-aws-control_tower_account_factory`](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues) | AWS 多账号 vending 的产品边界 | approval、drift、customization race |
| [`localstack/localstack`](https://github.com/localstack/localstack/issues) | 模拟器与 provider 兼容性 | API 字段缺失、新 API、无限 polling |
| [`opentofu/opentofu`](https://github.com/opentofu/opentofu/issues) | Terraform 分支设计的替代方案 | state encryption、provider `for_each`、backend variables |

### 为什么没有把所有热门仓库都塞进来

筛选优先级是：

1. 有 Terraform/AWS Core 行为；
2. 有维护者解释；
3. 能改变架构或测试；
4. 有真实故障或升级影响；
5. 能从 Issue 得到可执行检查项。

只展示某个项目“能部署一个 VPC/EKS”的仓库，信息密度不如一次真实升级导致节点丢失的 Issue。

---

## 2. GitHub 高论的独特价值：它记录失败的精确形状

Reddit/论坛常说：

> “升级要小心。”

GitHub Issue 会告诉你：

```text
provider 6.14.0
aws_s3_object 已经上传成功
provider 却返回 Missing Resource Identity After Update
下一次 plan 显示 no-op
```

这会把抽象建议变成具体故障模型：

```text
远端成功
    ↓
provider response/state write 异常
    ↓
CLI 失败
    ↓
操作者误以为什么都没发生
    ↓
直接重试/回滚
    ↓
冲突、重复资源或 state 解码失败
```

来源：

- [AWS Provider 6.14.0：S3 object 已更新但 apply 报 Missing Resource Identity](https://github.com/hashicorp/terraform-provider-aws/issues/44366)
- [Terraform Core：Stronger durability of remote state](https://github.com/hashicorp/terraform/issues/19488)

---

## 3. 最根本的规则：一个远端对象只能有一个配置 owner

### S3 bucket notification 是整份替换

AWS 的 `PutBucketNotificationConfiguration` 会替换整个 bucket notification configuration。

因此下面的设计从所有权上就是冲突的：

```hcl
resource "aws_s3_bucket_notification" "team_a" {
  bucket = aws_s3_bucket.shared.id
  # ...
}

resource "aws_s3_bucket_notification" "team_b" {
  bucket = aws_s3_bucket.shared.id
  # ...
}
```

两个 resource 都认为自己拥有“整份配置”，最终会 flip-flop：

```text
team A apply → 写 A
team B apply → 用 B 替换 A
team A plan  → 发现 A 丢失，再写 A
```

**[设计限制]** maintainer 明确指出，这是上游 AWS API 的 replace semantics，不是 provider 能轻易支持多个独立 resource 的问题。

来源：

- [Multiple aws_s3_bucket_notification resources flip-flop and overwrite each other](https://github.com/hashicorp/terraform-provider-aws/issues/23951)
- [Unexpected behavior declaring multiple notifications on the same bucket](https://github.com/hashicorp/terraform-provider-aws/issues/5299)
- [Notifications can be silently overwritten](https://github.com/hashicorp/terraform-provider-aws/issues/22147)

### 正确 owner 模式

```text
shared bucket notification owner
  ├─ input: team A destinations
  ├─ input: team B destinations
  ├─ input: security events
  └─ one aws_s3_bucket_notification resource
```

跨团队可选择：

- 中央 module 聚合所有 notification；
- S3 只发到 EventBridge/SNS，团队在下游各管 subscription/rule；
- bucket owner 提供显式接口；
- 不让多个 state 写同一份 replace-style 配置。

### 同类问题：Security Group rules

Provider 文档长期警告：不要同时使用：

- `aws_security_group` 的 inline ingress/egress；
- 指向同一 security group 的 standalone rule resource。

否则两个 owner 会覆盖或反复纠正彼此。

来源：

- [Request warning when inline and standalone security group rules are mixed](https://github.com/hashicorp/terraform-provider-aws/issues/12580)
- [Existing security group with standalone rules conflict](https://github.com/hashicorp/terraform-provider-aws/issues/22642)

### 同类问题的识别法

看到 AWS API 名称或文档出现以下词时要警觉：

```text
Put...
Set...
Replace...
Configuration
Policy
Associations
Rules
```

问：

1. API 是增量添加一项，还是整份 PUT？
2. read API 能否区分每个 Terraform resource 的所有权？
3. 多个 state 是否可能调用同一个 replace API？
4. console 或其他 controller 是否也在写这份配置？

---

## 4. `depends_on` 不能解决 AWS eventual consistency

### Lambda IAM role 已创建，不代表 Lambda 立刻能使用

Issue 显示：

```text
IAM role/policy attachment 创建成功
           ↓
Lambda CreateFunction
           ↓
AWS 返回 InsufficientRolePermissions
           ↓
有时 Lambda 实际已经存在
           ↓
下一次 apply 出现 resource conflict
```

**[仍开放]** 该线程持续有人报告，包括 2026 年反馈。

来源：

- [Lambda does not account for possible IAM eventual consistency](https://github.com/hashicorp/terraform-provider-aws/issues/29828)

### 为什么显式依赖仍可能失败

```hcl
resource "aws_lambda_function" "this" {
  role       = aws_iam_role.lambda.arn
  depends_on = [aws_iam_role_policy_attachment.lambda]
}
```

这只保证 Terraform 在 policy attachment API 返回成功后再调用 Lambda API。它不能保证：

- IAM 全局传播完成；
- Lambda 服务控制面已看到新权限；
- STS/服务缓存已更新；
- 附加策略在所有内部路径可用。

### 其他典型 eventual consistency

- S3 public access block 与 bucket policy 同时创建；
- IAM role 作为 policy principal 后立刻引用；
- RAM share invitation；
- route 创建后立即读取；
- Elasticsearch/OpenSearch 日志 policy；
- SQS 创建后 provider 轮询；
- Load Balancer rule/target group；
- S3 object 创建后立即读取。

来源：

- [S3 bucket policy 与 public access block 并发可能失败](https://github.com/hashicorp/terraform-provider-aws/issues/7628)
- [IAM role principal 暂时被认为 invalid](https://github.com/hashicorp/terraform-provider-aws/issues/8905)
- [Route read-after-create consistency](https://github.com/hashicorp/terraform-provider-aws/issues/19985)
- [SQS provisioning delay](https://github.com/hashicorp/terraform-provider-aws/issues/27393)

### 工程策略

优先顺序：

1. 使用已包含 waiter/retry 的新 provider；
2. 提供最小复现并报告 provider；
3. 设计 pipeline 对可识别的 transient error 做有界重试；
4. 失败后先 refresh/read/import 判断远端是否已存在；
5. 最后才考虑显式 `time_sleep`，并标注原因和移除条件。

不要：

- 对所有失败无限重跑；
- 看到 `ResourceAlreadyExists` 就删除远端；
- 用 60 秒 sleep 掩盖错误 IAM policy；
- 把 eventual consistency 当作没有依赖关系。

---

## 5. Partial success：CLI 失败不代表 AWS 没改变

### 三种状态

| 真实 AWS | State | CLI | 风险 |
|---|---|---|---|
| 未改变 | 未改变 | 失败 | 普通失败 |
| 已改变 | 已记录 | 失败 | 下次可能 no-op，但 CI 标红 |
| 已改变 | 未记录 | 失败 | orphan/冲突，最危险 |

### Provider 6.14.0 的 S3 object 案例

**[已修复/历史]**

- S3 object 已上传；
- provider 返回 missing resource identity；
- apply 失败；
- 下一次 plan 可能显示无变更。

这说明 runbook 不能写成：

```text
apply failed → retry
```

而应写成：

```text
apply failed
  ├─ 保存日志和 plan/run id
  ├─ 检查 state 是否更新
  ├─ 读取真实 AWS
  ├─ 判断是否是 transient/provider bug
  ├─ 必要时 import
  └─ 再决定 retry/rollback
```

来源：

- [Provider 6.14.0 Missing Resource Identity After Update](https://github.com/hashicorp/terraform-provider-aws/issues/44366)
- [Terraform Core：远端 state 更强 durability 的长期提案](https://github.com/hashicorp/terraform/issues/19488)

### S3 state bucket 必须有 versioning

Terraform Core 的 durability Issue 讨论了 state 写回失败、锁释放失败和 orphan resource。即使 Terraform 后来增加了更多保护，远端 state 仍需要：

- object versioning；
- state lock；
- 恢复权限和 runbook；
- apply 后审计；
- 失败时保存本地 `errored.tfstate` 或等价恢复材料；
- 禁止未经判断直接强制解锁。

---

## 6. 永久 diff 的六类根因

### 1. AWS 托管值与声明值不同

ElastiCache Redis 6 曾要求用户声明 `6.x`，AWS 实际返回具体 minor version，导致：

```text
config: 6.x
remote: 6.0.5
plan: 6.0.5 → 6.x
apply: AWS 拒绝
```

来源：

- [ElastiCache Redis 6.x support and perpetual version diff](https://github.com/hashicorp/terraform-provider-aws/issues/15625)

### 2. AWS 丢弃显式默认值

Firehose 某些 processor 参数等于 AWS 默认值时，API/read path 不返回它们，Terraform 下一次又想加回。

来源：

- [Firehose default Lambda transform params not stored in state](https://github.com/hashicorp/terraform-provider-aws/issues/9827)
- [Firehose processors configuration still not saved correctly](https://github.com/hashicorp/terraform-provider-aws/issues/19936)

### 3. 顺序不稳定或 JSON canonicalization

IAM/KMS/S3 policy 是结构化 JSON，但 AWS/provider 可能改变：

- statement 顺序；
- action 顺序；
- whitespace；
- ARN 规范化；
- set/list 表示。

来源：

- [aws_iam_policy_document order lost when applied](https://github.com/hashicorp/terraform-provider-aws/issues/11801)
- [S3 bucket policy NotPrincipal ordering](https://github.com/hashicorp/terraform-provider-aws/issues/13144)
- [IAM policy actions ordering](https://github.com/hashicorp/terraform-provider-aws/issues/6107)

### 4. Provider flatten/expand bug

Provider 将 HCL/state 扩展成 API 请求，再把 API response flatten 回 state。两边不对称时会永久 diff。

来源：

- [DynamoDB replica with default KMS key recreated repeatedly](https://github.com/hashicorp/terraform-provider-aws/issues/29636)
- [OpenSearch advanced options perpetual diff](https://github.com/hashicorp/terraform-provider-aws/issues/21318)

### 5. 动态值在同一个 run 内改变

在 provider `default_tags` 中使用 `timestamp()` 之类的运行时变化值，曾造成：

```text
planned tags_all
≠
apply expansion tags_all
```

最终触发 inconsistent final plan。

来源：

- [Provider produced inconsistent final plan for tags_all](https://github.com/hashicorp/terraform-provider-aws/issues/19583)

### 6. 双重所有权

多个 resource/state/controller 写同一远端配置，会形成真正的 flip-flop，不是 provider canonicalization。

来源：

- [Multiple S3 notification resources overwrite each other](https://github.com/hashicorp/terraform-provider-aws/issues/23951)
- [Inline and standalone security group rules conflict](https://github.com/hashicorp/terraform-provider-aws/issues/12580)

### 永久 diff 的排查顺序

```text
1. 是否两个 owner？
2. AWS read API 实际返回什么？
3. 是否只是顺序/标准化？
4. 是否显式设置了 AWS 会丢弃的默认值？
5. 是否使用 timestamp/random/current caller 等动态值？
6. provider changelog/Issue 是否已有回归？
7. 是否应升级、显式归一化或合理 ignore_changes？
```

`ignore_changes` 应是最后的所有权声明，不是第一个消音器。

---

## 7. `default_tags`：方便，但必须是 run 内稳定值

### 典型错误

```hcl
provider "aws" {
  default_tags {
    tags = {
      CreatedAt = timestamp()
    }
  }
}
```

`timestamp()` 在不同求值阶段变化，可能造成 inconsistent plan，且每次运行都会推动标签变化。

### 更稳的模式

```hcl
provider "aws" {
  default_tags {
    tags = {
      Environment = var.environment
      Owner       = var.owner
      ManagedBy   = "terraform"
      Repository  = var.repository
    }
  }
}
```

构建时间、部署 ID 等需要变化的值，应明确决定：

- 是否真的属于资源 desired state；
- 是否应由 CI/observability 保存；
- 是否会造成每次 apply 都改资源；
- 是否违反不可变审计语义。

相关 Issue：

- [default_tags always shows an update](https://github.com/hashicorp/terraform-provider-aws/issues/18311)
- [default_tags identical to resource tags caused errors](https://github.com/hashicorp/terraform-provider-aws/issues/19204)
- [inconsistent final plan for tags_all](https://github.com/hashicorp/terraform-provider-aws/issues/19583)

---

## 8. AWS Provider major upgrade：升级 state 后不能假设可降级

### v6 的关键教训

AWS Provider v6 引入了新的 resource identity/multi-region 方向和大量 breaking changes。

在 downgrade Issue 中，maintainer 明确指出：

> Downgrading provider versions is not supported.

升级后的 state 可能包含旧 provider 不认识的 identity attributes，例如：

```text
failed to decode identity: unsupported attribute "account_id"
```

来源：

- [AWS Provider v6.0.0 planning and release thread](https://github.com/hashicorp/terraform-provider-aws/issues/41101)
- [v6 → v5 state compatibility and unsupported downgrade](https://github.com/hashicorp/terraform-provider-aws/issues/43178)
- [Upgrade to v6: state could not be decoded](https://github.com/hashicorp/terraform-provider-aws/issues/46014)

### 真正的 upgrade rollback

升级前必须同时保留：

- 精确 Terraform CLI version；
- 精确 AWS provider lockfile；
- module source/version；
- state object version；
- plan/log；
- CI image digest；
- migration/moved/import steps。

如果升级后必须回退，可能需要：

```text
restore pre-upgrade state object version
            +
restore pre-upgrade code/module/provider
            +
verify real AWS did not receive incompatible changes
```

仅把：

```hcl
version = "~> 5.0"
```

改回去，不构成可靠 rollback。

### 升级 PR 的正确粒度

不要在一个 PR 中同时：

- Terraform Core major/minor；
- AWS Provider major；
- EKS module major；
- Kubernetes version；
- node AMI；
- access model；
- Terragrunt major。

推荐：

```text
1. Terraform Core
2. provider patch/minor
3. provider major
4. module major
5. service version
6. workload migration
```

每步都在低环境 no-op/apply 后再推进。

---

## 9. AWS Provider v4 S3 refactor：升级指南也会被现实修正

### 历史轨迹

AWS Provider v4 对 `aws_s3_bucket` 做了重大 refactor，把 versioning、lifecycle、replication 等能力拆向独立资源。

真实反馈显示：

- v4.0.0–v4.8.0 的行为给现有自动化升级造成严重困难；
- 用户需要 import 新资源以避免数据风险；
- 后续 v4.9.0 回调部分行为；
- drift detection 的语义也随之调整。

来源：

- [Upcoming Changes in AWS Provider Version 4.0](https://github.com/hashicorp/terraform-provider-aws/issues/20433)
- [Discussion around changes to aws_s3_bucket in v4](https://github.com/hashicorp/terraform-provider-aws/issues/23106)
- [S3 attributes became unconfigurable](https://github.com/hashicorp/terraform-provider-aws/issues/23103)
- [S3 versioning configuration upgrade failure](https://github.com/hashicorp/terraform-provider-aws/issues/23125)

### 高论

1. Major release 的 upgrade guide 是起点，不是生产证明；
2. 不要 day-one 全量升级关键 state；
3. 选择代表性 fixture 覆盖真实资源组合；
4. import/moved 步骤也必须自动化、可审查；
5. provider 新行为可能在后续 minor 调整；
6. 升级观察期内不要清理旧 state version 和旧 runner image。

---

## 10. Provider、Module、CLI 和 Emulator 是一个兼容矩阵

只写：

```hcl
required_providers {
  aws = {
    version = "~> 6.0"
  }
}
```

还不够。真实执行结果由以下矩阵共同决定：

| 维度 | 示例 |
|---|---|
| Terraform/OpenTofu | Terraform 1.15 / OpenTofu 1.12 |
| AWS Provider | 6.12 / 6.13 / 6.23 |
| Module | EKS v20 / v21 |
| AWS service API | 区域是否已支持新字段 |
| LocalStack | 4.11 / 4.12 |
| Runner OS/arch | Linux amd64 / Darwin arm64 |
| Provider mirror/cache | 官方 registry / mirror / cache |

### 真实兼容性事故

LocalStack 的 DynamoDB `DescribeTable` 缺少 AWS Provider 6.13 开始依赖的 `WarmThroughput` 字段，导致：

```text
DynamoDB table 已 ACTIVE
Terraform provider 持续轮询
最终 couldn't find resource
```

来源：

- [LocalStack DynamoDB missing WarmThroughput caused Provider 6.13+ polling loop](https://github.com/localstack/localstack/issues/13140)

AWS Provider 6.23 开始对 S3 调用新的 S3 Control tags API，旧 LocalStack 无法支持；更新 LocalStack 后恢复。

来源：

- [Provider 6.23 + LocalStack S3 Control incompatibility](https://github.com/hashicorp/terraform-provider-aws/issues/45292)

### 推荐兼容矩阵文件

```yaml
terraform: 1.15.1
aws_provider: 6.45.0
localstack: 4.x.y@sha256:...
modules:
  eks: 21.x.y
  vpc: 6.x.y
platforms:
  - linux_amd64
  - darwin_arm64
```

升级其中任何一项都运行：

```text
create → no-op → update → import → destroy
```

---

## 11. Backend variables：初始化阶段与普通配置阶段不同

### 长期开放的 Terraform Core 需求

Terraform backend 在普通 variable evaluation 之前初始化，所以不能简单写：

```hcl
terraform {
  backend "s3" {
    bucket = var.state_bucket
    key    = "${var.environment}/terraform.tfstate"
  }
}
```

截至整理时，Terraform Core 的相关 Issue 仍开放。

来源：

- [Using variables in Terraform backend config block](https://github.com/hashicorp/terraform/issues/13022)
- [Allow interpolation inside backend configuration](https://github.com/hashicorp/terraform/issues/17288)

### Terraform 中的稳健方式：Partial configuration

```hcl
terraform {
  backend "s3" {}
}
```

```hcl
# backend-prod.hcl
bucket       = "org-prod-tfstate"
key          = "network/terraform.tfstate"
region       = "ap-southeast-2"
use_lockfile = true
```

```powershell
terraform init -backend-config=backend-prod.hcl
```

### 安全注意

- backend credentials 优先通过 AWS credential chain/OIDC/env；
- 不把秘密硬编码进 backend HCL；
- `-backend-config` 值可能进入 `.terraform` 和 plan 相关材料；
- CI 中明确绑定 environment → backend config；
- 避免用 `sed` 临时改 source；
- backend config 变化要触发明确的 `-reconfigure`/migration 流程。

### OpenTofu 差异

OpenTofu 已支持更动态的 backend 配置能力；迁移时必须记录这是 implementation-specific feature，避免以后误切回 Terraform。

来源：

- [OpenTofu: Enable variables in backend config](https://github.com/opentofu/opentofu/issues/388)

---

## 12. S3 state locking：新旧 Issue 必须按时代阅读

当前 Terraform S3 backend 支持：

```hcl
terraform {
  backend "s3" {
    bucket       = "example-tfstate"
    key          = "prod/network.tfstate"
    region       = "ap-southeast-2"
    use_lockfile = true
  }
}
```

DynamoDB locking 已弃用。

事实校正：

- [Terraform S3 backend: native lockfile and deprecated DynamoDB locking](https://developer.hashicorp.com/terraform/language/backend/s3)

### Native lockfile 的真实边缘 Issue

- 缺少 `s3:DeleteObject` 时，某些 migration path 的错误提示曾不明显；
- lockfile 被外部删除后，lock timeout 路径出现问题；
- S3-compatible 实现未必完全兼容；
- backend 只对 Amazon S3 做正式测试。

来源：

- [Missing DeleteObject permission during migrate-state](https://github.com/hashicorp/terraform/issues/36407)
- [Lock timeout when lockfile is externally deleted](https://github.com/hashicorp/terraform/issues/37324)
- [S3 lockfile and Backblaze B2 incompatibility](https://github.com/hashicorp/terraform/issues/37143)

### 运行建议

- state object 和 `.tflock` 分别授权；
- lockfile 需要 Get/Put/Delete；
- state object本身通常不应授予 Delete；
- bucket versioning；
- 强制解锁前确认没有活跃 run；
- CI 用 stack-level concurrency key；
- 不把跨 state 依赖误认为 state lock 会处理。

---

## 13. Saved plan：不是一张截图，是可执行的上下文快照

### 历史事故

Terraform 曾有 saved plan 可被重复 apply 的 bug，造成重复创建和 orphan cleanup。

来源：

- [Terraform apply executes on stale plan file](https://github.com/hashicorp/terraform/issues/24078)

即使当前版本已有更多 stale plan 保护，saved plan 仍必须被当作：

- 敏感 artifact；
- 单次使用；
- 与 state serial 绑定；
- 与 provider/module/variables 绑定；
- 与工作目录和初始化上下文绑定。

### Terragrunt 进一步放大上下文

Terragrunt plan/apply 分两台机器时，还会依赖：

- `.terragrunt-cache` 路径；
- provider plugins/cache；
- 每个 unit 的 planfile；
- 相同绝对路径；
- dependency outputs；
- init 行为；
- lockfile 的 platform hashes。

来源：

- [Terragrunt: unable to run Plan and Apply on different machines](https://github.com/gruntwork-io/terragrunt/issues/494)
- [Terragrunt apply planfile still initializes dependencies](https://github.com/gruntwork-io/terragrunt/discussions/5357)
- [run --all/stacks and deterministic CI planfiles](https://github.com/gruntwork-io/terragrunt/discussions/4588)

### 推荐 artifact manifest

```json
{
  "commit": "abc123",
  "root": "accounts/prod/network",
  "terraform": "1.15.1",
  "provider_lock_sha256": "...",
  "module_manifest_sha256": "...",
  "backend_fingerprint": "...",
  "workspace": "default",
  "varset_revision": "...",
  "plan_sha256": "...",
  "expires_at": "..."
}
```

apply 前验证所有字段，apply 后销毁 plan artifact。

---

## 14. State secrets：Ephemeral/write-only 是进步，不是全局魔法

### 历史核心问题

Terraform 从早期就会把 RDS password 等敏感值放进 state。长期 Issue 讨论的难点是：

- 不保存某个值，下一次如何判断 drift？
- provider/resource schema 哪些字段能安全不持久化？
- provider configuration 与资源 identity 如何区分？
- secret 被其他资源引用时如何在下一次运行恢复？

来源：

- [Storing sensitive values in state files](https://github.com/hashicorp/terraform/issues/516)
- [Ability to not store certain attributes in state](https://github.com/hashicorp/terraform/issues/30469)

### 当前改善

Terraform 现在支持：

- ephemeral resources；
- write-only arguments；
- ephemeral variables/outputs。

例如 provider 暴露 `password_wo` 时，秘密可以不进入 state/plan。

事实校正：

- [Terraform ephemeral values and write-only arguments](https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral)

### 限制

**[设计限制]**

- provider 必须为具体 resource 实现 write-only argument；
- 普通 `password` 参数不会因为传入 ephemeral 值就自动变 write-only；
- 旧 module 可能只暴露普通参数；
- `terraform show -json`、日志和 external scripts 仍可能泄露其他值；
- state 中已有的秘密不会自动从历史版本消失；
- backend/versioned object 的旧版本仍需保护。

### 审查问题

1. 这个 provider/resource 支持 `_wo` 或 ephemeral 吗？
2. module 是否把 write-only 参数暴露出来？
3. 旧 state versions 是否含 secret？
4. saved plan 是否仍包含其他敏感属性？
5. CI artifact retention 是多少？
6. secret rotation 如何驱动 write-only version？

---

## 15. 动态 Provider：Terraform 多账号扩展的长期天花板

### 经典需求

```text
accounts = [A, B, C, ...]
对每个 account 创建一个 assume-role AWS provider
再对每个 account 实例化同一 module
```

Terraform 用户长期希望：

```hcl
provider "aws" {
  for_each = var.accounts
  # ...
}
```

但 Terraform Core 中相关需求仍开放。

来源：

- [Instantiating Multiple Providers with a loop](https://github.com/hashicorp/terraform/issues/19932)
- [Dynamically-generated provider configurations](https://github.com/hashicorp/terraform/issues/16967)
- [Pass providers to modules in for_each](https://github.com/hashicorp/terraform/issues/24476)
- [Dynamic provider configuration assignment](https://github.com/hashicorp/terraform/issues/25244)

### 为什么这不是普通 `for_each`

Provider configuration 参与：

- graph construction；
- resource identity/namespace；
- refresh；
- import；
- destroy；
- state 中 provider address。

Terraform 必须在足够早的阶段知道哪个 resource 由哪个 provider 实例负责。

### Terraform 中的工程选择

1. 账号少：显式 aliases；
2. 账号多：每 account/root/state 一次 pipeline；
3. CI 生成 root configuration；
4. Terragrunt 生成 provider/backend；
5. HCP Terraform Stacks；
6. 评估 OpenTofu provider `for_each`。

### OpenTofu 并没有消灭 graph 规则

OpenTofu 已提供 provider `for_each`，但同一个 `for_each` module 的一个实例不能随意依赖另一个实例的输出，否则仍可能形成自引用 cycle。

来源：

- [OpenTofu provider for_each typo/Q&A](https://github.com/orgs/opentofu/discussions/2804)
- [Provider for_each with cross-region KMS replica cycle](https://github.com/orgs/opentofu/discussions/2760)

### 多账号推荐边界

```text
account inventory
       ↓
CI matrix / stack generator
       ↓
one root + one provider identity + one state per deployment unit
```

比一个 graph 动态管理 100 个账号更容易做到：

- 并发控制；
- 账号级权限；
- 故障隔离；
- drift；
- 重试；
- 分批 rollout。

---

## 16. Large state：性能问题最终会变成组织边界问题

经典 Terraform Core Issue 中，约 1600 个 AWS resources：

- create 约 30 分钟；
- 小改动 plan 约 8 分钟；
- state 约 8.6 MB；
- 内存 2 GB 以上。

来源：

- [Terraform performance with large number of resources](https://github.com/hashicorp/terraform/issues/16375)

### 性能由多部分构成

```text
backend read
  + config/module loading
  + provider startup/schema
  + graph construction/transforms
  + AWS API refresh
  + throttling/retry
  + diff computation
  + policy/cost tooling
```

提高 `-parallelism` 只影响部分 provider operations，不会自动解决：

- 高连接度 graph；
- provider 内存；
- AWS throttling；
- state lock；
- 所有团队共享 blast radius。

### 社区实际绕行

Issue 评论中，多位用户最终：

- 拆多个 state；
- 使用 Terragrunt；
- 把每次 deployment 控制到较小 resource 数；
- 接受跨 state contract 和 orchestration 成本。

### 切分准则

不是固定“200 resources”或“1000 resources”，而是同时观察：

- P50/P95 plan time；
- refresh API 数；
- state lock wait；
- provider RSS；
- graph node/edge 数；
- 团队冲突；
- 一次 replacement 的最大风险；
- 失败恢复时间。

当性能、权限和所有权同时指向同一边界时，拆 state 最有价值。

---

## 17. State 拆分会带来 contract，不是免费午餐

Terraform 的早期讨论已经指出：

- 单 state 可直接形成完整 graph；
- 多 state 降低 blast radius；
- 但需要 outputs、remote state 或外部 registry 共享信息；
- 多 state 之间没有原子事务。

来源：

- [Any benefit of using separate remote state files?](https://github.com/hashicorp/terraform/issues/3838)

### 跨 state contract 的优先级

1. 稳定、低频、明确的 ID/ARN；
2. SSM Parameter Store / service discovery / DNS；
3. pipeline 显式传递；
4. `terraform_remote_state`；
5. 尽量避免直接读取对方所有 state。

为什么不默认 remote state：

- state 读取权限通常能看到所有 outputs/敏感内容；
- producer state 结构变化会破坏 consumer；
- 环形依赖难处理；
- plan 需要 producer state 可访问。

### Contract 应有版本

```text
/platform/network/v1/vpc-id
/platform/network/v1/private-subnet-ids
```

改变 contract 时：

- 先发布新版本；
- consumer 迁移；
- 再删除旧版本；
- 不让一个 apply 同时破坏所有下游。

---

## 18. `moved` / Import：安全重构仍有静态边界

### `moved` block 的价值

相比直接 `terraform state mv`：

- 可进 Git；
- 可 code review；
- plan 会显示 address move；
- 团队成员不会错过本地 state command；
- module 用户可随版本获得迁移。

### 当前限制

大规模 `for_each` 地址迁移仍常需要大量静态 moved blocks。

来源：

- [Dynamic moved blocks](https://github.com/hashicorp/terraform/issues/33236)
- [State migration actions driven by configuration](https://github.com/hashicorp/terraform/issues/19354)
- [Move resources from one state to another](https://github.com/hashicorp/terraform/issues/32777)

### 安全流程

```text
1. 冻结 address 集合
2. 备份 state
3. 生成 moved blocks
4. 人工检查 from/to 一一对应
5. plan 必须只显示 move，不显示 destroy/create
6. 分环境发布
7. 保留 moved blocks 足够长时间
8. 最后再清理旧兼容路径
```

### Import 不是 preserve-current 的按钮

Import 只建立：

```text
resource address ↔ remote object ID
```

如果 HCL 没描述远端现有属性，下一次 plan 会试图把它们改回配置/default。

OpenTofu 从 CDK import Aurora 的 Discussion 就出现：

- tags 被删除；
- snapshot flags 改变；
- logging/performance attributes 被清空；
- serverless scaling config 变化。

来源：

- [Migrating AWS CDK resources without unintended changes](https://github.com/orgs/opentofu/discussions/2909)

正确目标：

```text
import → generated/handwritten config → reconcile → no-op plan → 才允许重构
```

---

## 19. 公共 EKS module：维护者自己也在承受抽象税

terraform-aws-eks maintainer 曾公开询问：

> Is the complexity of this module getting too high?

列出的压力包括：

- managed node groups；
- Kubernetes provider；
- Windows；
- IRSA；
- Fargate；
- 越来越多资源和条件；
- 每次 release 的新增 issue。

来源：

- [Is the complexity of this module getting too high?](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/635)

### 高论

公共 module 的价值：

- 大量 AWS/EKS 边缘处理；
- 社区复现和修复；
- upgrade guides；
- node group/Fargate/KMS/access 组合。

公共 module 的代价：

- 变量和条件数量巨大；
- major upgrade 的迁移面广；
- 资源 address 变化；
- 你不使用的功能也影响 schema/graph；
- provider/module/Core 最低版本一起上升；
- maintainers 必须为相反需求做折中。

### 使用策略

不要在“直接使用”和“完全重写”之间二选一：

```text
thin root wrapper
  ├─ pin module version
  ├─ 固化组织默认
  ├─ 只暴露允许改变的 inputs
  ├─ 添加 tests
  └─ 保留官方 upgrade guide 的可见性
```

避免：

- fork 后永不跟上游；
- 再包一层并透传全部 200 个变量；
- module major 与 EKS major 同时升；
- 只看 example apply 成功，不测试升级。

---

## 20. EKS module major upgrade：Resource address 变化就是生产事件

### v17 → v18 的真实破坏面

用户报告：

- worker/ASG resource address 改名；
- plan 想重建 Auto Scaling Groups；
- 旧节点 role 未保留在 `aws-auth` 时，节点失联；
- 有用户描述忘记迁移步骤后节点立即丢失并造成严重 downtime。

来源：

- [Release 18+ Upgrade Guide Breaks Existing Deployments](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1744)
- [Upgrading v17 to v18 wants to destroy AutoScalingGroups](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2100)
- [Migration v17 to v18 caused cycle and destroyed resources](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2038)

Maintainer 的关键解释：

> v18 几乎所有东西都变了，因为旧 module 已经不可持续；Terraform 缺乏足够抽象时，breaking change 会非常 disruptive/destructive。

### 升级前必须做

```text
□ 精确 pin 旧/新 module
□ 阅读每个中间 major upgrade guide
□ 保存 state version
□ 列出 resource address change
□ 为 address move 准备 moved/import
□ 确认旧 node role 仍能加入 cluster
□ 检查 PDB、容量和 drain
□ 在非生产复制真实 node group 组合
□ plan 中任何 ASG/node replacement 都人工解释
□ 预留并行容量
```

### 不要把 module upgrade 当依赖机器人普通 PR

自动创建 PR 可以，但 merge gate 应检测：

- node group replacement；
- cluster replacement；
- access entry replacement；
- KMS key replacement；
- security group rule删除；
- addon version变化；
- bootstrap user data/AMI 变化。

---

## 21. EKS Access Entry：执行 Terraform 的身份必须稳定

### Caller identity 漂移

EKS module 的：

```hcl
enable_cluster_creator_admin_permissions = true
```

会把执行 Terraform 的 entity 加为 access entry。

如果今天是工程师 A 本地 apply，明天是 GitHub role，后天是工程师 B：

```text
cluster_creator principal ARN 改变
→ access entry replacement
→ 每次执行身份不同就持续 diff
```

maintainer 建议：

- Terraform 使用一个稳定 identity；
- 或关闭自动 cluster creator entry；
- 通过 `access_entries` 显式声明角色。

来源：

- [cluster_creator access entry replaced when another user runs Terraform](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2911)

### 推荐

```hcl
enable_cluster_creator_admin_permissions = false

access_entries = {
  platform_admin = {
    principal_arn = aws_iam_role.platform_admin.arn
    # explicit policies/scopes
  }
}
```

### 从 aws-auth 迁到 access entries

现有 access entry 可能由 EKS 自动创建。Terraform 再创建会返回 409 `ResourceInUseException`。

处理：

- 先盘点现有 entries；
- 判断谁创建、谁应拥有；
- import 已存在 entry/policy association；
- Karpenter/node group module 避免重复创建；
- 分阶段从 ConfigMap 迁移；
- 不在 apply prompt 等待期间去 console 删除生产 access entry。

来源：

- [Creating an access entry fails if it already exists](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2968)
- [Trouble migrating from configmap to access entries](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3424)
- [aws-auth submodule reports Unauthorized](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3082)

---

## 22. EKS：Cluster 创建与 Kubernetes API 操作最好分层

EKS v21 讨论中，用户总结了多年经验：

- 把 EKS cluster/access entries 与 Kubernetes/Helm provider 放同一 run，容易出现 chicken-and-egg；
- 新 CI role 需要 access entry 才能访问 Kubernetes API；
- 但 plan/provider 初始化又需要可访问 cluster；
- cluster 删除/升级时 Kubernetes provider 也容易失败。

来源：

- [EKS module v21 proposed changes](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3394)

### 推荐层次

```text
Layer 1: AWS foundation
  VPC, IAM, KMS, EKS control plane, node groups, access entries

Layer 2: cluster bootstrap
  CNI/Cilium, Argo CD/Flux, critical addons

Layer 3: workloads
  GitOps repositories and application releases
```

### 何时可以同一个 root

- 简单实验；
- provider 能安全 defer；
- public endpoint/credentials 稳定；
- cluster 不会在同一 run 创建和销毁；
- 团队接受一次 graph 的失败耦合。

### 何时应该拆

- private endpoint；
- CI role/access entry 会变；
- 自定义 CNI 必须先于 nodes；
- cluster 生命周期与 addons 不同；
- GitOps 已存在；
- 需要独立恢复。

---

## 23. VPC module：加一个 AZ 可能不是“只加一个 subnet”

公共 VPC module 需要根据列表 index 分配：

- subnet；
- route table；
- NAT Gateway；
- EIP；
- network ACL；
- route。

修改 AZ/subnet 列表顺序或长度时，resource address/index 可能变化。

真实 Issue：

- 增加新 AZ/subnets 曾计划 destroy NAT Gateway；
- NAT replacement 的删除/创建顺序问题；
- 用户惊讶于创建过多 NAT Gateways；
- route 已存在冲突。

来源：

- [Adding a new AZ and subnets destroys NAT gateways](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/106)
- [Error replacing NAT: old NAT not deleted before new](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/999)
- [Too many NAT gateways created](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/117)
- [RouteAlreadyExists](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/44)

### 高论

对于长期运行 VPC：

- 不要依赖 list index 表达稳定 identity；
- AZ → subnet CIDR 显式映射；
- 检查 module 是否使用 `count` 或 `for_each`；
- 加 AZ 前检查所有 index-based resources；
- 评估 single NAT、one NAT per AZ、无 NAT/fck-nat 的成本与可靠性；
- NAT replacement 需要 egress 中断计划；
- 网络 state 不随应用频率升级 module。

---

## 24. Terragrunt 不会消除 Terraform unknown value

### CI 中最尖锐的 Discussion

用户希望：

```text
unit A plan → 产生尚未知的 output
unit B plan → 使用 A 的未来 output
保存所有 plan
审批
apply 所有 plan
```

问题是 Terraform/OpenTofu saved plan 中，不能简单告诉下游：

> 这个变量现在未知，但 apply A 后你再重新算。

如果使用 mock output：

- plan 可能基于假 ARN/ID；
- apply saved plan 仍绑定假值；
- provider 可能要求格式正确；
- 重新 plan 又破坏“审的是同一个 plan”。

来源：

- [Terragrunt run --all and stacks in CI with deterministic plan files](https://github.com/gruntwork-io/terragrunt/discussions/4588)

### 这意味着什么

多 state dependency 的生产 CI 通常必须选择：

1. 上游先 apply，再 plan 下游；
2. 只对已存在 outputs 使用 saved plan；
3. 新依赖分多个 PR/阶段；
4. apply 前重新 plan 下游并重新审批；
5. 使用具有 run graph/approval 语义的 orchestration platform。

Terragrunt 能描述 DAG，但不能违反 Terraform plan 的信息模型。

---

## 25. Terragrunt 性能：依赖输入递归会指数式放大工作

一个 Issue 给出明确数据：

```text
旧版本 graph-dependencies ≈ 0.5 秒
新版本 ≈ 14 秒
约 15 倍变慢
```

根因讨论指出，读取 dependency 的 inputs 会迫使 Terragrunt 递归解析：

```text
dependency
  → dependency inputs
    → those inputs' dependencies
      → ...
```

来源：

- [Terragrunt 0.50.15 15x slower with multiple dependencies](https://github.com/gruntwork-io/terragrunt/issues/2806)

### 高论

在多 stack 系统中，性能不是只有 Terraform resource 数：

- HCL include；
- `read_terragrunt_config`；
- dependency outputs；
- dependency inputs；
- remote state reads；
- module downloads；
- provider cache；
- DAG discovery。

### 建议指标

```text
parse time
dependency discovery time
remote state read count
provider download bytes
plan time per unit
critical path
parallel lock contention
```

### Stacks 不是自动性能优化

有团队已有最多约 50 units 的 package，质疑 stacks 是否只是多一层抽象，并未自动减少已有 unit/dependency 定义。

来源：

- [Terragrunt stacks: worth the effort?](https://github.com/gruntwork-io/terragrunt/discussions/5998)

引入 stack abstraction 前必须明确：

- 它减少哪类重复？
- 是否改变执行 DAG？
- 是否减少 parse/remote state reads？
- CI 是否原生理解它？
- destroy/resume 如何处理？

---

## 26. Terragrunt Partial destroy：下游被删后，输出也消失了

场景：

```text
base
 ├─ module1
 │   └─ module1_extension
 └─ module2
```

`run-all destroy`：

- extension 和 module1/module2 已成功删除；
- base 删除失败；
- 再运行 destroy；
- Terragrunt 解析已删除 dependency 的 outputs 失败；
- 无法简单 resume。

来源：

- [Rerun run-all destroy after partial success](https://github.com/gruntwork-io/terragrunt/issues/3183)

### 为什么 mock 不容易自动生成

Provider 可能要求：

- 合法 ARN；
- 合法 CIDR；
- account ID；
- region；
- resource ID 格式。

Terragrunt 不知道 output 将被下游如何验证，因此不能生成一个通用 `"mock"`。

### 设计建议

- destroy path 单独测试；
- dependency block 为 destroy 提供明确 mock outputs；
- 不让下游 HCL 在 destroy 时强制读取已不存在的真实资源；
- run report/ledger 保存成功和失败 unit；
- 对重要环境提供逐 unit 恢复命令；
- 小心把超深 DAG 当成便利。

---

## 27. Git 删除一个目录，不会自动告诉 Terraform 销毁旧 state

CI 看到新 commit 时：

```text
old ref: units/foo/terragrunt.hcl
new ref: foo 目录不存在
```

只在 new ref 运行 Terragrunt，工具已经看不到 `foo`，自然不知道应 destroy。

来源：

- [Setting up infrastructure through a CI pipeline](https://github.com/gruntwork-io/terragrunt/discussions/3639)

Maintainer 分享的思路：

```text
added/changed unit:
  plan in source ref
  apply in post-merge ref

removed/renamed unit:
  plan -destroy in target/pre-merge ref
  destroy in target/pre-merge ref
```

### 高论

GitOps/IaC pipeline 必须理解三个 ref：

- source；
- target before merge；
- post-merge。

目录删除是基础设施事件，不只是“没有文件需要执行”。

### Pipeline 检查

- 检测 `terragrunt.hcl` 删除/rename；
- 保存旧 ref；
- 在旧配置上下文生成 destroy plan；
- 审批 destroy；
- merge 后确保 destroy 执行；
- 处理 rollback/reopen；
- 不靠扫描 backend 里孤立 state 作为唯一机制。

---

## 28. Terragrunt 并发：共享 dependency 也会形成 lock contention

公开 Issue 报告约 80 units、深依赖图、S3 backend，在 `run --all plan` 中：

- 多个 unit 并发解析/读取共享 dependency；
- 本地 `.terraform/terraform.tfstate` lock contention；
- 某些版本由 retry 掩盖；
- 后续版本演变为 fatal parse error；
- 只在复杂 CI 中稳定出现，难以最小复现。

来源：

- [run --all plan state lock contention on shared dependencies](https://github.com/gruntwork-io/terragrunt/issues/6002)

**[仍开放]** 这条更适合作为风险模式，不应写成所有 Terragrunt 版本必现。

### 工程建议

- 固定 Terragrunt 版本；
- 在代表性大图上做升级 benchmark；
- CI 与本地都测试；
- 记录并发度；
- 共享 dependency init/读取必要时串行；
- 升级不能只跑一个 3-unit fixture；
- 保留旧 runner image 用于故障对照。

---

## 29. Provider lockfile：跨平台 hash 是实际 CI 问题

Terragrunt Discussion 中的真实环境：

- 本地 `darwin_arm64`；
- CI `linux_amd64`；
- 约 135 Terragrunt modules；
- 少于 10 个 providers；
- provider cache 减少下载；
- 但 lockfile 只有当前平台 `h1` hash；
- CI 使用 `-lockfile=readonly` 时失败。

来源：

- [Multi-platform provider cache and h1 hashes](https://github.com/gruntwork-io/terragrunt/discussions/6136)

### 结论

Provider cache 优化下载，不自动等于完整 lockfile。

需要明确生成目标平台 hashes：

```powershell
terraform providers lock `
  -platform=darwin_arm64 `
  -platform=linux_amd64
```

在 Terragrunt 多 unit 环境还要解决：

- 每个 root 的 lockfile；
- 重复下载；
- registry rate limit；
- mirror 的 checksum 来源；
- read-only lockfile；
- provider cache 并发安全。

---

## 30. AFT：Account vending 不等于完整 Terraform governance

AWS Control Tower Account Factory for Terraform 解决：

- account request；
- account vending；
- global/account customization；
- account metadata；
- pipeline。

但公开 feature requests 显示两个重要 gap。

### Plan + Approval

用户请求为 AFT CodePipeline 增加：

```text
terraform plan
manual approval
terraform apply
```

以避免 customization 直接 apply。

来源：

- [AFT: Add plan and approval stage for CodePipeline](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/481)

### Continuous drift detection

用户请求为每账号 customization pipeline 增加 schedule：

```text
EventBridge schedule
  → TF_ACTION=plan
  → terraform plan -detailed-exitcode
  → exit 2 notification
```

来源：

- [AFT: Scheduled customization plan for continuous drift detection](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/628)

**[仍开放]** 这些是需求，不应误写为 AFT 已内建。

### AFT race 案例

AFT 曾在同步 custom fields 时：

- 先删除所有相关 SSM parameters；
- 再重新创建；
- 中间窗口依赖方读不到参数；
- 最终 state 一致，但过程不安全。

后来在 release 中修复。

来源：

- [AFT SSM custom fields delete/recreate race](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/531)

### 高论

最终 state 正确，不代表 transition 安全。治理设计必须检查：

- 是否先删后建；
- consumer 在中间态的行为；
- plan/approval；
- drift；
- customization 重跑；
- account factory 版本升级；
- Step Functions/CodePipeline 的失败恢复。

---

## 31. LocalStack Ultimate：固定版本比“latest”更重要

### 双向变化

LocalStack 兼容问题有两种来源：

```text
AWS Provider 新增 API/read 字段
  → 旧 LocalStack 不支持

LocalStack 实现/响应变化
  → 当前 provider 预期不匹配
```

真实案例：

- DynamoDB `WarmThroughput` 缺失造成 provider 无限 polling；
- Provider 6.23 的 S3 Control tag API 要求 LocalStack 更新；
- 新 EC2 VPC attribute 未实现；
- S3 endpoint/host 解析变化。

来源：

- [DynamoDB WarmThroughput polling loop](https://github.com/localstack/localstack/issues/13140)
- [Provider 6.23 S3 Control incompatibility](https://github.com/hashicorp/terraform-provider-aws/issues/45292)
- [LocalStack missing EC2 VPC attribute](https://github.com/localstack/localstack/issues/7046)
- [tflocal S3 host resolution issue](https://github.com/localstack/localstack/issues/7692)

### Ultimate 版推荐策略

```text
main CI:
  pinned Terraform
  pinned AWS provider
  pinned LocalStack image digest

compatibility CI:
  candidate Terraform/provider
  candidate LocalStack
  representative fixtures
```

### 测试失败时先分类

| 现象 | 可能原因 |
|---|---|
| AWS 成功、LocalStack 失败 | 模拟器缺 API/字段 |
| LocalStack 成功、AWS 失败 | IAM/SCP/quota/eventual consistency |
| 两边都失败 | Terraform/module/provider 配置 |
| 仅新 provider 失败 | provider regression 或 emulator lag |
| 仅 CI 架构失败 | lockfile/platform hash/cache |

### 不要立刻改生产 HCL 迎合模拟器

优先：

1. 查 provider changelog；
2. 查 LocalStack Issue；
3. 最小复现；
4. 升/降测试版本；
5. 使用 endpoint/skip 参数的明确兼容配置；
6. 必要时隔离 LocalStack-only workaround；
7. 在真实 AWS sandbox 校验。

---

## 32. LocalStack 兼容实验套件

### Fixture A：S3

覆盖：

- bucket；
- versioning；
- encryption；
- lifecycle；
- tags；
- notification；
- object；
- import；
- destroy。

检测：

- S3 Control API；
- notification whole-document ownership；
- tag read；
- object ETag/hash；
- no-op。

### Fixture B：DynamoDB

覆盖：

- billing mode；
- GSI；
- TTL；
- stream；
- PITR；
- table class；
- WarmThroughput 相关 response；
- replica。

检测：

- create waiter；
- ACTIVE 后 no-op；
- AWS 托管默认；
- provider polling。

### Fixture C：Lambda + IAM

覆盖：

- role；
- policy attachment；
- VPC config；
- code hash；
- event source；
- permissions；
- version/alias。

真实 AWS 专门测：

- IAM propagation；
- ENI cleanup；
- CloudWatch；
- concurrency/quota。

### Fixture D：Multi-service

```text
S3 → EventBridge/SNS/SQS → Lambda → DynamoDB
```

测试：

- 一个 notification owner；
- retry/DLQ；
- encryption；
- event shape；
- update/destroy；
- partial failure。

---

## 33. OpenTofu State Encryption：有价值，但 key loss 也是 state loss

OpenTofu 的 state encryption RFC 讨论了：

- 全量加密；
- 部分字段加密；
- key provider；
- KMS/Vault 等外部系统；
- plan 加密；
- key rotation；
- fallback method；
- CI 读取 state/plan 的兼容。

来源：

- [OpenTofu RFC: Client Side State Encryption](https://github.com/opentofu/opentofu/issues/874)
- [Proposal: State Encryption](https://github.com/opentofu/opentofu/issues/297)
- [Experimental encryption feedback request](https://github.com/orgs/opentofu/discussions/3416)

### 高论

加密解决：

- state object 被窃取后的明文暴露；
- backend/operator 不应直接看敏感值的部分场景。

不解决：

- apply 身份本身有权解密；
- provider/API 已能读取云资源；
- replay 旧 state；
- state corruption；
- 错误代码删除资源；
- key 丢失；
- CI 日志泄露。

### 关键风险

```text
encrypted state + lost key = unrecoverable state
```

因此需要：

- key backup；
- rotation/fallback 演练；
- break-glass；
- plan/state 同步策略；
- CI/runtime key injection；
- 禁止 key 与 state 放同一保护域；
- 从旧 state version 恢复时仍能取得旧 key。

---

## 34. Terraform-as-a-Service：不可信 HCL 近似运行不可信代码

OpenTofu Discussion 讨论“代表第三方运行不可信 `.tf`”。

Maintainer 明确警告：

- OpenTofu 并非按这个 threat model 设计；
- providers 是任意代码执行的重要入口；
- modules、provisioners、file functions、network 都是攻击面；
- 当前实现细节不是永远保证；
- 每个 release 都需重新审计。

来源：

- [Security implications of tofu-as-a-service](https://github.com/orgs/opentofu/discussions/2657)

### 对内部 Terraform 平台的启示

即使不是公开 SaaS，PR author 也可能通过：

- 新 provider；
- 外部 module；
- `local-exec`；
- `external` data source；
- file/template functions；
- malicious plan output；
- metadata service/network；
- provider install source；

突破 runner 权限。

### Runner sandbox

```text
ephemeral worker
  ├─ provider allowlist/mirror
  ├─ no arbitrary module source
  ├─ egress allowlist
  ├─ no host Docker socket
  ├─ OIDC short-lived AWS role
  ├─ read-only checkout where possible
  ├─ no unrelated repository secrets
  └─ destroy after run
```

PR approval不能替代 runtime isolation。

---

## 35. OpenTofu / Terraform 差异必须进入架构记录

当前值得记录的差异方向：

- backend configuration；
- provider `for_each`；
- state/plan encryption；
- OCI modules/providers；
- registry/provider signing；
- language和 module source能力。

相关 GitHub：

- [OpenTofu backend variables](https://github.com/opentofu/opentofu/issues/388)
- [Provider loop discussion](https://github.com/orgs/opentofu/discussions/2392)
- [State encryption RFC](https://github.com/opentofu/opentofu/issues/874)
- [Keyless provider signing with Sigstore](https://github.com/opentofu/opentofu/issues/307)
- [Terraform → OpenTofu migration and provider schema errors](https://github.com/orgs/opentofu/discussions/3317)

### ADR 模板

```text
Decision: Terraform or OpenTofu
Version:
Required implementation-specific features:
Provider registry assumptions:
State format/encryption:
CI binary selection:
Module compatibility:
Rollback path:
Dual-plan validation:
```

如果代码开始依赖 provider `for_each` 或 encrypted state，就不能再说“两者随便换”。

---

## 36. AFT、Terragrunt、Terraform 的职责不要混淆

```text
Control Tower / AFT
  负责账号 vending 和 customization pipeline

Terraform/OpenTofu
  负责单个 root 内的 graph、plan、state

Terragrunt
  负责生成配置、unit DAG、批量运行

CI/orchestrator
  负责 git diff、审批、artifact、并发、恢复、跨 state 顺序

AWS
  提供最终 API、eventual consistency、quota 和 replace semantics
```

任何一层都不能自动补齐其他层：

- Terraform state lock 不会锁其他 state；
- Terragrunt DAG 不会让 unknown output 变 known；
- AFT 不自动提供所有 plan/approval/drift 需求；
- LocalStack 不会复制真实 IAM propagation；
- AWS API whole-document replace 不会因为 module 拆分而变成增量。

---

## 37. 高信号故障分类表

| 错误 | 第一判断 | 不应立即做 |
|---|---|---|
| `Provider produced inconsistent final plan` | provider schema/动态值/unknown | 反复 apply |
| `Missing Resource Identity After Update` | 远端可能已成功 | 假设 AWS 未变化 |
| `ResourceAlreadyExists` after failed apply | orphan 或 state 未写回 | 删除远端 |
| 永久 diff | canonicalization/default/双 owner | 直接 ignore |
| `Invalid principal` right after IAM create | eventual consistency | 永久加超长 sleep |
| saved plan stale | state/上下文已变化 | 强行复用旧 plan |
| provider state decode unsupported | upgrade/downgrade schema | 只降 provider |
| EKS access entry 409 | 已存在/自动创建 owner | console 随手删除 |
| EKS ASG replacement | module address/launch template变化 | 自动 merge |
| Terragrunt dependency no outputs during destroy | 已删除上游 state | 盲目 run-all |
| LocalStack infinite polling | response schema/provider matrix | 改生产资源语义 |
| S3 notification flip-flop | whole-document API、多 owner | 加 depends_on |

---

## 38. 把 GitHub Issue 变成测试，而不是收藏夹

### 每发现一个 provider Issue，生成一个 fixture

目录示例：

```text
tests/regressions/
  aws-provider-19583-default-tags/
  aws-provider-23951-s3-notification-owner/
  aws-provider-29828-lambda-iam-consistency/
  aws-provider-44366-resource-identity/
  localstack-13140-dynamodb-warm-throughput/
  eks-module-2911-stable-caller/
```

每个 fixture 记录：

```yaml
issue: https://github.com/...
affected_versions:
fixed_versions:
environment:
expected:
cannot_prove:
owner:
remove_after:
```

### Test lifecycle

```text
init
validate
plan
apply
plan(no-op)
update
plan(no-op)
import
destroy
second destroy/no-op
```

### 对 LocalStack/真实 AWS 分层

LocalStack：

- graph；
- module interface；
- provider endpoint；
- create/update/destroy；
- event flow；
- regression speed。

真实 AWS：

- IAM；
- quota；
- region support；
- eventual consistency；
- EKS；
- KMS；
- service-managed versions；
- billing。

---

## 39. Provider 升级实验

### 两个版本同时跑

```text
baseline lane
  current Terraform
  current provider lock
  current module

candidate lane
  candidate provider
  same Terraform/module/config
```

比较：

- plan action count；
- replacement；
- state schema；
- resource identity；
- API calls；
- warning/deprecation；
- LocalStack；
-真实 AWS sandbox。

### 再分别升级 module

```text
provider candidate + old module
provider candidate + new module
```

这样能区分：

- provider regression；
- module breaking change；
- 组合兼容性。

### 通过条件

```text
□ create 成功
□ second plan no-op
□ update 收敛
□ import no unexpected changes
□ destroy 无 orphan
□ state 可由当前生产 runner读取
□ rollback rehearsal 完成
□ LocalStack failure 已分类
□真实 AWS 代表性服务通过
```

---

## 40. EKS 升级实验

### 环境

尽量复制生产：

- authentication mode；
- managed/self-managed nodes；
- Karpenter；
- Fargate；
- private endpoint；
- CNI；
- access entries；
- addon；
- runner location。

### 测试

1. 当前版本 no-op；
2. module major only；
3. provider major only；
4. EKS minor only；
5. node AMI only；
6. access model migration；
7. worker role 保留；
8. node drain；
9. PDB；
10. rollback。

### 自动禁止

对以下 plan action设置 hard gate：

```text
aws_eks_cluster: delete/replace
aws_autoscaling_group: delete/replace
aws_eks_node_group: delete/replace
aws_kms_key: delete/replace
aws_eks_access_entry: unexpected principal replacement
```

---

## 41. 多账号实验

### Terraform 路线

```text
accounts.json
  → CI matrix
  → one root per account
  → stable OIDC deploy role
  → one state per failure domain
```

测试：

- wrong account 必须失败；
- `allowed_account_ids`；
- prod role 不能从 PR 获取；
- account A failure 不阻塞 B；
- state prefix 最小权限；
- account removal 的 destroy/ref 语义。

### OpenTofu provider for_each 路线

测试：

- provider keys 在 plan 前 known；
- 无 module self-reference；
- provider instance address 稳定；
- 删除 account key 的 destroy 行为；
- state/plan 加密；
- Terraform fallback 已明确不可用或有限。

---

## 42. 代码评审问题

每个 Terraform × AWS PR 至少问：

1. 这个 AWS API 是增量还是整份 replace？
2. 远端对象是否只有一个 owner？
3. 是否与 console、Crossplane、GitOps、AWS service 双重管理？
4. 失败后远端可能已成功吗？
5. provider 是否改变 resource identity/state schema？
6. rollback 是否包含 state object version？
7. 是否升级 CLI、provider、module、service 中多个层？
8. lockfile 是否更新且跨平台完整？
9. backend/workspace 是否与 plan 相同？
10. saved plan 是否单次、短 TTL、敏感？
11. 是否使用动态 timestamp/current caller 进入 desired state？
12. 永久 diff 的根因是什么？
13. `ignore_changes` 是在声明外部 owner，还是在消音？
14. 是否依赖 IAM eventual consistency？
15. transient retry 是否有界？
16. EKS caller identity 是否稳定？
17. module major 是否有 address migration？
18. VPC 列表 index 变化会替换什么？
19. Terragrunt 删除 unit 时旧 ref 如何 destroy？
20. partial destroy 能否 resume？
21. LocalStack/provider 版本矩阵是什么？
22. 哪些性质必须真实 AWS 证明？
23. secret 是否可使用 write-only/ephemeral？
24. runner 是否执行不可信 provider/module/provisioner？
25. drift 与 approval 是否真的存在，还是工具名称让人误以为存在？

---

## 43. 推荐生产基线

### 版本

- Terraform/OpenTofu exact version；
- `.terraform.lock.hcl` 入库；
- root 限制 provider；
- module exact version/tag；
- LocalStack image digest；
- Terragrunt exact version；
- runner image digest。

### State

- S3 versioning；
- native lockfile；
- narrow IAM；
- encryption；
- audit；
- recovery drill；
- state object 与 lockfile 权限分离；
- upgrade 前备份。

### Identity

- OIDC；
- stable deployment role；
- plan/apply role 分离；
- account allowlist；
- no long-lived keys；
- EKS access entries 显式；
- break-glass。

### CI

- fmt/validate/lint；
- policy/security；
- provider/module diff；
- LocalStack fixtures；
-真实 AWS sandbox；
- plan action gates；
- approval；
- single-use plan artifact；
- concurrency；
- drift schedule。

### Failure

- partial success runbook；
- orphan detection/import；
- state restore；
- provider rollback limitations；
- unit destroy/resume；
- EKS node/access rollback；
- API owner map。

---

## 44. 精选阅读：Terraform Core

### Backend 与 state

- [Using variables in Terraform backend config](https://github.com/hashicorp/terraform/issues/13022)  
  看点：初始化阶段为何不能直接复用普通 variables；partial configuration 是现实方案。

- [Separate backend configurations for each workspace](https://github.com/hashicorp/terraform/issues/16627)  
  看点：workspace 名称和 backend isolation 不是一回事。

- [Stronger durability of remote state](https://github.com/hashicorp/terraform/issues/19488)  
  看点：state 写回失败、write-ahead/recovery 与 orphan resource。

- [S3 native lockfile missing permission behavior](https://github.com/hashicorp/terraform/issues/36407)  
  看点：新 locking 路径的权限边缘。

- [S3 lockfile timeout after external deletion](https://github.com/hashicorp/terraform/issues/37324)  
  看点：不要人工删除 lockfile 代替正确解锁。

### Provider graph

- [Instantiating multiple providers with a loop](https://github.com/hashicorp/terraform/issues/19932)  
  看点：动态 AWS account provider 是长期需求。

- [Dynamically generated provider configurations](https://github.com/hashicorp/terraform/issues/16967)  
  看点：provider address 必须足够早地确定。

- [Provider configs passed to module for_each](https://github.com/hashicorp/terraform/issues/24476)  
  看点：module/provider graph 限制。

- [Dynamic provider assignment](https://github.com/hashicorp/terraform/issues/25244)  
  看点：provider meta-argument 不是普通字符串。

### State、Plan 与 Secrets

- [Storing sensitive values in state](https://github.com/hashicorp/terraform/issues/516)  
  看点：秘密不持久化为何需要 provider/schema 共同支持。

- [Ability not to store specific managed resource attributes](https://github.com/hashicorp/terraform/issues/30469)  
  看点：write-only attributes 的需求轨迹和当前边界。

- [Saved plan executed repeatedly](https://github.com/hashicorp/terraform/issues/24078)  
  看点：plan artifact 必须单次使用和绑定 state。

- [Large state performance](https://github.com/hashicorp/terraform/issues/16375)  
  看点：API、graph、memory 与拆 state 的关系。

### Refactoring

- [Dynamic moved blocks](https://github.com/hashicorp/terraform/issues/33236)  
  看点：大规模 `for_each` migration 仍需要生成/审查。

- [Declarative import](https://github.com/hashicorp/terraform/issues/26364)  
  看点：import 从命令式走向配置化的背景。

- [Move resources between state files](https://github.com/hashicorp/terraform/issues/32777)  
  看点：跨 state refactor 仍是高风险操作。

---

## 45. 精选阅读：Terraform AWS Provider

### API owner

- [S3 notification resources overwrite each other](https://github.com/hashicorp/terraform-provider-aws/issues/23951)  
  看点：AWS whole-document API 决定只能有一个 owner。

- [Inline and standalone security group rules](https://github.com/hashicorp/terraform-provider-aws/issues/12580)  
  看点：两个 Terraform 表达方式同时管理一份规则会冲突。

- [S3 bucket policies overwrite each other](https://github.com/hashicorp/terraform-provider-aws/issues/6334)  
  看点：policy/configuration 的 replace semantics。

### Eventual consistency

- [Lambda IAM eventual consistency](https://github.com/hashicorp/terraform-provider-aws/issues/29828)  
  看点：显式依赖成功后服务仍可能看不到权限。

- [IAM role invalid principal immediately after creation](https://github.com/hashicorp/terraform-provider-aws/issues/8905)  
  看点：IAM propagation。

- [S3 policy and public access block race](https://github.com/hashicorp/terraform-provider-aws/issues/7628)  
  看点：API order 与 provider retry。

- [RAM share accepter cannot find invitation](https://github.com/hashicorp/terraform-provider-aws/issues/13494)  
  看点：跨账号传播。

### Diff 与 provider schema

- [default_tags update every plan](https://github.com/hashicorp/terraform-provider-aws/issues/18311)
- [default_tags identical to resource tags](https://github.com/hashicorp/terraform-provider-aws/issues/19204)
- [tags_all inconsistent final plan](https://github.com/hashicorp/terraform-provider-aws/issues/19583)
- [IAM policy document ordering](https://github.com/hashicorp/terraform-provider-aws/issues/11801)
- [ElastiCache 6.x managed minor version](https://github.com/hashicorp/terraform-provider-aws/issues/15625)
- [DynamoDB default KMS replica recreation](https://github.com/hashicorp/terraform-provider-aws/issues/29636)

### Upgrade 与 partial success

- [AWS Provider v6.0.0](https://github.com/hashicorp/terraform-provider-aws/issues/41101)  
  看点：beta、multi-region、breaking-change 计划。

- [Provider v6 downgrade/state incompatibility](https://github.com/hashicorp/terraform-provider-aws/issues/43178)  
  看点：provider downgrade 不受支持。

- [Provider 6.14 Missing Resource Identity after successful update](https://github.com/hashicorp/terraform-provider-aws/issues/44366)  
  看点：远端成功但 apply 失败。

- [AWS Provider v4 planned changes](https://github.com/hashicorp/terraform-provider-aws/issues/20433)  
  看点：S3 refactor 的升级影响和后续回调。

- [S3 v4 refactor community discussion](https://github.com/hashicorp/terraform-provider-aws/issues/23106)  
  看点：upgrade guide 不能替代真实 fixture。

---

## 46. 精选阅读：EKS / VPC Modules

### EKS module

- [Is EKS module complexity too high?](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/635)  
  看点：公共 module maintainer 对抽象税的直接反思。

- [v17 → v18 upgrade broke deployments](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1744)  
  看点：旧 node role、节点失联和 downtime。

- [v17 → v18 planned ASG destruction](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2100)  
  看点：resource address change。

- [v21 proposed changes](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3394)  
  看点：aws-auth 移除、AWS Provider v6、addon 和 Pod Identity。

- [cluster creator entry changes with caller](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2911)  
  看点：Terraform runner identity 必须稳定。

- [Existing EKS access entry causes conflict](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2968)  
  看点：盘点/import，不要重复 owner。

- [Karpenter v1 API mismatch](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3244)  
  看点：EKS/Karpenter/module 版本矩阵。

### VPC module

- [Adding an AZ destroys NAT gateways](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/106)
- [NAT replacement order](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/999)
- [Too many NAT gateways](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/117)
- [Explicit subnet CIDR to AZ mapping](https://github.com/terraform-aws-modules/terraform-aws-vpc/issues/1284)

---

## 47. 精选阅读：Terragrunt Discussions / Issues

### CI 与 saved plan

- [run --all / stacks in CI](https://github.com/gruntwork-io/terragrunt/discussions/4588)  
  看点：unknown dependency output 与 deterministic plan 的根本冲突。

- [Plan and apply on different machines](https://github.com/gruntwork-io/terragrunt/issues/494)  
  看点：cache、plugins、路径和 artifact 上下文。

- [Apply planfile still initializes dependencies](https://github.com/gruntwork-io/terragrunt/discussions/5357)  
  看点：saved plan 不一定让 dependency init 消失。

- [Deleted unit in CI](https://github.com/gruntwork-io/terragrunt/discussions/3639)  
  看点：必须在 target/pre-merge ref destroy。

### DAG、性能和恢复

- [15x dependency performance regression](https://github.com/gruntwork-io/terragrunt/issues/2806)  
  看点：递归读取 inputs 的代价。

- [Stacks worth the effort?](https://github.com/gruntwork-io/terragrunt/discussions/5998)  
  看点：新抽象是否只是再加一层。

- [Partial run-all destroy cannot simply resume](https://github.com/gruntwork-io/terragrunt/issues/3183)  
  看点：上游 outputs 在 destroy 后不存在。

- [Shared dependency lock contention](https://github.com/gruntwork-io/terragrunt/issues/6002)  
  看点：复杂 CI graph 的并发 race。

- [Multi-platform provider cache/hashes](https://github.com/gruntwork-io/terragrunt/discussions/6136)  
  看点：cache 与完整 lockfile 是两个问题。

- [Provider and state depending on IAM module output](https://github.com/gruntwork-io/terragrunt/discussions/5273)  
  看点：bootstrap identity 与 downstream assume-role 的边界。

---

## 48. 精选阅读：AFT / LocalStack / OpenTofu

### AFT

- [Add plan and approval stages](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/481)
- [Continuous drift detection](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/628)
- [SSM custom field synchronization race](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/531)
- [Tracking account creation and customization](https://github.com/aws-ia/terraform-aws-control_tower_account_factory/issues/429)

### LocalStack

- [DynamoDB WarmThroughput polling loop](https://github.com/localstack/localstack/issues/13140)
- [Provider 6.23 S3 Control compatibility](https://github.com/hashicorp/terraform-provider-aws/issues/45292)
- [EC2 VPC attribute not implemented](https://github.com/localstack/localstack/issues/7046)
- [Terraform from Docker to LocalStack](https://github.com/localstack/localstack/issues/718)

### OpenTofu

- [Client-side state encryption RFC](https://github.com/opentofu/opentofu/issues/874)
- [Backend variables](https://github.com/opentofu/opentofu/issues/388)
- [Keyless provider signing](https://github.com/opentofu/opentofu/issues/307)
- [Security of tofu-as-a-service](https://github.com/orgs/opentofu/discussions/2657)
- [Migrating AWS CDK resources](https://github.com/orgs/opentofu/discussions/2909)
- [Provider for_each cycle](https://github.com/orgs/opentofu/discussions/2760)
- [Encryption feedback](https://github.com/orgs/opentofu/discussions/3416)

---

## 49. 最后的 GitHub 社区能量

GitHub Issues / Discussions 最值得吸收的不是：

> “这个工具有 bug。”

而是：

> “失败发生在哪个阶段？远端是否已经改变？state 是否写回？API 的所有权是什么？哪个版本修复？回滚是否真的可行？”

把本轮材料压成一句话：

> Terraform × AWS 的成熟工程，不是避免所有 bug，而是让 API owner、版本矩阵、state transition、失败恢复和真实测试都显式可见。

真正的生产能力包括：

- 知道哪些 AWS 配置只能有一个 owner；
- 知道 `depends_on` 与 eventual consistency 的边界；
- 知道 apply 失败时远端可能已经成功；
- 知道 provider downgrade 不一定能读升级后的 state；
- 知道 module major 可能是资源迁移项目；
- 知道 Terragrunt DAG 不能创造未知 output；
- 知道 LocalStack/provider 要作为一个版本矩阵；
- 知道 AFT、Terraform、Terragrunt 和 CI 各自只解决一层问题；
- 把重要 GitHub Issue 变成自己的 regression fixture；
- 在社区再次踩坑以前，让 CI 先踩一遍。

