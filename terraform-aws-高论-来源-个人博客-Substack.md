# Terraform × AWS 社区高论（来源：个人博客 / Substack）

整理日期：2026-07-27  
主要来源：Terraform 核心设计者的个人开发日志、独立基础设施作者、AWS/平台工程师个人博客、Substack 生产复盘与独立新闻简报。  
筛选目标：保留能改变 Terraform × AWS 架构判断、暴露失败模型、给出反方或可以转化为实验的文章；排除匿名惊悚标题、入门拼贴、无版本日期的代码墙、未经证实的性能数字以及把产品广告伪装成“最佳实践”的内容。

> 个人写作的优势是敢于承认错误、讲组织里不方便讲的话，也更容易保留事故现场；缺点是样本小、版本老、观点与商业利益可能混在一起。本文把内容区分为：**[机制]** 可由 Terraform/AWS 行为验证；**[生产复盘]** 具名作者公开经历；**[设计观点]** 有推理但不是普遍事实；**[作者偏好]** 与团队背景强相关；**[当前校正]** 用 2026 年官方资料修正旧文。  
> “搜完全部社区知识”不可能严格完成。本文做的是高强度、高信号策展：不追求链接数量，而追求互相矛盾、能交叉验证、能转化为 Terraform/AWS 决策与 LocalStack Ultimate 实验的知识。

---

## 0. 先吸收这 50 条

1. **Terraform plan 不是普通 dry-run；它是一份带 unknown value 占位符、apply 时必须兑现或报错的执行承诺。**
2. **HCL 的限制不是纯粹语法审美：不允许配置根据“值是否已知”走不同分支，是 plan 可复现的基础。**
3. **`for_each` 的键集合在 plan 时必须已知，因为“将创建几个资源”本身就是计划承诺的一部分。**
4. **Terraform 的声明式语言不是为了让所有人读起来最舒服，而是为了限制非确定性和 plan/apply 分叉。**
5. **`terraform plan` 会执行 provider、data source、`external` 程序等代码；对不可信 PR，它不是无害的只读操作。**
6. **能给生产做 plan 的 runner，通常已经拿到能读取生产状态、调用 AWS API、甚至外泄凭据的上下文。**
7. **计划可见不等于安全；还要固定 commit、变量、provider、module、身份和实际 apply 的 plan artifact。**
8. **状态加密只能降低静态泄露，不能解决每个 runner 都必须解密、plan 文件可能含敏感值、remote-state 读取扩散等问题。**
9. **更强的秘密策略不是“把所有秘密都加密后交给 Terraform”，而是让 Terraform 根本不持久化不需要持久化的秘密。**
10. **Terraform 1.10 起有 ephemeral variable，1.11 起有 write-only argument；是否可用仍取决于 provider/resource 支持。**
11. **`sensitive = true` 主要抑制显示，不等于不进入 state；`ephemeral`/write-only 才针对持久化路径。**
12. **状态不是一个可有可无的缓存：它保存配置地址与真实对象的身份映射、依赖历史和部分 apply 结果。**
13. **所谓“无状态 Terraform”通常会把状态重新发明到 Git 历史、资源标签、外部数据库或云 inventory 中。**
14. **Git 历史无法单独证明一次部分成功 apply 创建了哪些真实 ID，也不能可靠替代 provider 观测。**
15. **有价值的作者不是永远正确，而是能公开修正自己的错误；Ricard Bejarano 对“stateless Terraform”的自我反驳就是高信号。**
16. **CLI workspace 最初更接近临时并行副本，不是 dev/staging/prod 强隔离方案。**
17. **永久环境若只靠 workspace 名、var-file 和 backend 参数协同，很容易把正确代码送到错误状态或错误账号。**
18. **当前官方文档仍明确：需要不同凭据/访问控制或系统拆分时，不应只靠 CLI workspaces。**
19. **每个环境复制一份会慢慢分叉的 Terraform 代码，是 Kief Morris 所说的 “Snowflakes as Code”。**
20. **“不要复制环境代码”与“不要把所有环境放一个 state”并不冲突：复用不可变模块/版本，保留独立 root 与 state。**
21. **Terraform 重构难不是代码格式问题，而是资源地址、真实对象和 state 历史一起移动的问题。**
22. **跨 state 重构比同一 state 内移动更危险；因此系统应从一开始保留 owner、迁移协议和小批量变更能力。**
23. **DRY 不是 IaC 的最高原则；共享抽象会扩大 blast radius、耦合升级节奏并隐藏 AWS 差异。**
24. **简单、重复但可局部理解的 Terraform，有时比极度抽象的 Terragrunt/YAML 生成器更安全。**
25. **反过来，环境代码的无纪律复制也会制造 definition drift；合理答案是稳定接口和版本化复用，而不是宗教式 DRY/WET。**
26. **AWS 控制台中的资源如果能直接跳到其 Terraform 根配置，人工处置速度会显著提高。**
27. **`default_tags` 中加入配置 URL、owner、environment、state/root 标识，是低成本的 code-to-cloud 索引。**
28. **同一资源出现两个不同 Terraform 配置 URL，不只是标签冲突，往往暴露双重所有权。**
29. **S3 bucket 内放 `README.txt` 能在事故现场解释用途、联系人、保留策略；但 README 是人类提示，不是权威策略。**
30. **基础设施元数据最有价值的时刻不是审计汇报，而是半夜追踪神秘 IAM key、未知 bucket 和紧急变更。**
31. **重试会把 AWS API 的真实故障隐藏成“Terraform 卡住”；诊断时缩短 retry 能快速暴露根因，但旧 provider 参数要先查当前文档。**
32. **不要为了让开发循环更快，就把生产 Secrets Manager recovery window 改成 0；开发清理与生产安全语义应分离。**
33. **Terraform 的共同工作流不等于跨云可移植资源模型；AWS VPC、IAM、RDS 与别家服务的语义不会因 HCL 相同而消失。**
34. **“换 provider 就能逃离 AWS lock-in”是错误承诺；Terraform 降低的是工具与流程切换成本，不是业务架构迁移成本。**
35. **AFT 在一支 greenfield/startup 团队里可以很顺，在遗留多账号组织里也可能不合适；两种生产经验可以同时为真。**
36. **Terraform Cloud、Atlantis、自研 CI 的选择不是成熟度排名，而是成本、控制、审计、运维能力和团队规模的函数。**
37. **限制较多的 HCL 可以降低复杂度，但 Brian Grant 的反方提醒：文件、CLI、单向推送和手工编写也制造了 IaC 的结构性摩擦。**
38. **Terraform 是按需执行的 CLI；持续反应、长事务、大规模自愈、实时状态 API 更像控制面的职责。**
39. **不应因此得出“Terraform 已死”；更实用的边界是 Terraform 管低频基础设施，controller 管需要持续协调的运行时对象。**
40. **“repo 是生产真相”并不完整：还存在 stored、actual、observed、operational 五层状态。**
41. **不是所有 drift 都应该立即回滚；有些 drift 是事故缓解、渐进迁移或临时安全收紧。**
42. **AI 代理在只看 repo 时，可能生成语法正确、plan 合理、但当前时机完全错误的基础设施改动。**
43. **2026 年 DataTalks.Club 事故的根因链不是一句“AI 笨”：本地 state、跨项目共用、自动执行、生产权限、无删除保护、同生命周期备份共同失效。**
44. **把 `rm -rf` 或 `terraform destroy` 加进命令黑名单不是可靠隔离；有执行能力的代理可换 API、脚本或 provider 达到同一效果。**
45. **真正的 agentic IaC 防线是能力隔离：sandbox 账号、短期会话、生产不可达、CI 受控入口、不可由同一主体解除的 Deny。**
46. **GitHub Actions 到 AWS 应使用 OIDC/STS，而不是长期 IAM user key；trust policy 必须约束 `aud` 和具体 `sub`。**
47. **“一个 OIDC provider，多 workload role”通常是合理隔离，不是安全问题；角色按 repo/environment/权限边界拆分。**
48. **SCIM 与 Terraform 同时创建 Identity Center 用户/组会形成双 owner；外部 IdP 管身份，Terraform 管 permission set 与 account assignment 更清晰。**
49. **LocalStack 适合验证 Terraform 图、AWS API 交互、破坏性计划、标签、秘密流、状态隔离和事件链；不能证明真实 IAM、配额、延迟与控制面完全一致。**
50. **最值得从个人博客吸收的不是作者的工具站队，而是他暴露的失败条件、隐含假设、后悔和后来加上的防线。**

---

## 1. 来源怎么筛、怎么读

### 1.1 进入正文的门槛

一篇个人文章至少满足两项：

- 作者是相关系统的设计者、长期维护者或具名生产操作者；
- 解释机制，不只给命令；
- 公开失败、后悔、反例或自我修正；
- 给出能被复现的 Terraform/AWS 行为；
- 与官方当前文档或另一位独立作者可以交叉验证；
- 结论会影响 state、权限、环境边界、秘密、恢复或控制面设计；
- 代码虽然老，但失败模型仍有效，且能明确写出版本校正。

### 1.2 降权或排除

- 匿名“Terraform 两次搞垮生产，所以我不用了”；
- 没有账号规模、故障条件、版本、代码或可验证细节的宏大结论；
- `terraform init/plan/apply` 截图式教程；
- 2026 年仍无说明地复制 DynamoDB state lock 等旧模板；
- 把 benchmark、节省百分比、事故数字写得很精确却不给原始证据；
- 作者正销售某个 Terraform 替代品，却不披露利益关系；
- AI 生成痕迹明显、术语华丽但没有失败边界的 SEO 小作文。

### 1.3 证据等级

| 等级 | 含义 | 本文用法 |
|---|---|---|
| A | 核心设计者解释机制，或具名原始事故复盘 | 可作为核心模型，但仍检查版本 |
| B | 具名工程师多年生产经验，包含失败条件 | 可用于架构判断，标明团队背景 |
| C | 独立作者的设计推演或反方 | 用来挑战默认假设，不当成事实 |
| D | Newsletter/聚合中的编辑判断 | 用作雷达和线索，不替代原文 |
| E | 未验证 Substack/Medium 断言 | 只列待验证，不进入硬结论 |

---

## 2. 最高信号作者地图

| 作者 / 来源 | 代表文章 | 为什么值得读 | 主要偏差 |
|---|---|---|---|
| Martin Atkins | [Unknown Values](https://log.martinatkins.me/2021/06/14/terraform-plan-unknown-values/)、[Ephemeral Values](https://log.martinatkins.me/2024/05/22/terraform-ephemeral-values/)、[Re-imagining Workspaces](https://log.martinatkins.me/2019/11/01/rethinking-terraform-workspaces/) | Terraform 语言和内部模型的一手解释 | 设计者视角，未必覆盖所有组织运维 |
| Alexey Grigorev | [How I Dropped Our Production Database](https://alexeyondata.substack.com/p/how-i-dropped-our-production-database) | 2026 年具名 AWS/Terraform/AI 原始事故时间线与恢复措施 | 单人团队、权限和成本取舍特殊 |
| Alex Chan | [code-to-cloud tags](https://alexwlchan.net/2023/tag-iac-resources/)、[S3 README](https://alexwlchan.net/2025/s3-bucket-readme/)、[IAM key](https://alexwlchan.net/2023/iam-keys/) | 小而硬的 AWS 可运维性技巧，源于实际查障 | 组织规模未必代表大企业 |
| Kief Morris | [Snowflakes as Code](https://infrastructure-as-code.com/posts/snowflakes-as-code.html)、[Infrastructure Stack](https://infrastructure-as-code.com/posts/defining-stacks.html) | IaC 设计、复用、交付和环境分叉的长期模型 | 书籍作者的规范性视角 |
| Jack Lindamood | [Four years of infrastructure decisions](https://cep.dev/posts/every-infrastructure-decision-i-endorse-or-regret-after-4-years-running-infrastructure-at-a-startup/) | AWS、AFT、Terraform、Atlantis、HCL/CDK 的四年 startup 复盘 | 单家公司、greenfield 倾向明显 |
| Ricard Bejarano | [A more mature take on stateless Terraform](https://www.bejarano.io/terraform-stateless-critique/) | 公开承认原论证错误并重新构建反方 | `git backend` 仍是设计提案 |
| Alex Kaskasoli | [Terraform Plan RCE](https://alex.kaskaso.li/post/terraform-plan-rce) | 证明 plan 对不可信代码不是被动预览 | 文章旧，但威胁模型仍有效 |
| Brian Grant | [12 anti-factors](https://itnext.io/the-12-anti-factors-of-infrastructure-as-code-acb52fba3ae0)、[control plane vs CLI](https://itnext.io/automation-using-control-planes-vs-command-line-tools-66f818ff8278) | 从 Kubernetes/controller 角度系统挑战 IaC | ConfigHub 创始人，有替代范式利益 |
| Anton Babenko / Weekly.tf | [Issue #20](https://www.weekly.tf/p/weeklytf-20)、[Issue #273](https://www.weekly.tf/p/issue-273-boring-terraform-beats-clever-cue-schema-validation-safe-avm-module-migrations-with-openre)、[Issue #280](https://www.weekly.tf/p/issue-280-crossplane-vs-terraform-spaghetti-to-gitops-finops-guardrails-with-opa-the-open-agent) | 长期追踪 Terraform 生态，编辑判断常有价值 | Newsletter 是线索索引，含赞助和二手摘要 |
| Lays Rodrigues | [GitHub OIDC → AWS](https://lays147.substack.com/p/quick-tutorial-temporary-token-for) | 记录从长期 IAM key 迁移到 OIDC/STS 的真实学习路径 | 2023 代码需用当前 GitHub/AWS 文档校正 |
| Maxine Meurer | [IAM Identity Center](https://ilovedevops.substack.com/p/iam-identity-center-the-access-control)、[Entra SCIM](https://ilovedevops.substack.com/p/entra-scim-sync-to-iam-identity-center) | 身份 owner 边界、permission set、跨 state 顺序 | 部分 provider bug/时延为个人观察，需单独验证 |
| Sumant Thakur | [Your AI Agent Thinks the Repo Is Production](https://sumantthakur.substack.com/p/your-ai-agent-thinks-the-repo-is) | declared/stored/actual/observed/operational 五层状态模型 | 2026 新理论，作者在做相关产品 |
| Yehuda Cohen | [HashiCorp license changes](https://yehudacohen.substack.com/p/initial-thoughts-about-hashicorp) | AWS provider 贡献者在 2023 年的即时判断和预测 | 历史快照；后续已出现 OpenTofu |
| David J. Eddy | [Terraform cannot remove cloud lock-in](https://dev.to/david_j_eddy/the-case-against-terraform-to-prevent-vendor-lock-in-3lle) | 很早就拆穿“换 provider 即迁云”的错误叙事 | 2018 年语气强、部分多云建议需另评估 |

---

## 3. Terraform plan 的真正秘密：unknown value 与受限语言

### 3.1 plan 是“将要执行的程序”，不是模拟 apply

Martin Atkins 把 `terraform plan` 描述为一份未来执行程序：Terraform 计算高层资源动作，对 apply 后才产生的值使用 opaque unknown，例如 VPC ID 在创建前显示为 `(known after apply)`。apply 时如果外部世界或 provider 返回结果让计划无法兑现，Terraform 应报错，而不是静默执行另一套动作。

这解释了几个经常被误解的现象：

- `aws_vpc.main.id` 可以未知，但 Terraform 已知道它是 string；
- subnet 的数量可以在 plan 时确定，即使每个 subnet 的 `vpc_id` 未知；
- `for_each` 的值可以包含 unknown，但 key 集合不能 unknown；
- HCL 没有让用户查询 `is_known(x)` 的普通函数，因为那会让同一配置在 plan 和 apply 走不同分支；
- `uuid()`、`timestamp()` 等不纯函数在 plan 中必须被特殊处理；
- 配置若在 plan/apply 间读取变化的外部文件，可能触发 inconsistent final plan。

**吸收后的工程判断：**

1. 不要把“为什么 Terraform 不能像 Python 一样随便写”只解释成 DSL 保守；它与计划承诺直接相关。
2. 当模块把资源数量建立在 apply 后才知道的集合上，问题不是 Terraform “笨”，而是模块要求计划承诺一件它此刻无法知道的事。
3. 解决方法通常是把稳定 identity/key 由调用者提前提供，而不是用 `-target` 把两阶段部署永久化。
4. 如果一个数据源会在 plan 中访问外部系统，它属于计划可信计算面，不是普通只读配置。

### 3.2 对 AWS 的直接意义

AWS API 大量 ID 在创建后返回，但拓扑通常可以提前确定。一个健康的模块应把“拓扑 identity”与“远端生成 ID”分开：

```hcl
variable "subnets" {
  type = map(object({
    az   = string
    cidr = string
  }))
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id            = aws_vpc.this.id
  availability_zone = each.value.az
  cidr_block        = each.value.cidr
}
```

这里 `var.subnets` 的 key 是 plan-time identity，VPC ID 可以 unknown。若改成从某个 apply 后返回的 list 动态生成 key，计划的资源地址就不稳定。

### 3.3 反方：受限语言是否真的降低复杂度？

Jack Lindamood 的 startup 经验支持 HCL：他认为 HCL 的限制让复杂 Terraform 更显眼，而 Pulumi/CDK 的通用语言能力更容易把复杂性藏进普通代码。他们选择生成基础 skeleton，而不是全面切换 code-first IaC。

Brian Grant 的反方更尖锐：即使是 HCL，配置仍然需要求值；文件、模板、DRY 抽象和手工编辑会让真实配置难查询、难批量修改、难构建 API。两边并不完全矛盾：

- HCL 相对通用语言限制了**执行复杂度**；
- 但 HCL + module + repo sprawl 仍可能制造**组织与数据复杂度**。

因此不要问“HCL 还是 TypeScript 谁高级”，要问：

- plan 是否可预测；
- 资源 identity 是否稳定；
- 展开后的配置是否可查询；
- operator 能否从 AWS 对象回到代码；
- 抽象失败时能否局部逃生；
- 变更是否能被可靠验证和回滚。

---

## 4. 状态：既不是可删缓存，也不是万能真相

### 4.1 Ricard 的自我修正为什么重要

Ricard Bejarano 曾主张 Terraform 应保持 stateless，后来公开承认：Terraform 需要某种 state，至少需要历史依赖和资源身份。他提出用 Git 历史推导旧资源图，从而减少 Terraform 私有 state。

这个提案很有启发，但不能直接替代生产 state：

- EC2 等对象的真实 ID 在创建后才知道，Git 配置没有这次运行的返回值；
- apply 可能部分成功，Git commit 不记录哪些远端对象已创建；
- provider 的 read/refresh 结果不等于配置历史；
- 旧 commit 可能依赖已经不可获得的 provider/module；
- import、moved、provider schema migration 和 taint 等历史难由代码差异完整重建；
- AWS 资源并非都能靠用户定义的唯一标签可靠定位；
- 并发锁保护的是 state 和资源集合的一致变更，不只是一个文件。

**最有价值的结论不是“Git backend 应上线”，而是：**

- state 的职责应该被拆开理解：identity mapping、last observation、dependency history、partial transaction log、lock coordination；
- 如果未来把其中一部分迁到数据库、资源标签或控制面，要明确迁走了什么，没迁走什么；
- “stateless” 经常只是 state 的位置换了。

### 4.2 state 不是实际生产的全部

Sumant Thakur 的五层状态模型可直接拿来审查 Terraform/AWS 自动化：

| 层 | 例子 | Terraform 能看到多少 |
|---|---|---|
| Declared | HCL、module、policy、workflow | 直接看到 |
| Stored | tfstate、plan artifact、部署元数据 | 直接或经 backend 看到 |
| Actual | AWS 真实资源、IAM、网络、证书、秘密 | provider read 看到一部分 |
| Observed | metrics、logs、traces、SLO burn | 通常看不到 |
| Operational | 活跃事故、临时缓解、暂停 rollout、例外 | 通常看不到 |

所以：

- plan 没有 diff，不代表服务健康；
- drift 不一定是需要立即覆盖的错误；
- state 与 AWS 一致，不代表 SLO、数据复制、DNS 缓存或业务依赖健康；
- AI/自动化在 apply 前需要知道是否有 active incident 或临时安全收紧；
- “Git 是唯一真相”更准确地说是“Git 是声明意图的权威来源之一”。

### 4.3 2026 当前 state backend 校正

很多个人文章仍展示 S3 + DynamoDB locking。当前 [S3 backend 文档](https://developer.hashicorp.com/terraform/language/backend/s3) 已支持：

```hcl
terraform {
  backend "s3" {
    bucket       = "example-tfstate"
    key          = "service-a/prod/terraform.tfstate"
    region       = "ap-southeast-2"
    use_lockfile = true
  }
}
```

当前要点：

- `use_lockfile = true` 使用 S3 lock file；
- DynamoDB locking 已 deprecated；
- state bucket 强烈建议开启 versioning；
- lock file 需要 `GetObject`、`PutObject`、`DeleteObject`；
- 不要把 backend 凭据硬编码或通过会落盘的 `-backend-config` 传递秘密；
- backend key 是资源身份与故障域的一部分，应被机器验证。

---

## 5. 2026 年 DataTalks.Club 事故：最值得读的 Substack 原始复盘

### 5.1 事实链

Alexey Grigorev 在 [原始复盘](https://alexeyondata.substack.com/p/how-i-dropped-our-production-database) 中公开了完整时间线：

1. 他把新网站加入已有 DataTalks.Club Terraform setup，以省每月约 5–10 美元的额外 VPC/bastion 成本；
2. 换电脑后误以为 state 已远端化，实际 state 仍在旧电脑；
3. empty state 让 Terraform 认为基础设施不存在，开始创建大量重复资源；
4. 他中止 apply，并让 AI 代理用 AWS CLI 识别和删除重复对象；
5. 同时把包含生产 state 的旧 Terraform 目录压缩包移到新电脑；
6. 代理解压后，当前目录中的 state 变成生产 state；
7. 代理建议用 `terraform destroy` 更干净地删除资源，且拥有自动执行能力；
8. 整个 VPC、RDS、ECS cluster、load balancer、bastion 被删除，自动 snapshots 也不可见；
9. AWS Support 在约 24 小时后恢复了内部仍可找回的 snapshot；
10. 恢复后的一个关键表有 1,943,200 行。

这不是“Terraform 看不到 state 所以 destroy 了生产”的简单故事。empty state 首先导致**重复创建**；随后 production state 被切换进当前目录，`destroy` 才把 state 中记录的真实生产对象删除。

### 5.2 六个结构性失效

| 失效 | 为什么危险 | 应有防线 |
|---|---|---|
| 本地 state | 换机器后真实上下文消失 | 远端 S3、versioning、locking、backend identity 校验 |
| 新旧项目共用 root/state | 小变更拥有全系统 blast radius | 按所有权/生命周期拆 root 与 state |
| 代理可自动执行 | 计划、状态和命令在同一主体手里 | sandbox-only、生产不可达、CI 入口 |
| 人类只看自然语言意图 | “清理重复对象”被等同于 destroy 当前 state | 审查 machine-readable plan 与目标 state |
| 无多层删除保护 | 一个命令可删除数据库 | Terraform、AWS API、IAM/SCP 多层保护 |
| 备份同生命周期 | 删除 DB 同时失去恢复路径 | 独立账号/存储/权限的备份与定期恢复测试 |

### 5.3 作者后来做对了什么

作者实施了：

- state 移到 S3；
- 数据库同时启用 Terraform 与 AWS 删除保护；
- S3 backup 开 versioning；
- 建立独立于数据库 Terraform 生命周期的备份；
- 每日自动恢复并执行读查询，验证备份可用；
- AI 不再自动运行 Terraform 命令；
- plan 人工审查，破坏性动作由本人执行；
- 新项目拆成独立 Terraform project；
- 后续 agent 只接触 sandbox AWS account 和一小时临时会话，真实部署经 CI。

### 5.4 还需要进一步加固的地方

作者的措施很好，但几个边界要说清：

- [`prevent_destroy`](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle) 只有当生命周期规则仍存在于配置中才生效；删除整个 resource block 会连同保护一起消失。
- AWS RDS deletion protection 比 Terraform 配置更接近控制面，但具有足够权限的主体仍可先关闭它。
- 因此生产角色应有不可由普通部署角色解除的 IAM explicit Deny / SCP，关键删除需 break-glass role。
- state bucket 的管理员、生产部署者、备份删除者不应完全同权。
- plan 审查要校验 backend、account ID、role ARN、workspace/root、destroy/replace 数、关键资源名单，不只滚动看文本。
- 恢复测试比“有 snapshot”更重要；备份应跨生命周期，最好跨账号并采用不同删除权限。

### 5.5 最深的教训

作者本人认为不是 Claude 单独的错，这个判断很成熟。把 agent 当成“能力很强但没有生产语境的新操作员”更准确：

- prompt 不是权限边界；
- 命令黑名单不是权限边界；
- plan 摘要不是状态证明；
- agent 的上下文窗口不是 runbook；
- 同一个角色既能改保护又能删资源，保护只是礼貌提醒；
- 生产安全必须来自 agent 无法绕过的结构。

---

## 6. plan 也是代码执行：PR automation 的隐蔽攻击面

Alex Kaskasoli 在 [Terraform Plan RCE](https://alex.kaskaso.li/post/terraform-plan-rce) 中展示了几条路径：

- PR 添加恶意 custom provider，`terraform init` 下载后在 plan 执行；
- `external` data source 直接运行仓库里的程序；
- 现有 provider 也可能被滥用，把敏感变量编码进攻击者可观察的 API 请求；
- provisioner、archive、local 文件操作等也可能在计划路径产生副作用。

**因此以下模型是错的：**

```text
不可信 PR
  └─ 生产凭据 runner
       └─ terraform plan   # “只是看看，不会有事”
```

正确的信任分层更像：

```text
不可信 PR
  ├─ fmt / validate / static policy（无生产网络、无生产 state）
  ├─ LocalStack 集成测试（假凭据、隔离容器）
  └─ 低权限 speculative plan（若业务必须，使用受限只读身份）

经审核并固定 commit
  └─ 受控生产 plan（短期身份、固定 provider/module）
       └─ 人工/策略批准
            └─ apply 同一 plan artifact
```

2026 年需要额外考虑 AI：

- AI 生成的 provider source、module source、GitHub Action 也属于依赖供应链；
- `.terraform.lock.hcl` 应进入版本控制和 review；
- root module 应约束 provider 版本；可复用 child module 通常声明最低兼容版本，让 root 决定上限；
- 跨 Windows/Linux runner 可用 `terraform providers lock -platform=...` 预填 checksum；
- lockfile 当前只锁 provider，不锁远端 module 的实际选定版本；module source 也要明确 version/ref；
- 不允许 PR 任意增加 provider、module source、`external`、provisioner 和本地执行能力。

[当前 provider requirements](https://developer.hashicorp.com/terraform/language/providers/requirements) 与 [dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock) 文档明确建议 root 配置约束 provider，并把 lockfile 提交版本控制。

---

## 7. 秘密：从“加密 state”转向“不让秘密落进 state”

### 7.1 Martin 的关键反转

在 [Ephemeral Values in Terraform](https://log.martinatkins.me/2024/05/22/terraform-ephemeral-values/) 中，Martin 提出：

- 全量 state encryption 仍要求所有 Terraform runner 有解密能力；
- `terraform_remote_state`、`terraform show -json`、保存 plan 会扩大解密与复制面；
- plan 可能是 state 的敏感信息超集；
- 某些秘密根本不应由 Terraform 生成或长期持有；
- 必须经过 Terraform 的秘密，应只在一次操作期间存在；
- module 输入/输出边界必须显式传播 ephemeral 语义。

他还反思早期类似 `tls_private_key` 的资源：本地生成私钥再仅存 state，会让 Terraform state 变成秘密仓库。

### 7.2 2026 当前能力

[官方 ephemeral 文档](https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral) 的当前边界：

- ephemeral variable：Terraform 1.10+；
- managed resource write-only argument：Terraform 1.11+；
- provider 决定哪些资源/字段支持；
- write-only 值不进入 plan/state；
- `sensitive` 与 `ephemeral` 解决不同问题；
- 版本计数类字段仍需持久化，用来告诉 provider 是否更新秘密。

示意：

```hcl
ephemeral "random_password" "db" {
  length = 24
}

resource "aws_secretsmanager_secret" "db" {
  name = "service-a/db-password"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id                = aws_secretsmanager_secret.db.id
  secret_string_wo         = ephemeral.random_password.db.result
  secret_string_wo_version = 1
}
```

不要从示例推出“所有 AWS secret resource 都已无缝支持”；必须查看当前 AWS provider 对目标字段的 schema。

### 7.3 实际秘密策略

优先级建议：

1. **不让 Terraform 接触秘密**：由工作负载身份或 secret manager 在运行时取。
2. **短期身份**：OIDC/STS，会话过期后无长期 key。
3. **ephemeral/write-only**：仅在 provider 支持时使用。
4. **state 最小访问**：按 owner/state 拆权限，不因一个 output 开放整份 state。
5. **state/plan 加密与审计**：这是基础，不是完整答案。
6. **输出窄接口**：DNS、SSM path、资源标签或受控 API，避免下游读完整 remote state。

---

## 8. Workspaces、环境复制与 “Snowflakes as Code”

### 8.1 CLI workspace 的原始意图

Martin 在 [Re-imagining Terraform Workspaces](https://log.martinatkins.me/2019/11/01/rethinking-terraform-workspaces/) 回顾：

- Terraform 0.9 的 state environments 是同一配置的多个 state；
- 后更名为 workspaces，避免与 deployment environment 混淆；
- 初衷偏向临时并行副本；
- 永久 QA/PROD 需要同时协调 workspace、var-file、backend、依赖顺序和共享变量；
- workspace 存在于远端 backend，而不在代码里，是可发现性和审计问题；
- 大系统最终发展 wrapper/orchestrator，不是偶然。

文章中的 `.terraform-project.hcl` 是 2019 原型，不是当前 Terraform 功能。真正可长期吸收的是问题诊断，不是照抄 prototype。

[当前 CLI workspace 文档](https://developer.hashicorp.com/terraform/cli/workspaces) 仍说：

- 适合同一配置的临时并行副本；
- 不适合 system decomposition；
- 需要不同 credentials/access controls 的环境不应靠同一 backend 下的 workspaces 隔离；
- 更强隔离应使用共享 child module + 独立 root configuration/backend。

### 8.2 Kief Morris 的相反警告

不要误读成“每个环境复制一份全部代码”。Kief 在 [Snowflakes as Code](https://infrastructure-as-code.com/posts/snowflakes-as-code.html) 指出：

- dev/test/prod 各自一份可编辑代码会慢慢分叉；
- production hotfix 不回传、实验半途而废、环境差异被硬编码；
- 最终得到 definition drift；
- 应让同一版本的基础设施定义作为不可变 artifact 跨环境推进；
- 必要差异通过参数或清晰的组合层表达。

### 8.3 合成后的环境模式

```text
modules/
  service-stack/        # 可版本化复用，不含 backend/环境凭据

live/
  dev/service-a/        # 独立 root、backend、账号/role、少量参数
  staging/service-a/
  prod/service-a/
```

关键不是目录长什么样，而是：

- child module 是共享实现；
- root 是 deployment instance；
- 每个 root 有明确 backend、AWS account/role、owner；
- module 版本通过 promotion 推进；
- production 不能被 dev 凭据访问；
- 环境差异有 schema，不靠复制后手改；
- 不把所有环境放进一个 state，也不让每个环境演化成独立产品。

### 8.4 重构为什么总拖到最后

Anton Babenko 在 [Weekly.tf #20](https://www.weekly.tf/p/weeklytf-20) 观察：同一 state 内重构已经烦，跨 state 重构痛苦到几乎没人做。这是 2020 年表达，但今天仍提醒：

- 资源地址是 API；
- module path 变化会影响地址；
- 大爆炸式拆 state 风险高；
- `moved` block 能改善同 state 地址迁移，但跨 state 仍需要显式迁移协议；
- 一开始完全猜对边界不现实，所以系统必须允许小批量、可验证地重构。

---

## 9. AWS 的“人类可运维性”：Alex Chan 的小技巧为什么很强

### 9.1 给资源打 code-to-cloud URL

Alex Chan 建议在 AWS provider `default_tags` 中放 Terraform 根配置 URL：

```hcl
provider "aws" {
  default_tags {
    tags = {
      TerraformConfigurationURL = "https://github.example/platform/iac/tree/main/live/prod/service-a"
      TerraformRoot              = "live/prod/service-a"
      TerraformState             = "service-a/prod"
      Owner                      = "team-a"
    }
  }
}
```

这能：

- 从 AWS Console 一跳回代码；
- 发现未被 Terraform 管理的遗留资源；
- 支持 IAM key/安全审计；
- 暴露两个 root 争同一对象的双 owner；
- 为事件处理和资产图提供索引。

边界：

- 并非所有 AWS 资源支持 tags；
- tag 可被修改，不是授权证据；
- URL 可能泄露内部 repo 结构，应按组织安全策略设计；
- owner/state 元数据应有机器校验，不能只依赖人工填写。

### 9.2 给 S3 bucket 放 README

[Adding a README to S3 buckets with Terraform](https://alexwlchan.net/2025/s3-bucket-readme/) 建议在 bucket 内创建可见对象，说明：

- bucket 用途；
- owner/联系渠道；
- 数据分类与保留期；
- 文档与 Terraform 配置入口；
- 是否可手工修改。

这是典型的人因工程：操作者正在看 bucket 内容时，说明就在现场。它不能替代：

- bucket policy；
- object lock；
- lifecycle rule；
- Macie/数据分类；
- 机器可读 catalog；
- 访问审计。

最好的组合是：**结构化 tag/catalog 供机器使用，README 供现场人类使用。**

### 9.3 神秘 IAM key

[Finding a mystery IAM access key](https://alexwlchan.net/2023/iam-keys/) 展示一个小工具如何把 access key 追到：

- AWS account；
- IAM user；
- key age/status；
- 权限定义；
- AWS Console URL；
- Terraform URL。

这说明 Terraform metadata 的价值应从“写代码”延伸到 incident response。短期 STS credential 需要 CloudTrail 等其他观测路径，不能只靠 IAM user lookup。

### 9.4 retry 会掩盖故障

[Debugging a stuck Terraform plan](https://alexwlchan.net/2019/debugging-a-stuck-terraform-plan/) 通过降低 AWS provider retry 让 flaky API 立即暴露。文章中的参数属于旧 provider 时代，不应无脑复制；长期模型仍成立：

- 重试把快速失败变成“卡住”；
- 诊断环境需要可控 retry、timeout 和详细日志；
- 生产 retry 与诊断 retry 是不同策略；
- 先确认 AWS API、DNS、代理、credential、region，再怀疑 Terraform graph。

### 9.5 不要为开发便利削弱生产语义

在 [Listing deleted secrets](https://alexwlchan.net/2021/listing-deleted-secrets/) 中，作者反复 create/destroy Secrets Manager secret，recovery window 阻止同名重建。他拒绝仅为开发循环把 production module 的 `recovery_window_in_days` 改成 0，而是做独立清理工具。

可泛化为：

- prod module 保留安全默认；
- test fixture 可以有更快生命周期；
- 清理工具要显式、受限、只针对 sandbox；
- 不要把“测试容易”变成生产删除语义。

---

## 10. AWS 多账号、AFT 与执行平台：个人经验为什么会互相打架

### 10.1 Jack Lindamood 的四年 startup 判断

Jack 的具名生产复盘给出以下选择：

- AWS 胜过 GCP：基于其团队对支持、稳定性和生态的体验；
- Control Tower Account Factory for Terraform：支持，解决账号自动化和统一 tag；
- account tag 比单一 Organizations 树更适合表达多维属性；
- Terraform 胜过 CloudFormation：HCL 可读，且可同时管理 PagerDuty 等 SaaS；
- 不选 Pulumi/CDK：HCL 限制被视为降低复杂度；
- 不选 Terraform Cloud：成本不合适，改用 Atlantis + CI glue；
- GitOps：支持，但必须补“commit 为什么还没部署”的可见性；
- 外部 Secrets 同步优于把密文封进 Git；
- 简单与可调试性优先于漂亮但复杂的技术。

### 10.2 不能泛化的部分

这是 greenfield、快速增长 startup 的一个样本：

- AFT 是否适合取决于 Control Tower 历史、账号基线、既有 pipeline 和组织政治；
- Terraform Cloud 价格/价值会随团队规模、合同和治理需求变化；
- EKS、Karpenter、Datadog 等判断受具体 workload 强烈影响；
- “HCL 简单”不代表 module/repo/state 体系自动简单。

正确吸收方式是把文章当作**带条件的 decision log**：

```text
团队规模 + 既有系统 + 账号历史 + 操作能力 + 成本模型
                    ↓
              当时的选择
                    ↓
          四年后 endorsement/regret
```

而不是把绿色方块复制成你的标准答案。

### 10.3 Terraform Cloud vs Atlantis vs 自研 CI

评估维度：

| 维度 | 托管平台 | Atlantis | 自研 CI |
|---|---|---|---|
| 上手 | 通常较快 | 中等 | 慢 |
| 运维 | 厂商承担较多 | 自己运维服务 | 自己运维全部 glue |
| 定制 | 受平台约束 | workflow 可扩展 | 最大 |
| 审计/策略 | 通常内建较多 | 需组合 | 全自建 |
| 成本 | 订阅/用量 | 人力 + infra | 人力通常最高但隐蔽 |
| 凭据边界 | 平台模型 | 自行设计 | 自行设计 |
| 迁移锁定 | 平台 API/状态 | 较低 | 内部工具锁定 |

“免费 Atlantis”不代表零成本；“买平台”也不代表不需要 Terraform 平台工程。

---

## 11. OIDC、IAM Identity Center 与“双重所有权”

### 11.1 GitHub Actions 到 AWS：角色多不是问题，信任过宽才是

Lays Rodrigues 的 Substack 记录了一个典型成熟过程：从 IAM user 的长期 access key，迁到 GitHub OIDC + AWS STS 短期会话。核心方向正确：

- GitHub workflow 不保存长期 AWS key；
- AWS 注册 GitHub OIDC issuer；
- workload 通过 `AssumeRoleWithWebIdentity` 获得短期凭据；
- trust policy 约束 audience；
- trust policy 约束 subject 到具体 repo/branch/tag/environment；
- permission policy 只给该 workload 需要的 AWS 权限。

她问“每个 repo 一个 role 是否构成安全问题”。通常相反：一个 OIDC provider 可服务多个 workload role，而 role-per-workload 能分离权限、审计和吊销范围。

当前 [GitHub 官方 AWS OIDC 文档](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) 与 [AWS IAM 文档](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html) 的关键校正：

- provider URL：`https://token.actions.githubusercontent.com`；
- 使用官方 action 时 audience 常为 `sts.amazonaws.com`；
- AWS 要求/强烈建议检查 `token.actions.githubusercontent.com:sub`；
- 不可把 `sub` 设为空或只有通配符；
- 使用 GitHub Environment 时，subject 格式不同，并应启用 environment protection；
- 2026 年新建或 opt-in 的仓库可能使用带 immutable owner/repository ID 的 subject，trust policy 必须匹配实际格式；
- 旧博客中的 thumbprint 操作细节可能已经变化，实施时以当前 AWS/GitHub 文档为准。

### 11.2 OIDC 不自动等于最小权限

OIDC 只解决“凭据如何获得”，不自动解决：

- role 权限是否过大；
- session duration 是否合理；
- 哪些 branch/environment 可部署；
- workflow 中第三方 Action 是否可信；
- fork PR 是否可触达秘密；
- apply 是否与已审 plan 同一 commit；
- role 能否关闭 deletion protection；
- CloudTrail、session tag、GitHub run ID 是否能关联。

建议给 Terraform session 加可追踪字段，至少能从 CloudTrail 回到：

```text
repo + workflow + run_id + commit + environment + terraform_root
```

### 11.3 IAM Identity Center 的正确对象模型

Maxine Meurer 的 [IAM Identity Center 文章](https://ilovedevops.substack.com/p/iam-identity-center-the-access-control) 解释了几个常见误区：

- Identity Center user 不是 IAM user；
- permission set 不是现成 IAM role，而是生成目标账号 role 的模板；
- account assignment 是 `principal × permission set × AWS account` 三元关系；
- customer managed policy 若被 permission set 引用，目标账号必须已存在同名 policy；
- policy 分布与 permission set provisioning 可能跨 state，需要显式顺序；
- 批量 assignment 应使用稳定的 `for_each` key，避免 list index churn；
- Identity Center 解决凭据与集中分配，不自动保证 least privilege。

文章还提到 duplicate permission-set name 可能让 provider 长时间 retry、规模刷新可能 throttling。这类具体 bug 属于 **[个人生产观察]**，在你的 provider 版本上应先查 release notes/GitHub issue 再作为事实使用。

### 11.4 SCIM 与 Terraform：同一个对象只能有一个 writer

[Entra SCIM 文章](https://ilovedevops.substack.com/p/entra-scim-sync-to-iam-identity-center) 给出清晰 owner 边界：

```text
Entra / 外部 IdP
  └─ owns users, groups, membership

Terraform
  └─ owns permission sets, policies, account assignments
       └─ data lookup 已同步的 group
```

若 Terraform 和 SCIM 都创建 group：

- external ID 可能不同；
- 同名对象可能重复；
- plan 时 group 尚未同步会失败；
- pipeline 出现异步 chicken-and-egg；
- 人员离职/组变更的权威来源不清。

这不是 Identity Center 特例，而是 Terraform 的普遍规律：**同一字段/对象若有两个 controller，必须明确 ownership 或使用可验证的共享字段协议。**

---

## 12. “Terraform 能避免 AWS lock-in”为什么是错误话术

David J. Eddy 在 2018 年就指出：

- Terraform 提供跨 provider 的共同工作流；
- 但 AWS、Azure、GCP 的网络、实例、存储、IAM、托管服务语义不同；
- 把 `provider "aws"` 改成另一 provider 不会迁移系统；
- 真正 lock-in 在数据、API、运维知识、网络、身份、SLO 和业务架构。

更精确地说，Terraform 可以降低：

- 团队学习多个声明式工具的成本；
- CI、review、policy、module registry 的流程差异；
- 管理 SaaS + AWS 的工具碎片；
- 迁移时重新发明基础设施交付流程的成本。

Terraform 不能消除：

- RDS/Aurora 数据迁移；
- IAM 与其他云身份模型差异；
- VPC/TGW/PrivateLink 网络语义；
- EKS/ECS/Lambda 的运行时耦合；
- CloudWatch/EventBridge/Step Functions 的业务依赖；
- AWS 配额、故障模式和团队知识。

所以多云设计应该从业务可替换性出发：

| 组件 | 可替换性问题 |
|---|---|
| 数据 | 导出格式、复制、RPO/RTO、出口成本 |
| 身份 | principal、policy 语义、federation |
| 网络 | 地址、路由、私网服务、DNS |
| 运行时 | 镜像/协议是否可移植，控制面是否专有 |
| 可观测性 | telemetry 是否开放标准 |
| IaC | 工作流是否统一，资源定义仍需重写多少 |

“使用 Terraform”只是最后一行。

### 12.1 License 争论的当前校正

Yehuda Cohen 在 2023 年 license 变更当天预测：Terraform 的最终用户惯性仍很强，provider 很可能保持开放，直接与 Terraform Cloud 竞争的平台受影响更大，社区可能 fork。他的文章有价值，因为它是 AWS provider 贡献者的即时判断，但今天不能当现状说明。

当前可确认：

- HashiCorp 把 Terraform 后续版本从 MPL 2.0 改为 BSL 1.1/source-available，并给最终用户和部分集成场景额外使用授权；具体商业适用性应看当前条款，而不是博客转述；
- 社区 fork 已成为 Linux Foundation 旗下 [OpenTofu](https://opentofu.org/faq/)；
- Terraform 与 OpenTofu 仍可在大量既有 HCL/provider/module 工作流上互通，但已是不同项目，不能假设未来功能、state、registry 和兼容承诺永远相同；
- 对普通 AWS 基础设施团队，选择不应只看许可证口号，还要看 provider/module 兼容、升级测试、执行平台、支持责任和退出成本；
- 若公司把 Terraform 嵌入商业产品或托管竞争服务，应让法务基于 [HashiCorp 当前许可说明](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license) 判断，个人 Substack 不是法律意见。

这组材料最值得吸收的是治理风险：**IaC engine 也是供应链与长期所有权决策，不能只把它当一个 CLI 二进制。**

---

## 13. Brian Grant 的反方：IaC、CLI 与控制面

### 13.1 十二个 anti-factors 值得听，但不能全盘接受

Brian Grant 把 IaC 的问题概括为手工编写、体验差、可执行/需求值、过度 DRY、file-based、CI factory、单向、sprawl、secret-as-code、与其他 controller 互斥、脆弱 state、portal 只是包装 CLI 等。

其中对 Terraform × AWS 最有用的挑战：

- 一次 `plan/apply` 是 push-based，不是持续 reconciliation；
- break-glass、cost optimizer、安全响应、autoscaler 会直接改 AWS，Terraform 不会自动吸收；
- repo 写入口与 AWS API 读入口不对称；
- fleet-wide 查询/批量修改很难只靠散落 HCL；
- state/CLI 缺少长运行控制面的状态 API 与可观测性；
- 平台 portal 若只写 module input，却从 AWS 读 actual state，用户看到的是两个不同模型。

### 13.2 需要给 Brian 的论证打折

- Terraform refresh/plan 并非完全不读真实系统；
- Git 的 review、历史和不可变 artifact 不是纯粹“文件负担”；
- WET 配置也会复制错误并增加批量变更；
- controller 引入持续服务、HA、升级、租户、权限和自身 state；
- 作者在构建 ConfigHub，替代范式具有商业利益；
- “API/数据库优于代码”需要回答 schema migration、review、离线编辑、回滚和供应链问题。

### 13.3 什么时候 Terraform CLI 足够

- 变更低频；
- 资源生命周期长；
- apply 可在分钟内完成；
- 不要求实时自愈；
- ownership 清晰；
- 规模可按 state 分片；
- 失败可人工重试；
- AWS 本身提供长期控制面，如 RDS、ASG、EKS；
- CI 能可靠保存计划、身份和日志。

### 13.4 什么时候应该引入控制面

Brian 在 [control plane vs CLI](https://itnext.io/automation-using-control-planes-vs-command-line-tools-66f818ff8278) 给出的判断很实用：

- 需要持续运行或小时/天级事务；
- 需要故障恢复、高可用和自动 retry；
- 要对资源自发变化做反应；
- 要协调大量实体；
- 需要稳定 API、实时 status、GUI/LLM 多入口；
- 需要服务端权限代理、quota 和 state-based policy；
- 想隐藏 backend/credential/实现细节。

### 13.5 实用边界：不是 Terraform 或 Crossplane 二选一

```text
Terraform / OpenTofu
  ├─ AWS account baseline
  ├─ VPC / IAM foundation
  ├─ RDS / durable managed services
  └─ 控制面的安装与权限

Controller / GitOps / platform API
  ├─ 高频 workload deployment
  ├─ 持续 drift reconciliation
  ├─ autoscaling / rollout / failover
  └─ 自助请求和长事务状态
```

Terraform 可以创建 controller；controller 不必接管所有 Terraform 对象。边界按更新频率、故障模型和 owner 决定。

---

## 14. 独立 Newsletter 的价值：雷达，不是圣经

### 14.1 Weekly.tf #20：重构债

最强编辑判断是：Terraform code organization 很重要，但团队在项目早期信息最少；偏偏跨 state 重构又最难。这意味着平台应投资：

- `moved`/import 迁移规范；
- state owner registry；
- 自动生成迁移计划；
- 小批量转移与双重验证；
- 旧/new root 的 AWS inventory 对比；
- 回滚路径，而不是禁止重构。

### 14.2 Weekly.tf #42：plan 的攻击面

Anton 把 Plan RCE 与 default tags 放在同一期很有启发：

- 一边是 code-to-cloud 可追踪性；
- 一边是 cloud credential 进入不可信代码；
- Terraform 平台必须同时解决 provenance 与 execution trust。

### 14.3 Weekly.tf #273：boring Terraform 与 AI review

2026 年社区雷达里同时出现：

- KISS vs DRY：过早引入层层 abstraction 增加 3 AM 调试成本；
- schema validation：YAML/外部配置需要 CUE 等 schema；
- 组织级 HCL refactor；
- CloudTrail 归因的 drift detection；
- AI 对 plan/diff 做 intent mismatch review。

不要把 AI reviewer 变成批准者。更合理是：

```text
deterministic checks（fmt/validate/policy/plan）
  + runtime evidence（drift/CloudTrail/health）
  + AI explanation / anomaly ranking
  + human or policy authority
```

### 14.4 Weekly.tf #280：一份 state 管所有生命周期会变成意大利面

2026-07-22 的一期收录了：

- Terraform vs Crossplane 的 push/pull 争论；
- 一个 state 同时管理 infra、application pod、secret 导致 destroy 卡死和秘密进入 S3；
- Infracost + OPA 的预部署成本 guardrail；
- 能运行 Terraform 的 agent runtime 与阻断 destroy 的 agent guard。

这些内容仍需看原文验证，但指向一个共同趋势：**2026 年竞争焦点已从“AI 能否写 HCL”转向“谁能证明生成的基础设施变更有正确状态、权限、成本和恢复边界”。**

---

## 15. 个人博客中最有价值的冲突

### 15.1 HCL 限制：安全特性还是生产力阻碍？

**支持：**

- unknown value 与 plan 承诺需要受限求值；
- 限制让复杂度更明显；
- review 时资源图比任意程序更可读。

**反方：**

- HCL 仍需执行，展开结果难查询；
- 复杂 module 只是把程序藏起来；
- 批量修改和 self-service API 很难；
- 通用语言/配置数据库更容易构建工具。

**合成：**

- root/module API 保持窄、稳定、可验证；
- 生成器只生成普通、可读、可接管的 HCL；
- 对 fleet 查询另建 inventory/graph，不强迫 HCL 兼任数据库；
- 任何 abstraction 都要有展开视图和 escape hatch。

### 15.2 DRY vs WET

**过度 DRY：**

- 一个 module change 影响几十个 account；
- conditional/dynamic block 隐藏真实资源；
- 团队被迫同步升级；
- 3 AM 无法局部理解。

**过度 WET：**

- dev/prod 代码分叉；
- 安全修复漏环境；
- provider 升级重复劳动；
- definition drift。

**合成：**

- 复用稳定 capability，不复用偶然相似；
- 版本化 module，不直接跟随 main；
- root 保留清晰参数，不用 YAML 重新发明 HCL；
- 用 promotion 保证同一 artifact 跨环境；
- 允许安全地 fork，但必须有 owner 和退出计划。

### 15.3 state 分片 vs 全局图

**大 state：**

- 完整依赖图；
- 少 remote-state/接口；
- 操作简单。

**代价：**

- 单锁；
- 慢 plan；
- 大 blast radius；
- owner/权限不清；
- agent 一次拥有太多能力。

**小 state：**

- 并行、隔离、清晰 owner；
- 计划更快、权限更窄。

**代价：**

- 跨 state 依赖和顺序；
- 接口发布；
- 迁移与一致性；
- 分布式故障。

**合成：** 按 owner、生命周期、权限、故障域和变更频率切，不按固定资源数切。

### 15.4 drift：错误还是运营现实？

**GitOps 正统：** live 必须回到 desired。  
**运营反方：** 事故缓解、临时流量切换、紧急权限收紧可能是正当 drift。  
**合成：**

1. 发现并归因 drift；
2. 分类 accidental / malicious / mitigation / migration / stale-code；
3. 对 active incident drift 冻结自动回滚；
4. 要么回写代码，要么显式到期；
5. 结束时恢复单 owner。

### 15.5 AFT：值得还是过度？

- greenfield startup：AFT 自动账号、标准 tag，很值；
- brownfield/legacy：既有账号、基线、网络和 pipeline 可能让 AFT 改造成本过高；
- 单人小项目：Control Tower/AFT 可能超过运营收益；
- 大组织：不使用可审计 account factory 也会被手工账号债淹没。

没有脱离组织历史的 AFT 最佳答案。

---

## 16. 从这些高论导出的 Terraform × AWS 设计规约

### 16.1 每个 root 必须声明的 identity

```yaml
terraform_root: live/prod/service-a
state_key: service-a/prod/terraform.tfstate
aws_account_id: "123456789012"
aws_role: arn:aws:iam::123456789012:role/terraform-service-a-prod
owner: team-a
environment: prod
criticality: tier-1
backup_owner: data-platform
```

CI 在 init/plan 前验证实际值与声明一致；不一致直接失败。

### 16.2 plan 门禁最少检查

- 当前 AWS account ID 与 allowlist；
- caller ARN 与预期 role；
- backend bucket/key/region；
- CLI workspace 必须是预期值，最好生产不依赖 CLI workspace；
- create/update/replace/destroy 数；
- RDS、S3、KMS、IAM、Route53 等关键资源 destructive action；
- module/provider source 和版本变化；
- `.terraform.lock.hcl` diff；
- `external`、provisioner、local execution 新增；
- `prevent_destroy`/deletion protection 的删除或关闭；
- plan commit 与 apply commit；
- active incident / freeze / change window；
- backup restore 最近成功时间；
- drift 与 manual mitigation；
- estimated monthly cost delta。

### 16.3 角色分层

```text
PR static-test role
  └─ 无生产 state、无生产 AWS

production-plan role
  └─ 只读 AWS + state read（能避免则不给敏感 secret read）

production-apply role
  └─ 受控写权限，短期 OIDC，会话可追踪

break-glass role
  └─ 可关闭 deletion protection / 执行关键删除
     与日常 apply role 分离，多人/事件审批

backup-admin role
  └─ 与生产删除者分离，最好跨账号
```

### 16.4 资源 owner 规则

- 一个对象一个 primary writer；
- 允许外部系统修改时，按字段声明 owner；
- `ignore_changes` 不是隐藏冲突的垃圾桶；
- controller、console、Terraform、SCIM、autoscaler 的写边界写进 runbook；
- 手工变更必须有事件、owner、TTL 和回写/回滚决定。

### 16.5 数据资源规则

- Terraform 创建数据库基础设施，不管理 schema/data migration；
- database deletion protection 在 Terraform 与 AWS 两层；
- 日常 apply role 不得关闭控制面保护；
- backup 与 source resource 不同生命周期/权限；
- 定期 restore + 业务查询；
- plan 中 replace/destroy 数据资源默认阻断；
- “有 snapshot”不算恢复能力。

---

## 17. 给 LocalStack Ultimate 的 16 个高价值实验

> LocalStack 是云服务模拟器，不是 AWS 的形式化等价物。实验用于验证 Terraform 逻辑、状态、API 交互、事件链和防线失败方式；真实 IAM/SCP、配额、延迟、DNS、服务级删除语义仍需在隔离 AWS sandbox 做最小 canary。可参考 [LocalStack Terraform Testcontainers 教程](https://docs.localstack.cloud/aws/tutorials/using-terraform-with-testcontainers-and-localstack/)、[Terraform test 集成文章](https://blog.localstack.cloud/efficient-infrastructure-testing-localstack-terraform-tests-framework/) 和 [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/test-aws-infra-localstack-terraform.html)。

### 实验 1：unknown value 与稳定 identity

**目标：** 理解为什么 `for_each` key 必须 plan-time known。

1. 用固定 map key 创建 VPC/subnet；
2. 保存 plan，确认 VPC ID unknown 但 subnet 地址确定；
3. 改为从 apply 后结果推导 key；
4. 观察 plan 拒绝；
5. 重构为调用者提供稳定 key。

**验收：** 能解释 value unknown 与 collection identity unknown 的区别。

### 实验 2：plan/apply 间外部文件变化

**目标：** 观察 plan 承诺如何被不纯输入破坏。

1. 配置读取一个本地 JSON；
2. 保存 plan；
3. apply 前修改文件；
4. 记录 Terraform 是否使用保存值、重新读取或报一致性错误；
5. 将输入改为 CI 生成且 checksum 固定的 artifact。

**验收：** 外部输入进入 plan provenance 清单。

### 实验 3：empty state 的重复创建

**目标：** 复现 DataTalks.Club 事故的第一阶段，不触碰真实 AWS。

1. 在 LocalStack 创建 S3/SQS/DynamoDB 组合；
2. 保留真实 state A；
3. 切到空 state B，对同一配置 plan；
4. 观察哪些资源因名字唯一而报错，哪些会创建重复对象；
5. 用 tags/inventory 检查双 owner。

**验收：** pipeline 在 state 为空但账号中存在同 root tag 资源时必须 fail closed。

### 实验 4：state swap + destroy canary

**目标：** 证明“当前目录 state”比自然语言意图更有执行权。

1. 两套不同前缀资源和 state；
2. 把 production-like state 复制到 cleanup root；
3. 只生成 destroy plan，不 apply；
4. 机器解析计划中资源 root tag 与当前 root identity 不一致；
5. 门禁阻断。

**安全：** 只在 LocalStack，使用随机实验前缀；禁止连接真实 AWS endpoint。

### 实验 5：backend/root/account 三元绑定

**目标：** 消灭错 state/错账号。

在 preflight 中读取：

- `aws sts get-caller-identity`；
- 期望 account ID；
- backend key；
- root manifest；
- provider region；
- current workspace。

任何一个不匹配退出非零。为 dev/prod 交叉组合写负测试。

### 实验 6：untrusted plan RCE 靶场

**目标：** 证明 plan 不是安全只读。

1. 建隔离容器，只有假凭据与 LocalStack 网络；
2. PR fixture 添加 `external` data source 写 canary 文件；
3. 运行 plan，确认 canary 出现；
4. 加 policy 阻断 `external`/provisioner/未知 provider；
5. runner 禁止生产网络、metadata endpoint 与真实 credential。

**验收：** untrusted PR 流程拿不到任何生产能力，即使规则漏掉一种执行路径。

### 实验 7：provider/module 供应链

**目标：** 固定执行依赖。

- 删除 lockfile，观察 init 选版；
- 恢复 lockfile，确认版本固定；
- 修改 checksum，确认安装失败；
- 为 Windows/Linux 预填 platform hash；
- policy 阻断未知 provider source、Git module 浮动 branch；
- 记录 module 未被 lockfile 锁定的事实。

### 实验 8：ephemeral/write-only 秘密

**目标：** 比较 `sensitive` 与“不持久化”。

1. 传统 random password → resource；
2. 检查 plan/state JSON 中的持久化；
3. 改为 ephemeral + provider 支持的 write-only 字段；
4. 再检查 plan/state；
5. 加 secret rotation version；
6. 验证 module input/output 的 ephemeral 传播。

**边界：** 目标资源不支持 write-only 时，不假装解决；记录 provider gap。

### 实验 9：Secrets Manager recovery window

**目标：** 分离生产安全语义与测试清理。

- production module 保留 recovery window；
- sandbox fixture 使用独立前缀和受限 cleanup；
- 重复 create/destroy；
- 确认同名重建失败路径；
- cleanup 工具只能匹配实验 tag/前缀；
- 不允许把 `recovery_window_in_days = 0` 回传到 prod module。

### 实验 10：code-to-cloud tags 与双 owner

**目标：** 把 Alex Chan 的想法变成自动控制。

1. 所有可 tag 资源注入 `TerraformRoot/State/Owner/ConfigURL`；
2. 第二 root 尝试管理同一对象并写另一 URL；
3. drift/plan 检测 tag 冲突；
4. inventory 报告 untagged 与 multi-owner；
5. 对不支持 tag 的资源建立外部 ownership registry。

### 实验 11：S3 README 与机器元数据

**目标：** 同时服务人和机器。

- bucket 创建 `README.txt`；
- 内容由 owner、用途、数据分类、保留、runbook 生成；
- tag/catalog 存结构化同源数据；
- 测试 README 与 manifest 不一致时 CI 失败；
- 证明 README 不能替代 bucket policy/lifecycle。

### 实验 12：OIDC trust policy 负测试

**目标：** 验证 trust 条件，而不是只看 happy path。

为以下 subject 建测试：

- 正确 repo + branch；
- 正确 repo + 错 branch；
- fork repo；
- 正确 repo + environment；
- audience 错误；
- 通配到整个互联网；
- 2026 immutable-ID subject 格式。

LocalStack 可测试 Terraform 生成与 IAM policy shape；真实 token exchange 仍要在隔离 AWS sandbox 验证。

### 实验 13：SCIM/Terraform 双 owner 模拟

**目标：** 验证异步身份边界。

若 LocalStack 对 Identity Center/SCIM 覆盖不足：

- 用小型 mock SCIM API 或 fixture inventory；
- Entra 模拟器异步创建 group；
- Terraform data lookup 在同步前失败；
- Terraform 若也 create group，制造 duplicate external ID；
- 两阶段 pipeline 等待同步完成后再 account assignment。

**验收：** owner contract 明确，不以无限 retry 掩盖依赖。

### 实验 14：declared/stored/actual/observed/operational reconciliation

建立一个最小 JSON 汇总：

```json
{
  "declared_commit": "...",
  "state_serial": 42,
  "actual_inventory_hash": "...",
  "health": "degraded",
  "incident": "INC-123",
  "manual_mitigation": true
}
```

策略：

- state/actual 不一致 → 需要 drift review；
- health degraded → 禁止普通 apply；
- active incident + mitigation → 禁止自动回滚 drift；
- destructive plan + restore test stale → 阻断；
- agent 只能提出建议，不能越过策略。

### 实验 15：备份恢复而不是备份存在

用 S3 + DynamoDB/RDS 可模拟部分建立：

1. 写入带 checksum 的测试数据；
2. 生成与主资源不同生命周期的备份；
3. 销毁主资源；
4. 从备份恢复到新资源；
5. 执行业务级查询/checksum；
6. 记录 RTO 与最后成功时间；
7. 让 destructive plan 门禁读取该时间。

真实 RDS snapshot、跨账号 Vault/Backup、KMS 权限仍需真实 AWS sandbox 验证。

### 实验 16：CLI 与 controller 边界

同一个 SQS/Lambda/DynamoDB 事件链：

- Terraform 创建低频基础设施；
- 应用/controller 处理高频消息和 runtime config；
- 人工制造 drift；
- 比较定时 `plan` 与持续 reconciliation 的发现/恢复时间；
- 明确哪些字段归 Terraform、哪些归 controller；
- 测试 controller 暂停时 Terraform 不应误覆盖 incident mitigation。

---

## 18. 五层验证金字塔

| 层 | 工具 | 证明什么 | 不能证明什么 |
|---|---|---|---|
| L0 静态 | fmt、validate、tflint、policy、schema | 语法、类型、显式规则 | provider/API 行为 |
| L1 计划 | plan JSON、成本、destroy/replace gate | Terraform 预期动作 | 服务实际健康、并发、真实控制面 |
| L2 本地云 | LocalStack Ultimate、Terraform test、Testcontainers | AWS API 交互、事件链、state/owner/失败注入 | 完整 AWS parity、真实 IAM/SCP/配额/延迟 |
| L3 真 AWS | 临时 sandbox account、最小 canary | 真实 provider 与 AWS 行为 | 生产数据/SLO，仍需上线策略 |
| L4 生产验证 | health probe、SLO、CloudTrail、restore drill | 真实业务与恢复能力 | 不能替代前面各层 |

LocalStack 的正确位置是 L2，不是“假装 L3/L4 已完成”。它的最大价值是让破坏性实验便宜、快速、可重复：

- state swap；
- wrong-account guard；
- destroy policy；
- untrusted plan；
- event-driven drift；
- backup/restore workflow；
- module contract；
- provider failure injection。

---

## 19. 30 天吸收路线

### 第 1 周：建立 Terraform 机制模型

1. Martin：Unknown Values；
2. 用 LocalStack 做实验 1–2；
3. Ricard：stateless 自我修正；
4. 写出 state 的五个职责；
5. Martin：workspaces；
6. 对照当前官方 workspace/S3 backend 文档。

**产出：** 一页 `plan-state-workspace.md`，能解释 unknown、identity、state、backend 和 root。

### 第 2 周：安全与秘密

1. Plan RCE；
2. 实验 6–8；
3. Martin：Ephemeral Values；
4. GitHub/AWS OIDC 当前文档；
5. Lays OIDC 实战；
6. provider lockfile 与 module source policy。

**产出：** PR trust matrix、runner role matrix、secret persistence audit。

### 第 3 周：AWS 运维与恢复

1. Alex Chan：tags、S3 README、IAM key、Secrets Manager；
2. Alexey DataTalks.Club 原始复盘；
3. 实验 3–5、9–11、15；
4. 为数据库设计 Terraform/AWS/IAM/backup 四层防线。

**产出：** production preflight 与 destructive-plan gate 规范。

### 第 4 周：平台边界与 agentic IaC

1. Kief：Snowflakes as Code；
2. Jack：四年 decision log；
3. Brian：anti-factors、control plane vs CLI；
4. Sumant：五层状态；
5. 实验 14、16；
6. 写出 Terraform/controller/人类/AI ownership matrix。

**产出：** 你的 Terraform × AWS operating model，而不是另一份“最佳实践清单”。

---

## 20. 可直接执行的审计清单

### State / backend

- [ ] 所有生产 state 已远端化；
- [ ] S3 versioning 已启用；
- [ ] S3 `use_lockfile = true` 或已记录迁移计划；
- [ ] backend bucket/key 与 root/account/owner 有机器绑定；
- [ ] state 管理员与普通 apply role 分离；
- [ ] 不使用 CLI workspace 充当强安全边界；
- [ ] state 丢失时 pipeline fail closed，不会直接创建重复资源；
- [ ] 有 state restore 演练。

### Dependency / plan

- [ ] `.terraform.lock.hcl` 已提交；
- [ ] root provider version 有合理上限；
- [ ] module source 使用明确 version/ref；
- [ ] 新 provider/module source 需要显式审批；
- [ ] untrusted PR 无生产凭据/state/network；
- [ ] plan 与 apply 绑定同一 commit 和 artifact；
- [ ] destroy/replace 关键资源机器阻断；
- [ ] 新增 `external`/provisioner/local execution 被检查。

### AWS identity

- [ ] CI 使用 OIDC/STS，不用长期 IAM user key；
- [ ] trust policy 限制 `aud` 和具体 `sub`；
- [ ] GitHub Environment 有 protection；
- [ ] session 能回溯 repo/run/commit/root；
- [ ] plan role 与 apply role 权限分开；
- [ ] break-glass 能力与日常 role 分开；
- [ ] 普通 apply role 不能关闭关键删除保护；
- [ ] account ID 在每次运行前验证。

### Secrets

- [ ] 已区分 sensitive 与 ephemeral；
- [ ] 检查 state 与保存 plan 是否包含秘密；
- [ ] workload 尽量运行时取 secret；
- [ ] provider 支持时使用 write-only；
- [ ] remote-state consumer 不获得整份敏感 state；
- [ ] secret recovery window 没因测试便利被削弱；
- [ ] rotation 流程可验证。

### Ownership / drift

- [ ] 可 tag 资源有 root/state/owner/config URL；
- [ ] 不可 tag 资源有外部 registry；
- [ ] 同一对象只有一个 primary writer；
- [ ] `ignore_changes` 有字段 owner 和原因；
- [ ] drift 会归因到 CloudTrail principal/run；
- [ ] incident mitigation drift 不会被盲目回滚；
- [ ] manual change 有 TTL 与回写决定。

### Data / recovery

- [ ] 数据库 Terraform `prevent_destroy`；
- [ ] AWS native deletion protection；
- [ ] IAM/SCP 层删除限制；
- [ ] 备份与源资源不同生命周期；
- [ ] 备份最好跨账号/权限域；
- [ ] 定期自动恢复并执行业务查询；
- [ ] destructive plan 会读取最近恢复测试状态；
- [ ] RPO/RTO 是实测值，不是文档愿望。

### AI agent

- [ ] agent 默认只能接触 LocalStack/sandbox；
- [ ] sandbox 使用短期一小时级会话；
- [ ] agent 无法访问 production state；
- [ ] 生产部署只经 CI；
- [ ] prompt/命令黑名单不被当作主安全边界；
- [ ] agent 生成依赖也走 supply-chain review；
- [ ] agent 读取 actual/observed/operational state 后才可建议变更；
- [ ] 决策权与执行权有确定性策略/人类门禁。

---

## 21. 版本与事实校正表

| 旧文/常见说法 | 2026 校正 |
|---|---|
| S3 backend 必须 DynamoDB lock | 当前 S3 backend 支持 `use_lockfile`；DynamoDB locking deprecated |
| `sensitive` 让秘密不进 state | 错；主要抑制显示，需 ephemeral/write-only 才解决部分持久化 |
| CLI workspaces 适合 dev/prod 强隔离 | 官方明确不适合不同凭据/访问控制和系统拆分 |
| `prevent_destroy` 是不可绕过删除锁 | 规则必须仍在配置；删掉 resource block 就删掉规则 |
| plan 是安全只读 | provider/data/external 等可在 plan 执行代码或访问 API |
| lockfile 固定所有 Terraform 依赖 | 当前主要锁 provider；远端 module 仍需明确版本 |
| GitHub OIDC 只配 issuer/audience 即可 | 必须限制 `sub`；environment 与 2026 immutable subject 格式需匹配 |
| 一个 repo 一个 AWS role 浪费且危险 | role-per-workload 通常强化 least privilege；真正风险是 trust/permission 过宽 |
| state 加密就解决 secret 问题 | runner 解密、plan、remote-state、导出仍扩散；应减少 Terraform 持有秘密 |
| LocalStack 通过就等于 AWS 通过 | 只证明仿真层行为；真实 IAM、配额、延迟和控制面需 sandbox canary |
| Terraform 避免云锁定 | 它统一工作流，不统一 AWS 与其他云的资源语义 |
| Git 是生产唯一真相 | Git 是声明意图；actual、observed、operational state 同样决定变更安全 |
| 所有 drift 都应自动回滚 | 先分类；incident mitigation 可能应暂时保留 |

---

## 22. 未采信或仅保留为线索的内容

以下类型没有进入硬结论：

- 匿名 Medium “Terraform 把生产搞垮两次”；
- 只给产品 benchmark、不提供 workload/repo/脚本的数据；
- 2026 新文仍直接复制旧 DynamoDB lock 模板，却不说明兼容背景；
- 声称某 provider bug “必现”但没有 issue、版本和 reproduction；
- 把 LocalStack 说成完全 production-identical；
- 把一次 startup 选择包装成所有企业答案；
- 把 OpenTofu/Terraform license 争论简化成技术性能优劣；
- 把 AI reviewer 当作 deterministic policy 或生产批准者；
- 用“multi-cloud”暗示应用、数据和网络无需重构；
- 只有几十行 HCL 和成功截图、没有 destroy/upgrade/drift/恢复路径的教程。

---

## 23. 结论：真正该吸收的社区能量

个人博客与 Substack 最强的地方，不是提供更多 HCL 片段，而是把隐含前提说出来：

1. Martin 解释了 Terraform 为什么必须限制语言，plan 才能成为承诺；
2. Ricard 展示了认真推理的人如何公开承认“我错了”，并把状态问题拆得更细；
3. Alex Chan 说明小型人因设计能显著改善 AWS 事故现场；
4. Jack 展示了工具选择必须绑定团队、成本与四年后的后悔；
5. Brian 迫使 Terraform 用户正视 CLI、文件、单向推送和控制面的边界；
6. Lays/Maxine 把身份、OIDC、SCIM 和 owner 冲突从抽象原则落到 AWS；
7. Sumant 把 agentic IaC 的问题从“会不会写 HCL”提升到“是否理解生产五层状态”；
8. Alexey 用一次代价巨大的公开复盘证明：状态、权限、备份和 agent 边界必须是结构性的。

最终可压缩成一句：

> **成熟的 Terraform × AWS 系统，不是让更多人和代理更快地运行 `apply`，而是让任何一次变更都能证明：我拿的是正确 state、进入的是正确账号、只拥有必要能力、理解当前生产状态，并且在最坏情况下真的恢复得回来。**

---

## 24. 来源索引

### A 级：机制与原始事故

- Martin Atkins — [Unknown Values: The Secret to Terraform Plan](https://log.martinatkins.me/2021/06/14/terraform-plan-unknown-values/)
- Martin Atkins — [Ephemeral Values in Terraform](https://log.martinatkins.me/2024/05/22/terraform-ephemeral-values/)
- Martin Atkins — [Re-imagining Terraform Workspaces](https://log.martinatkins.me/2019/11/01/rethinking-terraform-workspaces/)
- Alexey Grigorev — [How I Dropped Our Production Database and Now Pay 10% More for AWS](https://alexeyondata.substack.com/p/how-i-dropped-our-production-database)
- Alex Kaskasoli — [Terraform Plan RCE](https://alex.kaskaso.li/post/terraform-plan-rce)

### B 级：具名生产经验

- Alex Chan — [Tag your infrastructure-as-code resources](https://alexwlchan.net/2023/tag-iac-resources/)
- Alex Chan — [Adding a README to S3 buckets](https://alexwlchan.net/2025/s3-bucket-readme/)
- Alex Chan — [Finding a mystery IAM access key](https://alexwlchan.net/2023/iam-keys/)
- Alex Chan — [Debugging a stuck Terraform plan](https://alexwlchan.net/2019/debugging-a-stuck-terraform-plan/)
- Alex Chan — [Listing deleted secrets in AWS Secrets Manager](https://alexwlchan.net/2021/listing-deleted-secrets/)
- Jack Lindamood — [(Almost) Every infrastructure decision I endorse or regret](https://cep.dev/posts/every-infrastructure-decision-i-endorse-or-regret-after-4-years-running-infrastructure-at-a-startup/)
- Lays Rodrigues — [Temporary token for GitHub Actions to deploy into AWS](https://lays147.substack.com/p/quick-tutorial-temporary-token-for)
- Maxine Meurer — [IAM Identity Center in production](https://ilovedevops.substack.com/p/iam-identity-center-the-access-control)
- Maxine Meurer — [Entra SCIM Sync to AWS IAM Identity Center](https://ilovedevops.substack.com/p/entra-scim-sync-to-iam-identity-center)

### C 级：设计观点与反方

- Kief Morris — [The Snowflakes as Code antipattern](https://infrastructure-as-code.com/posts/snowflakes-as-code.html)
- Kief Morris — [Infrastructure Stack](https://infrastructure-as-code.com/posts/defining-stacks.html)
- Ricard Bejarano — [A more mature take on stateless Terraform](https://www.bejarano.io/terraform-stateless-critique/)
- Brian Grant — [The 12 Anti-factors of Infrastructure as Code](https://itnext.io/the-12-anti-factors-of-infrastructure-as-code-acb52fba3ae0)
- Brian Grant — [Automation using Control planes vs. Command-line tools](https://itnext.io/automation-using-control-planes-vs-command-line-tools-66f818ff8278)
- Brian Grant — [Is GitOps actually useful?](https://itnext.io/is-gitops-actually-useful-a1c851ba99d8)
- Sumant Thakur — [Your AI Agent Thinks the Repo Is Production](https://sumantthakur.substack.com/p/your-ai-agent-thinks-the-repo-is)
- David J. Eddy — [The case against Terraform to prevent vendor lock-in](https://dev.to/david_j_eddy/the-case-against-terraform-to-prevent-vendor-lock-in-3lle)
- Yehuda Cohen — [Initial thoughts about HashiCorp license changes](https://yehudacohen.substack.com/p/initial-thoughts-about-hashicorp)

### D 级：独立 Newsletter 雷达

- Anton Babenko — [Weekly.tf #20: Terraform refactoring and repo strategy](https://www.weekly.tf/p/weeklytf-20)
- Anton Babenko — [Issue #42: plan security and dependency updates](https://www.weekly.tf/p/weeklytf-issue-42-terraform-security-dependency-updates)
- Anton Babenko — [Issue #143: multi-account authentication and OIDC](https://www.weekly.tf/p/issue-143-everything-you-need-to-know-about-multi-account-authentication-and-configuration)
- Anton Babenko — [Issue #273: boring Terraform, drift, AI review](https://www.weekly.tf/p/issue-273-boring-terraform-beats-clever-cue-schema-validation-safe-avm-module-migrations-with-openre)
- Anton Babenko — [Issue #280: Crossplane, GitOps, FinOps and agents](https://www.weekly.tf/p/issue-280-crossplane-vs-terraform-spaghetti-to-gitops-finops-guardrails-with-opa-the-open-agent)

### 当前官方校正

- HashiCorp — [CLI Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)
- HashiCorp — [S3 Backend](https://developer.hashicorp.com/terraform/language/backend/s3)
- HashiCorp — [Ephemeral values](https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral)
- HashiCorp — [Lifecycle / prevent_destroy](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- HashiCorp — [Provider requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)
- HashiCorp — [Dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- GitHub — [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- AWS — [Creating a role for GitHub OIDC](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html)
- HashiCorp — [Business Source License announcement](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license)
- OpenTofu — [FAQ](https://opentofu.org/faq/)
- LocalStack — [Terraform + Testcontainers](https://docs.localstack.cloud/aws/tutorials/using-terraform-with-testcontainers-and-localstack/)
- LocalStack — [Terraform test framework integration](https://blog.localstack.cloud/efficient-infrastructure-testing-localstack-terraform-tests-framework/)
- AWS Prescriptive Guidance — [Test AWS infrastructure with LocalStack and Terraform Tests](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/test-aws-infra-localstack-terraform.html)
