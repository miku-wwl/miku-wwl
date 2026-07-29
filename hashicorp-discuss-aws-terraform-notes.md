# HashiCorp Discuss：AWS × Terraform 实践精华

整理日期：2026-07-27  
筛选范围：HashiCorp Discuss 中与 AWS、Terraform 工程实践相关的讨论  
筛选标准：优先保留能改变架构决策、能转化为实验、包含 Terraform 核心维护者解释的内容；排除纯安装问题、无结论问答和认证推广。

## 结论先行

最值得带回项目里的，不是某个目录模板，而是以下六条：

1. Provider 配置属于 root module；共享模块只声明依赖和必要的 `configuration_aliases`。
2. AWS account/region 与 Provider configuration 静态绑定，不能在 `for_each` 中动态选择 Provider。
3. 当账号数增加时，不要把“大量 alias + 大量模块实例”无限塞进同一 Terraform graph；应按故障域、权限域和变更域拆 state。
4. Terraform state lock 只协调同一个 state；多个相互依赖的 state 之间的并发必须由 CI/CD 编排层解决。
5. 公共模块如果已经适合使用，就直接调用；只有在收窄接口、固化组织规则或增加业务能力时才值得包一层。
6. LocalStack 适合验证 Terraform graph、模块接口、跨服务连接、state 行为和幂等性，但不能证明真实 AWS 的配额、权限边界、最终一致性、区域差异和服务实现细节。

---

## 1. Provider 配置只放在 root module

来源：[Terraform reusable modules and provider declarations best practices](https://discuss.hashicorp.com/t/terraform-reusable-modules-and-provider-declarations-best-practices/39808)

### 高论

共享模块不应该包含 `provider "aws" { ... }`。模块只通过 `required_providers` 声明它需要 AWS Provider；凭证、region、assume role、endpoint 等具体配置由 root module 注入。

只有模块本身必须同时操作两个 AWS 身份或区域，例如跨账号 VPC peering，才应声明：

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      configuration_aliases = [
        aws.requester,
        aws.accepter,
      ]
    }
  }
}
```

调用者再明确映射：

```hcl
module "peering" {
  source = "./modules/vpc-peering"

  providers = {
    aws.requester = aws.account_a
    aws.accepter  = aws.account_b
  }
}
```

### 为什么重要

Provider 是运行时环境的一部分，不是模块业务逻辑的一部分。把 Provider 配置藏进模块，会让凭证、region 和 endpoint 难以替换，也会破坏 `for_each`、`depends_on` 等模块级能力。

### LocalStack 实验

- 使用相同 child module，分别从两个 root module 调用。
- 一个 root 使用真实风格的 AWS Provider 配置，另一个通过 `tflocal` 注入 LocalStack endpoint。
- child module 保持完全不变。
- 再构造双 Provider 的跨账号模块，给两个 LocalStack account ID 注入不同 access key，验证资源归属。

---

## 2. Provider alias 不能动态计算

来源：

- [Is a variable provider alias possible in a module call?](https://discuss.hashicorp.com/t/is-a-variable-provider-alias-possible-in-a-module-call/22371)
- [Error: Invalid provider configuration reference](https://discuss.hashicorp.com/t/error-invalid-provider-configuration-reference/41189)

### 高论

Terraform 在构建 dependency graph 时，就必须确定资源与 Provider configuration 的关联。因此不能写出类似：

```hcl
providers = {
  aws = aws[each.key]
}
```

也不能把 Provider alias 当字符串或普通变量动态选择。

这不是 HCL 语法的小缺陷，而是 Terraform graph 模型的边界。若要向 20 个账号部署同一模块，通常需要：

- 静态声明 20 个 module call；或
- 为每个账号启动独立 root configuration / pipeline job；或
- 用外层生成器生成 root modules，但生成后的 Terraform graph 仍然是静态的。

### LocalStack 实验

- 配置 3 个模拟账号，每个账号一个 Provider alias。
- 先尝试动态 Provider，记录 Terraform 初始化或验证错误。
- 再改为 3 个独立 root configurations，由 PowerShell/Make/CI matrix 并行调用。
- 比较单一大 graph 与多 root graph 的 plan 时间、内存、失败隔离和 state 粒度。

---

## 3. 多账号不是 alias 数量竞赛

来源：

- [Multiple provider aliases calling a single module](https://discuss.hashicorp.com/t/multiple-provider-aliases-calling-a-single-module/57155)
- [Structuring Terraform for Multi-Account AWS](https://discuss.hashicorp.com/t/structuring-terraform-for-multi-account-aws-modular-resource-management-with-cross-account-access/77271)

### 高论

一个配置中创建十几个 AWS Provider 实例，会增加 Provider 进程的内存和 graph 复杂度。降低 `-parallelism` 不一定有帮助，因为它限制资源操作并发，而不是消除 Provider 实例。

更稳健的拆分依据是：

- AWS account / 权限边界；
- 独立部署与回滚需求；
- 变更频率；
- blast radius；
- 团队所有权；
- 跨 state 依赖数量。

不要为了“一个 apply 管全部”而牺牲故障隔离。

### LocalStack 实验

建立两套方案：

1. 单 root、8 个 Provider alias、8 个相同模块实例；
2. 8 个 root、8 个独立 state，由外层脚本并发执行。

记录：

- `terraform plan` 峰值内存；
- 初始化耗时；
- 一个账号故意失败时，其余账号是否能继续；
- destroy 的 blast radius；
- state 文件大小和可读性。

---

## 4. State lock 不会保护多个 state 之间的依赖

来源：

- [Terraform remote state: lock for multiple states](https://discuss.hashicorp.com/t/terraform-remote-state-lock-for-multiple-states/23714)
- [S3-native state locking discussion](https://discuss.hashicorp.com/t/feature-request-terraform-state-locking-in-aws-with-s3-strong-consistency-no-dynamodb/18456)

### 高论

Terraform 的锁是每个 configuration/workspace/state 独立的。`terraform_remote_state` 读取另一个 state 时不会取得对方的锁；Terraform 假定 backend 能原子写入完整 snapshot。

因此：

- `network` state 正在修改时，`database` state 仍可能读取旧或新的 network 输出；
- Terraform 没有内建“把 network、EKS、database 三个 state 一起锁住”的机制；
- 跨 state 顺序和并发控制必须放在 CI/CD job queue、DAG 或部署编排层。

另外，S3 强一致性不等于不需要锁。强一致性保证对象读取结果，不会阻止两个 Terraform 进程同时基于旧 state 创建资源。Terraform 1.10 起可使用 S3 原生 lockfile；是否采用仍应结合 Terraform 版本、backend 兼容性和现有运维流程。

### LocalStack 实验

- 使用 LocalStack S3 建立 `network` 与 `app` 两个 state。
- `app` 通过 `terraform_remote_state` 读取 subnet 输出。
- 人为让两个 apply 并发，观察跨 state 无共同锁的行为。
- 在外层增加互斥 job group，再比较结果。
- 验证当前 Terraform 版本与 LocalStack 对 `use_lockfile = true` 的兼容性。

---

## 5. 公共模块是否要包一层

来源：[Best practice when use Terraform public modules](https://discuss.hashicorp.com/t/best-practice-when-use-terraform-public-modules/45965)

### 高论

如果 wrapper module 只是把公共模块所有变量和输出原样透传，它没有提供抽象，只增加升级成本。

值得 wrapper 的情况：

- 固化组织级 tags、日志、加密、备份和网络规则；
- 禁止不安全选项；
- 把几十个底层参数收窄为少数业务参数；
- 组合额外资源，例如为 RDS 同时创建监控和 secrets integration；
- 对上层提供稳定接口，隔离公共模块升级。

### LocalStack 实验

选一个 AWS VPC、S3 或 serverless 公共模块：

- 先直接调用；
- 再写一个只透传变量的 wrapper，测量其维护成本；
- 最后写一个 opinionated wrapper：强制 encryption、versioning、tags 和 access policy；
- 用 `terraform test` 或 apply 后的 AWS API 查询断言组织规则确实生效。

---

## 6. Module 边界应围绕能力，而不是 AWS 资源清单

来源：

- [Best Practices for Terraform Modules](https://discuss.hashicorp.com/t/best-practices-for-terraform-modules/33499)
- [What is Terraform custom Module best practises?](https://discuss.hashicorp.com/t/what-is-terraform-custom-module-best-practises/48788)

### 高论

模块边界没有唯一目录模板。关键问题是：

- 谁维护它；
- 谁独立发布它；
- 是否需要原子升级；
- 它是否代表一个有意义的能力；
- 跨 Provider 的生命周期是否真的耦合。

例如 RDS module 是否同时管理 Datadog dashboard 和 Vault secret engine，取决于这三者是否必须作为一个产品能力一起发布和回滚。模块越“全家桶”，用户越简单，但 Provider 耦合、测试成本和升级 blast radius 越大。

同一团队维护、经常跨模块一起修改时，monorepo 通常更简单；独立团队和独立发布节奏则适合独立仓库。

### LocalStack 实验

用一个 serverless capability 做对照：

- 细粒度版本：Lambda、API Gateway、DynamoDB、IAM 分成四个模块；
- 能力版本：一个 `order_api` module 组合全部资源；
- 比较输入数量、跨模块 outputs、plan 可读性、替换 blast radius 和测试夹具复杂度。

---

## 7. LocalStack 不是“另一个 Provider”

来源：

- [To use aws or docker as provider when running LocalStack](https://discuss.hashicorp.com/t/to-use-aws-or-docker-as-provider-when-running-localstack-from-container/4349)
- [Switching between local and remote states](https://discuss.hashicorp.com/t/switching-between-local-and-remote-states/7374)
- [LocalStack Terraform documentation](https://docs.localstack.cloud/aws/connecting/infrastructure-as-code/terraform/)

### 高论

LocalStack 仍然使用 `hashicorp/aws` Provider，只是把 AWS service endpoints 指向本地网关。不要把它理解为 Docker Provider，也不要在共享模块里硬编码 LocalStack endpoint。

当前最省维护的做法通常是：

```powershell
tflocal init
tflocal plan
tflocal apply
```

`tflocal` 通过 Terraform override 临时注入 endpoint，使生产 Terraform 配置保持不变。手工 Provider 配置适合需要精确控制 endpoint、多账号身份或诊断网络问题的场景。

需要特别注意：

- Terraform backend 与 AWS Provider 是两个独立配置面；
- Terraform 在容器内运行时，`localhost` 指向 Terraform 容器自身，不是 LocalStack 容器；
- S3 virtual-host addressing 与普通服务 endpoint 不同；
- 当前 AWS Provider 还可能调用 S3 Control API，因此手工 endpoint 配置要覆盖实际使用到的服务。

---

## 第二轮高强度搜索：Terraform × AWS 深层工程规律

本轮继续搜索了 dependency graph、`for_each`、环境隔离、导入重构、漂移、安全资源所有权、Provider 升级、测试、EKS、Lambda artifact、IAM 最终一致性和失败恢复。下面只保留对设计有长期影响的结论。

## 8. 优先通过数据流表达依赖，慎用 module-level `depends_on`

来源：

- [Terraform plan changes based on existence of depends_on in module](https://discuss.hashicorp.com/t/terraform-plan-changes-based-on-existence-of-depends-on-in-module/33169)
- [Dependent module and data source](https://discuss.hashicorp.com/t/dependent-module-and-data-source/18486)
- [How can I achieve dependency between modules?](https://discuss.hashicorp.com/t/how-can-i-achieve-dependency-between-modules/4331)

### 高论

表达式引用比 `depends_on` 携带更多信息：

```hcl
module "app" {
  source = "./modules/app"

  subnet_ids       = module.network.private_subnet_ids
  database_endpoint = module.database.endpoint
}
```

Terraform 能从具体值的流动中构建精确 dependency edges。相反：

```hcl
module "app" {
  source     = "./modules/app"
  depends_on = [module.network]
}
```

表示 `network` 中任何动作都可能影响 `app` 中任何对象。后果包括：

- data sources 被推迟到 apply 才读取；
- 更多属性变成 `(known after apply)`；
- 下游资源可能出现保守的替换计划；
- plan 噪音增加；
- 实际并不相关的对象也被串行化。

module-level `depends_on` 不是“更安全的依赖”，而是信息更少、范围更大的依赖。

### 实践原则

- 首选传递真正需要的 ID、ARN、endpoint 和结构化对象。
- 输出值可以通过自己的 `depends_on` 表达“这个值可用之前还必须完成哪些隐含动作”。
- 只在依赖无法通过数据流表达时使用显式 `depends_on`。
- 如果两个模块总是需要大范围相互排序，重新检查模块边界是否切错。

### LocalStack 实验

建立两个版本：

1. `module.app depends_on = [module.network]`；
2. app 只引用 `module.network.private_subnet_ids`。

在 network 中只修改一个不影响 app 的 tag，比较两个 plan 中 data source 延迟、unknown values 和下游 diff。

---

## 9. `for_each` 的 key 是持久身份，不是循环下标

来源：

- [Invalid for_each argument with data source result](https://discuss.hashicorp.com/t/invalid-for-each-argument-with-data-source-result/17242)
- [Terraform for_each known only after apply](https://discuss.hashicorp.com/t/terraform-for-each-will-be-known-only-after-apply-workaround-not-working-for-multiple-entries/51576)
- [Create subnets from list with object](https://discuss.hashicorp.com/t/create-subnets-from-list-with-object/34397)

### 高论

Terraform 必须在 plan 阶段知道 `for_each` 的完整 key 集合，因为 key 决定 state address：

```text
aws_subnet.private["prod:ap-southeast-2a"]
```

map 的 value 可以是 apply-time unknown，但 key 必须已知：

```hcl
resource "aws_route_table_association" "private" {
  for_each = {
    "az-a" = aws_subnet.private_a.id
    "az-b" = aws_subnet.private_b.id
  }

  subnet_id = each.value
}
```

避免把远端生成的 ARN、ID、随机值作为 key，也不要把 unknown 字符串转换成 set 后用于 `for_each`。Terraform 无法提前判断 set 是否会因重复值而收缩。

### AWS 建模建议

- 子网：`"${environment}:${availability_zone}:${tier}"`；
- AWS account：内部稳定 account key，不直接用可能变化的显示名称；
- IAM binding：`"${role_key}:${policy_key}"`；
- Route：`"${route_table_key}:${destination_key}"`。

key 一旦进入 state，就应像数据库主键一样对待。改名应配合 `moved` block。

### LocalStack 实验

- 用 list + `count` 创建三个 subnet，删除中间元素，观察 index shift。
- 改用稳定 map key + `for_each`，重复同一操作。
- 再把未知 resource ID 用作 key，复现 plan-time error。

---

## 10. Workspaces 是 state 选择器，不是安全隔离边界

来源：

- [Recommended Approaches for Staging with Terraform](https://discuss.hashicorp.com/t/recommended-approach-es-for-staging-with-terraform/13177)
- [Workspaces for Dev/Test/Prod](https://discuss.hashicorp.com/t/workspaces-for-dev-test-prod-or-other-different-environments/13776)
- [How to clone an existing AWS environment](https://discuss.hashicorp.com/t/how-to-clone-and-existing-aws-environment-new-to-terraform/50949)

### 高论

CLI workspaces 解决的是“同一配置目录对应多个 state”。它们不会自动提供：

- 不同 backend credentials；
- 不同 IAM 权限；
- 不同审批流程；
- 不同 Provider 版本；
- 不同组织策略；
- 防止操作者选错 workspace 的物理隔离。

dev/staging/prod 若属于不同 AWS account、不同权限域或不同 blast radius，通常应使用独立 root configuration 和独立 backend 配置。是否共用模块代码是另一个问题。

### 实践模型

```text
modules/
  service/
live/
  dev/service/
  staging/service/
  prod/service/
```

三个 root 可以调用同一版本化模块，但分别拥有 backend、Provider 身份和审批管道。

### LocalStack 实验

模拟 dev/prod 两个账号：

- 方案 A：同一 root 使用两个 workspace；
- 方案 B：两个 root、两个 backend、两个 account key。

故意在错误 workspace/目录执行命令，比较防错能力。

---

## 11. State 拆分依据是生命周期和 blast radius，不是文件美观

来源：

- [Separate workspaces per project or separate state files](https://discuss.hashicorp.com/t/separate-workspaces-envs-per-project-or-separate-state-files-or/37974)
- [Terraform remote state: lock for multiple states](https://discuss.hashicorp.com/t/terraform-remote-state-lock-for-multiple-states/23714)
- [Migrating to a Multi-repo approach](https://discuss.hashicorp.com/t/migrating-to-a-multi-repo-approach/55061)

### 高论

state 太大：

- plan/refresh 慢；
- 一个 Provider/API 故障阻塞全部资源；
- 权限过宽；
- apply blast radius 大；
- 团队容易互相阻塞。

state 太碎：

- outputs 和 remote-state coupling 增加；
- 跨 state 顺序需外部编排；
- 版本兼容和 rollout 更复杂；
- 每次变更需要协调多个 apply。

合理拆分信号：

- 资源是否需要一起原子变更；
- 是否由同一团队管理；
- 是否使用同一凭证和审批；
- 变更频率是否相近；
- 故障是否应互相隔离；
- 输出接口能否保持小而稳定。

“VPC、EKS、RDS 分三个 state”不是天然正确；必须根据组织和变更模型证明它。

---

## 12. Refactor 必须显式迁移 state identity

来源：

- [How to change current Terraform structure?](https://discuss.hashicorp.com/t/how-to-change-current-terraform-structure/38781)
- [Migrate easily from count to for_each](https://discuss.hashicorp.com/t/migrate-easily-from-count-to-for-each/62548)
- [Request for testing: removed block](https://discuss.hashicorp.com/t/request-for-testing-removed-block/60511)

### 高论

把资源移入 module、重命名、从 `count` 改为 `for_each`，改变的是 Terraform address，不一定改变 AWS 对象。若不告诉 Terraform 地址映射，它会把旧地址视为删除、新地址视为创建。

```hcl
moved {
  from = aws_rds_cluster.main
  to   = module.database.aws_rds_cluster.main
}
```

从 `count` 到 `for_each` 必须为每个实例建立确定映射：

```hcl
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["az-a"]
}
```

`removed` 与 `import` 可用于把对象管理权从一个 configuration 转移到另一个，但这属于迁移事务，应分阶段执行和验证。

### 安全迁移流程

1. 保存当前 plan 和 state version；
2. 添加 `moved` blocks；
3. 要求 plan 显示 address move 而不是 destroy/create；
4. apply 后再次运行无变更 plan；
5. 保留 moved blocks 一段兼容期，让尚未升级的调用者也能迁移；
6. 跨独立 package 的 move 需要更谨慎，因为模块包之间不能随意声明移动关系。

### LocalStack 实验

创建 VPC + subnet 后完成三次无损重构：

- root resource → child module；
- `count` → `for_each`；
- 一个 state → 另一个 state。

每一步都以“底层资源 ID 不变”为验收。

---

## 13. Import 不是发现工具，而是建立对象与 address 的绑定

来源：

- [Only create resources that don't already exist](https://discuss.hashicorp.com/t/only-create-resources-that-dont-already-exist/60123)
- [Terraform plan after import showing add/destroy](https://discuss.hashicorp.com/t/terraform-plan-after-import-showing-add-destroy-for-an-existing-resource/54395)
- [Import resource using count meta-argument](https://discuss.hashicorp.com/t/is-it-possible-to-use-import-config-to-import-a-resource-that-uses-the-count-meta-argument/60258)

### 高论

正确顺序是：

1. 先写 resource configuration；
2. 确定精确 address，包括 module path 与 `for_each`/`count` key；
3. import AWS object；
4. plan；
5. 调整配置，直到 plan 收敛到预期。

Terraform 不会因为 AWS 中“已经有同名资源”就自动采用它，也不会在 plan 阶段可靠预测所有服务的名称冲突。远端唯一性校验往往只会在 apply 调用 API 时失败。

导入到错误 address 的常见表现是：一个 address 计划 destroy，另一个 address 计划 create。

### LocalStack 实验

- 用 `awslocal` 手工创建 bucket/security group；
- 写 Terraform resource；
- 分别演示未 import、import 到错误 key、import 到正确 key；
- 验证最终无变化 plan。

---

## 14. 安全组和 IAM 的“追加式”与“权威式”管理要主动选择

来源：

- [Use of aws_security_group_rule vs inline definition](https://discuss.hashicorp.com/t/use-of-aws-security-group-rule-vs-inline-definition-what-do-you-prefer/45234)
- [Terraform Plan Doesn't Catch Manual Resource Changes](https://discuss.hashicorp.com/t/terraform-plan-doesnt-catch-manual-resource-changes/45899)
- [Safely migrate inline security group rules](https://discuss.hashicorp.com/t/how-can-i-safely-migrate-from-inline-security-group-rules-to-seperate-security-group-rule-resources/2867)

### 高论

Terraform 只能管理 state 中有 address 的对象。独立 rule/attachment resources 往往具有“追加式”语义：Terraform 管理自己创建的规则，但 AWS 中额外手工添加的规则可能不会被删除。

这意味着：

- 可扩展性更好，多个模块可以向同一 SG/IAM role 添加绑定；
- 但 Terraform 不一定能发现和清除未受管的额外权限；
- 安全审计不能只依赖 `terraform plan`。

内联或权威式资源更容易表达“完整集合必须等于配置”，但会增加共享所有权冲突，且迁移时可能暂时删除规则。

### 工程决策

- 高安全边界：倾向单一权威 owner，并增加 AWS Config/CloudTrail/自定义审计。
- 平台扩展点：允许追加式资源，但明确哪些规则可由谁添加。
- 不要混用两种模式管理同一个远端集合。
- 迁移前检查 Provider 文档中资源是否 authoritative、additive 或存在冲突警告。

### LocalStack 实验

- Terraform 创建 SG rule；
- 使用 `awslocal` 手工添加一条开放规则；
- 运行 plan，观察是否能检测；
- 使用 AWS API 查询断言完整规则集合；
- 为 CI 增加“禁止未声明 0.0.0.0/0”行为测试。

---

## 15. `ignore_changes` 是共享所有权契约，不是消除 plan 噪音的胶带

来源：[Using lifecycle ignore_changes for attributes managed outside Terraform](https://discuss.hashicorp.com/t/using-lifecycle-ignore-changes-for-attributes-managed-outside-of-terraform/37493)

### 高论

`ignore_changes` 的合理用途是明确声明某些属性在创建后由另一个控制器管理，例如：

- Auto Scaling desired capacity 由 autoscaler 管理；
- 某些运行时 tags 由组织工具管理；
- ECS desired count 由部署系统管理。

它的风险是把真实 drift 隐藏掉。添加之前必须回答：

- 谁是这个属性的真实 owner？
- 谁验证它仍符合安全和合规要求？
- 若外部 owner 消失，Terraform 是否会永久忽略错误状态？

不要因为 Provider bug 或不收敛 diff 就大面积 `ignore_changes = all`。先找出不收敛的根因。

---

## 16. 动态 AMI data source 会把“查最新版”变成隐式部署

来源：

- [Terraform plan wants to recreate all infrastructure](https://discuss.hashicorp.com/t/terraform-plan-wants-to-recreate-all-of-my-infrastructure/53984)
- [Terraform returning single AMI](https://discuss.hashicorp.com/t/terraform-returning-single-ami/35625)

### 高论

```hcl
data "aws_ami" "latest" {
  most_recent = true
  # filters...
}
```

每次 plan 都会重新选择满足条件的最新 AMI。即使 Git 没有代码变化，新 AMI 发布也可能使 EC2、launch template、node group 等产生替换或 rollout。

这不是 drift；这是配置明确要求“当前最新值”。

### 生产策略

- CI discovery job 查找候选 AMI；
- 把 AMI ID 写入版本化变量、artifact manifest 或参数；
- 通过 PR 明确升级；
- plan 中能看到“某个确定 ID → 某个确定 ID”；
- rollout、健康检查和回滚都绑定到该版本。

LocalStack 可验证数据流和替换 graph，但无法证明真实 AMI 可启动、驱动兼容或 userdata 正确。AMI fidelity test 必须进入真实 AWS sandbox。

---

## 17. Terraform plan 不保证提前发现所有 AWS API 冲突

来源：

- [Need Support with Terraform AWS EKS](https://discuss.hashicorp.com/t/need-support-with-terraform-aws-eks/36933)
- [Data sources forcing replacement even though nothing has changed](https://discuss.hashicorp.com/t/data-sources-forcing-replacement-even-though-nothing-has-changed/46452)

### 高论

Terraform plan 主要基于：

- configuration；
- state；
- Provider refresh/read；
- Provider 提供的 plan logic。

它不会普遍扫描 AWS，寻找所有未来可能发生的名称冲突、配额不足、最终一致性、权限缺失或控制面条件。某个 EKS node group 与手工资源重名，可能运行很久后才由 AWS create API 返回错误。

因此生产管道还需要：

- API 级 preflight；
- 配额检查；
- 独立行为测试；
- 禁止/检测手工资源；
- 超时和失败恢复设计；
- 小 blast-radius rollout。

“plan 成功”只表示 Terraform 能构造一个自洽的执行计划，不表示远端一定接受全部动作。

---

## 18. IAM 最终一致性不是 dependency graph 问题

来源：[Eventual consistency for aws_iam_role_policy](https://discuss.hashicorp.com/t/eventual-consistency-for-aws-iam-role-policy/8424)

### 高论

`depends_on` 只能保证 API 调用顺序：

1. 创建 IAM policy；
2. policy API 返回成功；
3. 创建依赖资源。

它不能保证 IAM policy 已传播到所有 AWS 控制面。若后续服务立即验证权限，仍可能失败。

应对层次：

1. 优先依赖 AWS Provider 内建 retry/waiter；
2. 向 Provider 报告可复现的 eventual-consistency gap；
3. 在流水线层做有界重试；
4. 最后才考虑显式 wait，而且要记录原因和退出条件。

LocalStack 通常比真实 AWS 更快、更一致，因此这类问题可能完全复现不了。需要在真实 sandbox 做时序测试。

---

## 19. Provider 与 Terraform 版本升级是一次基础设施变更

来源：

- [Guidance for configuring Provider and Terraform Versions](https://discuss.hashicorp.com/t/guidance-for-configuring-provider-and-terraform-versions/55935)
- [What does terraform init -upgrade do?](https://discuss.hashicorp.com/t/what-does-terraform-init-upgrade-do/32638)
- [Terraform 0.14 dependency lock file](https://discuss.hashicorp.com/t/terraform-0-14-the-dependency-lock-file/15696)

### 高论

模块中的 version constraint 表示“兼容范围”；root configuration 的 `.terraform.lock.hcl` 表示“当前选择的精确 Provider 版本”。两者职责不同。

推荐：

- shared module 声明最低兼容版本，必要时加已知不兼容上限；
- root 提交 `.terraform.lock.hcl`；
- `terraform init -upgrade` 的 lockfile diff 进入 PR；
- 升级后先在 LocalStack 和 sandbox 跑 plan/apply/destroy/reapply；
- apply 可能升级 state 中的 Provider schema，使旧 Provider 不再适合回退；
- 先升级 Terraform CLI，再单独升级 AWS Provider，避免同时改变两个变量；
- 多平台 CI 使用 `terraform providers lock -platform=...` 预先记录所需平台 checksum。

不要在日常 pipeline 中无审查地执行 `terraform init -upgrade`。

### LocalStack 实验

对同一 fixture 建立 Provider N 与 N+1 的测试矩阵：

- clean create；
- no-op plan；
- in-place update；
- destroy；
- 从旧版本创建后用新版本 refresh/apply；
- plan JSON 差异。

Terraform 单次配置不能同时加载同一 Provider 的多个版本，因此矩阵应由外层独立 jobs 执行。

---

## 20. 测试分层：mock、LocalStack、真实 AWS 各自证明不同事情

来源：

- [Best practices of Terraform staging testing](https://discuss.hashicorp.com/t/best-practices-of-terraform-staging-testing/6762)
- [Terraform test using mock but want data source to generate data](https://discuss.hashicorp.com/t/terraform-test-using-mock-but-want-data-source-to-generate-data/72503)
- [How to test data block conditions](https://discuss.hashicorp.com/t/how-to-test-data-block-conditions/73279)

### 高论

mock Provider 能证明：

- 变量 validation；
- locals 和表达式逻辑；
- 输出契约；
- 某些 graph 结构。

mock 不能证明：

- data source filter 真能从 AWS 返回正确对象；
- Provider schema/API 转换行为；
- 服务之间的实际连接；
- IAM 权限有效。

LocalStack 能进一步证明：

- AWS Provider 真实调用路径；
- Terraform apply/destroy/idempotency；
- 多服务集成；
- API 查询后的行为断言；
- 多账号和本地 state/backend 实验。

真实 AWS sandbox 才能证明：

- IAM/SCP/permission boundaries；
- service quotas；
- eventual consistency；
- region-specific behavior；
- AMI、EKS、RDS 等真实控制面；
- 性能与成本。

### 推荐测试金字塔

```text
大量：validate / lint / policy / mock tests
中量：LocalStack integration tests
少量：真实 AWS fidelity tests
极少：staging / production rollout checks
```

测试 module 时，建立最小 root fixture，为它创建最少依赖。应同时验证：

- 从零 create；
- 修改已有资源；
- no-op plan；
- destroy；
- destroy 后重新 create。

---

## 21. Terraform 测试必须验证远端行为，不能只验证 output

来源：

- [terraform test and data source limitations](https://discuss.hashicorp.com/t/terraform-test-and-data-source-limitations/38220)
- [Best Practices for Terraform Modules](https://discuss.hashicorp.com/t/best-practices-for-terraform-modules/33499)

### 高论

模块自己输出的值来自同一份配置和 state，容易形成“自己证明自己”的测试。例如模块输出 `encryption_enabled = true`，只断言该 output 没有证明远端 bucket 真启用了加密。

更强的测试使用第二条观察路径：

- AWS SDK/CLI 读取资源；
- 发起真实 HTTP/API 请求；
- 写入 S3 后检查 SQS event；
- 调用 API Gateway 后读取 DynamoDB；
- 以无权限身份验证访问被拒绝。

LocalStack Ultimate 的价值主要在这里：让大量行为断言可以低成本、高频率运行。

---

## 22. EKS 与 Kubernetes Provider 常常需要分阶段管理

来源：

- [EKS/GKE/AKS to Kubernetes Resources: Provider Dependency](https://discuss.hashicorp.com/t/eks-gke-aks-kubernetes-resources-provider-dependency/18217)
- [Looking for recommendations on Terraform EKS deployment](https://discuss.hashicorp.com/t/looking-for-recommendations-on-our-current-terraform-deployment/53080)

### 高论

Provider configuration 通常在 plan 阶段就必须可用。若同一次 apply：

1. 创建 EKS cluster；
2. 根据 cluster endpoint/token 配置 Kubernetes/Helm Provider；
3. 创建 Kubernetes resources；

就可能遇到 bootstrap 边界：规划 Kubernetes resources 时，cluster 尚不存在。

常见拆分：

- state/run A：VPC、EKS control plane、node groups、基础 IAM；
- state/run B：Kubernetes/Helm add-ons；
- state/run C：应用 namespace 和应用级 AWS dependencies。

拆分也改善权限：创建 EKS 的云权限与部署 Kubernetes workloads 的集群权限不必相同。

LocalStack 对 EKS 控制面的 fidelity 有边界。可用它验证 Terraform 结构、IAM、VPC 和部分 API，但 Kubernetes Provider bootstrap 应在可用的真实/临时 cluster 上验证。

---

## 23. 构建 Lambda artifact 与部署基础设施是两个不同阶段

来源：[Lambda source_code_hash](https://discuss.hashicorp.com/t/lambda-source-code-hash/30407)

### 高论

`source_code_hash` 需要代表实际部署 artifact 的内容。S3 key 只是对象地址，不是内容摘要；如果 key 不变而内容改变，Terraform 未必知道应更新 Lambda。

更稳健的模式：

1. 构建系统生成不可变 zip/image；
2. 计算 digest；
3. 上传到带版本或 digest 的 S3 key/ECR tag；
4. Terraform 接收 artifact version/digest；
5. Terraform 负责部署，不负责隐式构建。

这样基础设施 plan 对应一个确定 artifact，而不是在执行机上临时压缩后得到不可复现结果。

### LocalStack 实验

- 使用相同 S3 key 覆盖不同 zip，观察 plan；
- 改为 versioned key + digest；
- 验证 Lambda 更新触发和回滚路径。

---

## 24. `-target` 与 `-replace` 是外科工具，不是日常部署模型

来源：

- [-target warning seems inappropriate](https://discuss.hashicorp.com/t/target-warning-seems-innapropriate/4091)
- [Warning: failed to decode resource from state](https://discuss.hashicorp.com/t/warning-failed-to-decode-resource-from-state/61907)
- [replace_triggered_by behavior after Terraform failure](https://discuss.hashicorp.com/t/replace-triggered-by-behavior-after-terraform-failure/75024)

### 高论

`-target` 会让 Terraform 只处理依赖图的一部分，可能造成：

- outputs 未更新；
- Provider state schema 未完整升级；
- 配置不能完全收敛；
- 隐藏其他必要变更。

适用场景是故障恢复、拆解循环或 Terraform 明确建议的两阶段操作。目标操作后必须运行完整 plan。

若一次 apply 中触发 A 更新、B 因 `replace_triggered_by` 计划替换，但 apply 在 B 之前失败，下一次 plan 可能因为 A 已无变化而不再触发 B。此时应审计实际状态，并使用：

```powershell
terraform apply -replace='resource.address'
```

恢复，而不是期待旧触发条件自动重现。

---

## 25. Terraform 并不承诺 `for_each` 实例按顺序执行

来源：[For_each support sequential operation?](https://discuss.hashicorp.com/t/for-each-support-sequential-operation/34680)

### 高论

同一个 resource/module 的多个 `for_each` 实例在 graph 中被视为彼此独立，因此可能并行运行。Terraform 没有通用的 `for_each_sequence`。

如果远端 API 不允许并发：

- 首先应由 Provider 实现 mutex、semaphore 或 retry；
- 可以降低全局 `-parallelism`，但会拖慢整个 graph；
- 真正有顺序语义的操作可能不适合建模为一组同构 Terraform resources；
- 需要严格业务顺序时，考虑外层工作流分阶段执行。

LocalStack 可通过故意构造并发冲突测试 pipeline 的限流和恢复逻辑。

---

## 26. Secrets 标记为 sensitive，不等于 secrets 不进入 state

来源：

- [How does Terraform detect sensitive data?](https://discuss.hashicorp.com/t/how-does-terraform-detect-sensitive-data/23339)
- [Statefile intent for aws_secretsmanager_secret](https://discuss.hashicorp.com/t/statefile-intent-for-aws-secretsmanager-secret/44612)

### 高论

`sensitive = true` 主要控制 CLI/UI 展示和表达式传播，不是 state 加密机制。只要某个 secret value 作为 managed resource argument/result 参与 Terraform 管理，它就可能写入 state。

因此：

- state backend 必须按 secret store 级别保护；
- S3 state bucket 应启用 encryption、versioning、严格 IAM、访问日志和恢复策略；
- 避免把 secret value 用作 `for_each` key，因为 key 会出现在 address 和日志中；
- 优先让 AWS 服务自行生成/管理 secret，Terraform 只管理 secret container 或引用；
- CI plan artifacts 也可能包含敏感结构，应设置保留期和访问控制。

LocalStack 可验证 secret 流程和 state 中是否出现明文，但生产安全结论必须基于真实 backend、IAM 和审计配置。

---

## 27. AWS Organizations account factory 天然是多阶段工作流

来源：

- [For_each AWS account creation](https://discuss.hashicorp.com/t/for-each-aws-account-creation/19858)
- [Avoid Destroying AWS Organization Account](https://discuss.hashicorp.com/t/avoid-destroying-aws-organization-account/27866)
- [How would you manage an AWS account for each client?](https://discuss.hashicorp.com/t/how-would-you-manage-an-aws-account-for-each-client-with-terraform/59395)

### 高论

在一个 apply 中：

1. 创建 AWS account；
2. 得到 account ID；
3. 动态构造新 Provider；
4. assume role 进入该账号；
5. 部署 baseline；

会碰到 Provider 静态绑定、账号创建异步、Organizations 限流和 IAM 传播等多重边界。

更自然的 account factory：

1. Organizations state 创建/登记账号；
2. 等待账号和 bootstrap role 可用；
3. 外层编排系统为该 account ID 创建独立 root job；
4. baseline state 部署 CloudTrail、Config、IAM、网络等；
5. workload states 后续接管。

AWS account 也不是普通可随意 destroy 的临时资源。配置中应明确 offboarding、关闭和 state removal 流程，不要让删除 map entry 自动代表“销毁账号”。

LocalStack 可以模拟多账号 Provider 拓扑，但真实 account creation lifecycle 必须在 AWS Organizations sandbox/测试 OU 验证。

---

## 28. 判断 Terraform 设计是否健康：看它能否稳定收敛

来源：

- [Refresh state after apply](https://discuss.hashicorp.com/t/refresh-state-after-apply-recommendations-best-practices/34354)
- [Data sources forcing replacement](https://discuss.hashicorp.com/t/data-sources-forcing-replacement-even-though-nothing-has-changed/46452)

### 高论

健康配置的核心性质：

```text
apply 成功 → 再次 plan → No changes
```

持续不收敛通常说明：

- Provider normalization bug；
- 配置与 API 默认值不一致；
- 两个 controllers 争夺同一属性；
- data source 被粗粒度 dependency 推迟；
- 动态 discovery 每次选择不同对象；
- 资源模型选错，例如 EC2 中混淆 `security_groups` 与 `vpc_security_group_ids`；
- 外部系统持续改写对象。

不要把“再 apply 一次就好了”当成稳定运行模式。应把 no-op plan 纳入每个模块的测试验收。

---

## 29. 用 `allowed_account_ids` 把“部署错账号”变成硬错误

来源：[How can I verify which AWS account Terraform is targeting during plan?](https://discuss.hashicorp.com/t/how-can-i-verify-which-aws-account-cdktf-terraform-is-targeting-during-plan/76738)

### 高论

依赖当前 shell profile、默认 credential chain 或操作者记忆来选择账号，是多账号 Terraform 中最危险的隐式输入之一。AWS Provider 可限制它允许操作的账号：

```hcl
provider "aws" {
  region = var.region

  allowed_account_ids = [var.expected_account_id]

  assume_role {
    role_arn = "arn:aws:iam::${var.expected_account_id}:role/terraform"
  }
}
```

推荐同时输出/检查：

```hcl
data "aws_caller_identity" "current" {}

check "correct_account" {
  assert {
    condition     = data.aws_caller_identity.current.account_id == var.expected_account_id
    error_message = "AWS account does not match this root configuration."
  }
}
```

`allowed_account_ids` 是 Provider 层硬防线，`check` 则提供更清晰的诊断。两者不能替代 prod/dev 独立凭证和 backend 隔离，但可以显著降低误操作风险。

### LocalStack 实验

- 为两个模拟 account ID 建立两个 root；
- 故意交换 access key；
- 要求 plan 在任何 resource action 之前失败；
- 将该测试加入所有 root fixture。

---

## 30. 所有 Provider 都有 alias 时，会出现隐式空 default Provider

来源：[Default Provider when all providers have alias](https://discuss.hashicorp.com/t/default-provider-when-all-providers-have-alias/56727)

### 高论

如果配置中只有：

```hcl
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-2"
}

provider "aws" {
  alias  = "global"
  region = "us-east-1"
}
```

Terraform 会把它理解为还存在：

```hcl
provider "aws" {}
```

任何没有显式 Provider 映射的 AWS resource/module 可能落到这个隐式空配置，继而使用环境中的默认 region/credentials，或直接报缺少配置。最危险的情况是它“恰好能工作”，但指向了错误账号或区域。

### 防护

- root 中提供一个明确 default AWS Provider，作为最常用且受 `allowed_account_ids` 限制的配置；或
- 要求每个 resource/module 显式映射 Provider，并用静态检查禁止裸 AWS resources；
- 对 `us-east-1` 全局资源（CloudFront、ACM 等）使用命名清晰的 alias；
- Provider alias 改名后执行受控 refresh/plan，确认 state 中 Provider references 已迁移。

---

## 31. S3 backend 正从 DynamoDB locking 迁移到原生 lockfile

来源：

- [Feature request and arrival of S3-native state locking](https://discuss.hashicorp.com/t/feature-request-terraform-state-locking-in-aws-with-s3-strong-consistency-no-dynamodb/18456)
- [Deprecation of dynamodb_table in Terraform S3 Backend](https://discuss.hashicorp.com/t/deprecation-of-dynamodb-table-in-terraform-s3-backend/77060)

### 高论

现代 Terraform 已支持：

```hcl
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "prod/service/terraform.tfstate"
    region       = "ap-southeast-2"
    use_lockfile = true
  }
}
```

社区讨论显示旧 `dynamodb_table` locking 已进入弃用迁移阶段。迁移时不要只删除 DynamoDB 配置然后假设安全：

1. 确认团队和 CI 使用的 Terraform 版本都支持 S3 native lockfile；
2. 验证 backend IAM 包含 state 与 lock object 所需权限；
3. 停止并发 applies；
4. 切换 backend 配置并执行 `terraform init -reconfigure`；
5. 用两个并发 plan/apply 验证锁竞争；
6. 保留 S3 versioning 和恢复流程；
7. 最后再退役 DynamoDB table。

锁解决的是同一 state 的并发写入，不会解决跨 state 顺序、错误账号、坏 plan 或人工误删。

### LocalStack 实验

- 先测试旧 DynamoDB lock；
- 再测试 `use_lockfile = true`；
- 启动两个并发 apply，确认只允许一个 writer；
- 模拟异常中断，演练 lock 诊断与安全解锁；
- 记录你当前 Terraform 与 LocalStack 版本的实际兼容结果。

---

## 社区能量压缩：15 条应形成肌肉记忆的规则

1. Terraform address 是持久身份；重构地址必须迁移 state。
2. `for_each` key 必须稳定、可读、plan-time known。
3. 数据流引用优先于 `depends_on`。
4. Provider 配置属于 root，child module 只声明能力。
5. account/region Provider 关联是静态的；大规模多账号要靠外层编排。
6. state 边界按权限、生命周期、团队和 blast radius 划分。
7. workspaces 不等于环境安全隔离。
8. plan 成功不代表 AWS apply 一定成功。
9. no-op plan 是 Terraform 设计的基本健康指标。
10. mock、LocalStack、真实 AWS 分别证明不同层次，不能互相替代。
11. state、plan artifact 和 logs 都按潜在 secret 处理。
12. Provider 升级、module 升级和 artifact 升级都应作为显式、可审查的变更。
13. 每个 AWS root 都应限制 `allowed_account_ids`，防止 credential chain 指向错误账号。
14. 所有 Provider 都有 alias 时要警惕隐式空 default Provider。
15. backend locking 方案升级也必须做并发和恢复演练。

---

## 建议你的 LocalStack Ultimate 优先实验路线

### P0：Provider 注入与模块纯度

目标：同一 child module 无修改地跑 LocalStack 与 AWS 风格 root。

验收：

- child module 没有 Provider block；
- `tflocal apply` 成功；
- 普通 `terraform validate` 成功；
- LocalStack endpoint 没有泄漏到模块接口。

### P1：多账号部署拓扑

目标：比较“多 alias 单 state”和“每账号独立 state”。

验收：

- 模拟至少 4 个账号；
- 记录内存、耗时、失败隔离；
- 明确最终选择的 state 边界。

### P2：跨 state 并发

目标：证明 state lock 不能替代 pipeline orchestration。

验收：

- 复现并发读取/写入；
- 在外层增加依赖队列；
- 输出一次失败记录与修复后结果。

### P3：组织级安全 wrapper

目标：验证 wrapper module 是否真正提供组织价值。

验收：

- 默认加密；
- 默认 tags；
- 默认阻止公共访问；
- 使用 AWS API 查询而不仅是 Terraform output 进行断言。

### P4：服务级集成测试

目标：测试真实行为，而不是只检查 `apply` 成功。

推荐链路：

- API Gateway → Lambda → DynamoDB；
- S3 event → SQS/Lambda；
- SNS → SQS；
- IAM policy enforcement。

LocalStack 官方文档支持通过 Terraform init hooks、Testcontainers、Cloud Pods 和 IAM Policy Stream 构建这类测试环境：

- [Terraform init hooks / Testcontainers](https://docs.localstack.cloud/aws/tutorials/using-terraform-with-testcontainers-and-localstack/)
- [Cloud Pods collaborative environments](https://docs.localstack.cloud/aws/tutorials/cloud-pods-collaborative-debugging/)
- [IAM Policy Stream](https://docs.localstack.cloud/aws/tutorials/iam-policy-stream/)

---

## 完整吸收路线：从会写 Terraform 到会运营 Terraform

### 第一阶段：语言与 state identity

- `for_each`、map key、unknown values；
- `moved`、`import`、`removed`；
- Provider inheritance 与 aliases；
- no-op plan。

验收：能完成无损模块重构，不依赖手工修改 state。

### 第二阶段：AWS 模块与所有权

- VPC/subnet 稳定 key；
- SG/IAM additive vs authoritative；
- organization tags；
- immutable artifact 与 AMI pinning；
- secret/state 风险。

验收：每个模块都有明确远端资源 owner 和行为测试。

### 第三阶段：多账号与多 state

- root-per-account/environment；
- assume role；
- pipeline matrix；
- state concurrency；
- account factory。

验收：一个账号失败不阻塞其他账号，且 prod 凭证与 dev 物理隔离。

### 第四阶段：测试和升级

- mock tests；
- LocalStack integration；
- AWS sandbox fidelity；
- Provider version matrix；
- state/schema upgrade；
- failure recovery。

验收：Provider 升级 PR 自动产生 create/update/no-op/destroy 测试结果。

### 第五阶段：平台工程

- opinionated modules；
- policy as code；
- deployment DAG；
- drift detection；
- audit trail；
- golden paths。

验收：应用团队只提供业务输入，平台层仍能强制安全和合规。

---

## 不应从 LocalStack 测试中得出的结论

LocalStack 测试通过，不代表以下事项已经被证明：

- 真实 AWS IAM/SCP/permission boundary 完全正确；
- 服务配额和 throttling 行为正确；
- IAM、Organizations、Route53 等服务的最终一致性时序正确；
- 跨区域故障行为正确；
- AWS 托管服务内部升级、维护窗口和控制平面异常已覆盖；
- 成本、容量、性能和真实网络延迟符合生产要求。

推荐分层：

1. `terraform validate`、lint、静态安全检查；
2. LocalStack 中执行快速集成测试；
3. 临时真实 AWS sandbox account 做少量 fidelity tests；
4. 生产前 plan、policy 和人工审批。
