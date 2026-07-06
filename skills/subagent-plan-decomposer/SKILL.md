---
name: subagent-plan-decomposer
description: 当用户提供已有方案、执行计划、PRD、implementation spec、QA plan、产品方案、代码实现方案或 Codex 分步计划，并要求改写为“主 agent + 子 agent”执行结构时使用。本技能将方案转成 requirement-led、模块级、可并行调度的 Subagent-Ready DAG，输出 requirement ledger、读写集、依赖、join gate、reviewer 时机、子 agent brief、调度规则和合并协议。只改写方案，不执行原方案。
---

# 子 Agent 方案拆解技能

## 目标

把已有方案改写成主 agent 可调度的 Subagent-Ready DAG。

本技能不执行方案、不写业务代码、不做最终验收。它只解决：

- 需求如何逐项对账。
- 模块如何拆分。
- 哪些任务给 subagent。
- 哪些任务可并行。
- 哪些任务必须等待 join gate。
- reviewer 审什么、何时审。
- 主 agent 如何调度和合并。

## 核心原则

默认使用 subagent。不要把“是否使用 subagent”作为主要判断。

默认按业务模块、用户路径、数据契约、安全边界或文档章节拆分；禁止按函数级小改、局部 helper、小 UI 片段机械拆分。

保留原方案目标、约束和意图，不新增用户未要求的新目标。

主 agent 保留：目标解释、优先级、冲突处理、最终验收、最终用户可见输出。

子 agent 只拿边界清晰、输入稳定、可独立验收的工作包。

方案不够详细时，不生成伪 DAG；输出 `PLAN_INSUFFICIENT`，列出必须由用户确认的缺口。

所有 `READY` 结论必须带 scope，例如 `READY(scope=decomposition)`、`READY(scope=dry-run-plan)`、`READY(scope=execution)`。没有真实文档、仓库路径、写权限或执行授权时，不得输出 `READY(scope=execution)`。

## 输入充分性 Gate

先判断方案是否足以拆解。

必须至少有：

- 目标。
- 范围和非目标。
- 可验收结果。
- 主要模块或功能清单。
- 关键约束。
- 外部副作用边界。
- 需要人类确认的位置。

如果缺少模块边界、验收标准、依赖关系、证据类型或副作用边界，输出：

```text
Decision: PLAN_INSUFFICIENT
Missing:
- ...
Questions for user:
- ...
```

不要自行脑补关键产品或工程决策。

## Requirement Ledger

当输入包含 PRD、spec、QA plan 或详细清单时，必须先抽取 requirement ledger。

每条 requirement 包含：

```text
requirement_id
source
requirement_text
owner_module
acceptance_evidence
status_seed: TODO / HUMAN_ONLY / DECISION_REQUIRED
notes
```

Ledger 只建立待执行账本，不宣称 PASS。完成度由后续 loop controller 更新。

Requirement 必须原子化。若一句需求包含多个入口、多个角色、多个负面 case、多个 `和/或` 条件或多个验收点，必须拆成多条 requirement。不要用一条 ledger 覆盖“外群/单聊/历史入口/URL 篡改/已退出成员”等多个独立 case。

## 拆分粒度

合格模块：

- 一个完整用户路径。
- 一个业务能力。
- 一个数据 contract。
- 一个安全 / 权限边界。
- 一个可独立验收的内容或 QA 包。

不合格模块：

- 单个普通函数。
- 普通空值判断。
- 仅为 reviewer 找问题而拆出的微小 helper。
- 不对应 requirement 的兜底分支。

除非小函数本身是加密、权限、删除、支付、发布等高风险边界，否则不要为它单独分配 subagent 或 reviewer。

## 标记定义

| 标记 | 使用规则 |
| --- | --- |
| `[[MAIN_AGENT_ONLY]]` | 目标、优先级、最终判断、冲突处理、用户可见输出。 |
| `[[SUBAGENT_REQUIRED]]` | 默认委派给子 agent 的模块级任务。 |
| `[[SPLIT_TO_SUBAGENTS]]` | 一个步骤应拆成多个模块或批次。 |
| `[[PARALLEL_GROUP: <id>]]` | 同组任务依赖已满足、写集不冲突，可并行。 |
| `[[JOIN_GATE: <id>]]` | 主 agent 必须等待指定输出后再继续。 |
| `[[DEPENDENCY: <id>]]` | 当前任务依赖某前置任务或 join gate。 |
| `[[REVIEW_AGENT]]` | 独立 reviewer，只审稳定输入。 |
| `[[HUMAN_ONLY]]` | 真机、真实账号、签收、审批、发布、清理等人类或外部条件。 |

不要输出 `SUBAGENT_OPTIONAL` 作为默认逃避项。若确实不应委派，标 `MAIN_AGENT_ONLY` 并说明原因。

## 读写集和副作用

每个任务必须声明：

- 输入范围。
- 允许读集。
- 允许写集。
- 禁止写集。
- 共享瓶颈文件或资源。
- 是否允许外部副作用。
- 是否需要用户确认。

如果两个任务写集重叠，不得放入同一 `PARALLEL_GROUP`。

远端部署、数据库写入、删除、清理、真实用户数据、正式发布默认不允许，除非用户明确授权。

如果输入提到共享瓶颈文件或资源，例如公共 client service、adapter、cloudfunction entry、schema、action registry，必须建立 shared foundation / ownership stage。不能只把它写进 ledger；后续模块必须依赖该 stage，或只改它预留的接入区。

Shared foundation 必须拆成两层：`MAIN_AGENT_ONLY` 只负责冻结 ownership、接口边界和写集规则；如果需要实际代码改动，必须另建 `SUBAGENT_REQUIRED` foundation worker，除非用户明确要求主 agent 亲自实现。

## Reviewer 规则

Reviewer 只能审稳定输入：

- 已冻结方案。
- 已完成模块。
- 已生成 artifact。
- 已完成测试计划。

Reviewer 不得审查尚未写出或正在写的代码。

Reviewer 的 revise 必须绑定：

- requirement。
- 用户路径。
- 数据 contract。
- 安全 / 隐私 / 权限风险。
- 发布或破坏性操作风险。

不得因为纯理论输入组合要求增加兜底代码。

## Scheduler 输出规则

输出必须让主 agent 能直接调度。

每个任务要有：

```text
task_id
module
requirements
dependencies
parallel_group
read_set
write_set
blocked_by
review_after
join_gate
dependency_rationale
```

还要输出 ready-task 规则：

1. 依赖满足。
2. 输入稳定。
3. 写集不冲突。
4. 外部副作用已授权。
5. reviewer 只在被审对象完成后 ready。

主 agent 应一次派发所有 ready 且不冲突的任务。

每条 dependency 必须写 rationale，说明为什么依赖存在。涉及 member state、access policy、shared contract、data rights、DLP 或发布边界时，不允许只有箭头没有理由。

QA 必须拆成两类任务：early QA plan / fixture / guardrail 准备，以及 final coverage gate。不要把所有 QA 都放到最后。

Human-only 必须拆成两类：agent 可自动生成的 readiness checklist，以及只能由人类或真实外部环境提供的 evidence requirement。不要把 checklist task 标成 evidence blocked。

Checklist generation task 不得标为 `HUMAN_ONLY`，也不得 `blocked_by` 人类执行；它应标为 `MAIN_AGENT_ONLY` 或 `SUBAGENT_REQUIRED`。真实 evidence requirement 才能标 `HUMAN_ONLY` / `BLOCKED`。

调度表缺少 `dependency_rationale` 字段时，输出不合格。不要用只含 `deps` 的表格替代。

## 输出结构

最终输出必须包含：

1. `拆解总览`
2. `Requirement Ledger`
3. `模块 DAG`
4. `调度表`
5. `改写后的 Subagent-Ready 方案`
6. `子 Agent 任务简报`
7. `Reviewer 计划`
8. `Join / 合并协议`
9. `主 Agent 调度检查清单`

## 输出模板

```markdown
# 拆解总览

- Decision: READY / PLAN_INSUFFICIENT
- Ready scope:
- 原方案来源：
- requirement 数：
- 模块数：
- subagent 任务数：
- reviewer 任务数：
- 并行组：
- join gate：
- human-only 项：
- 主 agent 保留职责：

# Requirement Ledger

| id | source | requirement | owner_module | evidence_needed | status_seed |
|---|---|---|---|---|---|

# 模块 DAG

```text
M0_REQUIREMENTS
  -> P1A_MODULE
  -> P1B_MODULE
P1_JOIN
  -> P2A_MODULE
```

# 调度表

| task_id | module | deps | dependency_rationale | parallel_group | read_set | write_set | review_after | blocked_by |
|---|---|---|---|---|---|---|---|---|

# 改写后的 Subagent-Ready 方案

## 阶段 0：主 Agent 初始化

1. `[[MAIN_AGENT_ONLY]]` 固定目标、范围、ledger 和外部副作用边界。

## 阶段 1：并行模块

2. `[[SUBAGENT_REQUIRED]] [[PARALLEL_GROUP: P1]]` `<module>`
   - requirements:
   - input:
   - output:
   - read_set:
   - write_set:
   - do_not:
   - acceptance:

3. `[[JOIN_GATE: P1_JOIN]]`
   - wait_for:
   - merge_rule:
   - pass_condition:

# 子 Agent 任务简报

## `<task_id>` - `<role>`

**目标：**
**输入范围：**
**允许读集：**
**允许写集：**
**禁止事项：**
**输出格式：**
**验收标准：**
**依赖：**
**压缩等级：**

# Reviewer 计划

| reviewer_id | reviews | ready_after | must_check | must_not_do |
|---|---|---|---|---|

# Join / 合并协议

- 必须等待：
- 主 agent 如何合并：
- 冲突处理：
- 证据优先级：
- 更新哪些 requirement：
- 进入下一阶段条件：

# 主 Agent 调度检查清单

- 是否有 requirement ledger。
- 是否每个 requirement 都是原子验收点。
- 是否按模块拆分，不是函数级碎片。
- 是否每个任务都有读写集。
- 是否共享瓶颈已有 foundation / ownership stage。
- 是否每条 dependency 都有 rationale。
- 是否并行组写集不冲突。
- 是否 QA 被拆成 early preparation 和 final gate。
- 是否 reviewer 只审稳定产物。
- 是否 human-only checklist 和 human-only evidence 已分离。
- 是否 human-only 和远端副作用已隔离。
- 是否有 join gate。
- 是否能一次派发所有 ready tasks。
```

## 质量底线

- 不输出空泛“分析一下”任务。
- 不让子 agent 越权做最终决策。
- 不把未完成代码交给 reviewer 审。
- 不机械复制原方案再加标签。
- 不用 optional 逃避默认 subagent。
- 不把微小函数拆成独立 loop。
- 不新增用户没要求的目标。
- 不用兜底代码替代方案澄清。
