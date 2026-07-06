---
name: agent-loop-controller
description: >-
  当用户想把一次性回答、项目计划、代码任务、科研流程、产品任务或写作任务变成
  可循环执行的 Agent 工作流时使用。本技能将需求或既有方案转成 requirement ledger、
  模块级执行方案、Subagent-Ready DAG、并行调度、join gate、独立验收和
  accept/revise/block 决策，并通过 loop_state 文件持久化状态。触发词包括
  loop engineering、agent loop、门控执行、subagent、evaluator、verifier、gate、
  验收标准、迭代执行、状态化 workflow、按 PRD 执行、多 agent 并行开发。
---

# Agent Loop Controller

## 目标

把复杂任务变成一个可调度、可对账、可打回的模块级 Agent 工作流。

核心不是“让 agent 多做几轮”，而是防止三类失真：

- 没有逐项账本，却声称“基本完成”。
- 把函数级小改拆成大量 subagent 和验收，制造无意义复杂度。
- 主 agent 没有调度 DAG，派一个 subagent 就阻塞等待，或审计尚未完成的代码。

本技能采用 5 层结构：

```text
1. Controller Layer     控制目标、范围、状态和最终责任
2. Decomposer Layer     强制把方案改写成 Subagent-Ready 模块 DAG
3. Scheduler Layer      按 DAG 并行派发 ready tasks
4. Module Execution     以业务模块为单位执行和验收
5. Coverage Gate        按 requirement ledger 汇总完成度
```

## 总原则

Subagent 默认启用。不要把“是否启用 subagent”作为待判断事项。

任何非微小任务都必须先输出一个方案。方案不详细，先打回给用户确认，不能直接执行。

方案通过后，必须进入 Decomposer Layer：把方案重写成 subagent-ready 执行结构，再进入执行。

最终汇报不能说“基本完成”“差不多完成”。只能基于 requirement ledger 汇报：

```text
PASS / FAIL / UNKNOWN / BLOCKED
```

所有 `READY` 结论必须带 scope，例如 `READY(scope=dry-run-plan)`、`READY(scope=decomposition)`、`READY(scope=execution)`、`READY(scope=final-gate)`。没有真实文档、仓库路径、写权限或执行授权时，不得写 `READY(scope=execution)`。

## 必用配套 Skill

必须实际调用 `$subagent-plan-decomposer` 把方案改写成 Subagent-Ready 方案。

如果当前环境无法加载或调用 `$subagent-plan-decomposer`，必须停止并输出：

```text
BLOCKED(reason=decomposer-unavailable)
```

不得由主 agent 手工模拟 decomposer 输出，不得把“按 decomposer 结构自己写一版”当作通过 Decomposer Layer。

如果当前环境能调用 `$subagent-plan-decomposer`，但无法实际创建 subagent，仍必须先完成 decomposer 改写；随后在 Scheduler 阶段输出：

```text
BLOCKED(reason=subagent-runtime-unavailable)
```

不得把无法创建 subagent 解释为可以跳过 Decomposer Layer。

不得跳过 Decomposer Layer 直接写代码。

## 状态文件

使用：

```text
loop_state/
  task_contract.md
  requirement_ledger.md
  execution_plan.md
  decomposed_plan.md
  scheduler_state.md
  current_state.md
  delegation.md
  subagent_outputs/
  join_report.md
  gate_report.md
  decision_log.md
  artifacts/
```

状态职责：

- `task_contract.md`：目标、范围、非目标、外部副作用、人类确认点。
- `requirement_ledger.md`：逐项需求账本，是完成度唯一来源。
- `execution_plan.md`：用户可确认的执行方案。
- `decomposed_plan.md`：Subagent-Ready DAG，来自 Decomposer Layer。
- `scheduler_state.md`：ready/running/blocked/done tasks 和写集锁。
- `current_state.md`：当前事实看板，必须短而新。
- `delegation.md`：当前 subagent brief 和写集。
- `join_report.md`：模块汇合结果。
- `gate_report.md`：验收结论。
- `decision_log.md`：append-only 决策日志。

`current_state.md` 必须最后更新。每轮开始和最终汇报前，检查它是否落后于最新 `decision_log`、`gate_report`、`artifacts`。

## Layer 1：Controller

Controller 只做全局控制，不把完整实现吞进主 agent。

### 1.1 写任务契约

创建或更新 `task_contract.md`：

- 用户原始请求。
- 真实目标。
- 范围。
- 非目标。
- 验收标准。
- 可接受证据。
- 风险等级。
- 外部副作用授权：本地编辑、远端写入、部署、真实用户数据、删除或清理。
- human-only 项：真机、真实账号、内容签收、审核、发布、回滚、正式数据清理。
- 需要用户确认的位置。

### 1.2 强制输出执行方案

必须先给出一份可执行方案，写入 `execution_plan.md`。

方案至少包含：

- 需求来源和优先级。
- 模块划分。
- 每个模块的业务目标。
- 每个模块的输入、输出、依赖。
- 每个模块的允许写集和禁止事项。
- 每个模块的验收标准。
- 哪些证据能证明完成。
- 哪些事项必须由用户确认。
- 明确不做什么。

如果方案缺少模块边界、验收证据、依赖关系、写集、或用户必须确认的决策，结论必须是：

```text
block: plan-insufficient
```

然后向用户确认缺失项。不能边写代码边补方案。

### 1.3 建 requirement ledger

当用户提供 PRD、implementation spec、QA plan、详细任务清单或既有方案时，必须先建立 `requirement_ledger.md`。

每条记录包含：

```text
requirement_id
source_doc / source_message
requirement_text
owner_module
status: PASS / FAIL / UNKNOWN / BLOCKED
implementation_evidence
verification_evidence
human_only_reason
notes
```

规则：

- 无证据不得 PASS。
- 自动可修复缺口不得标为 BLOCKED。
- 真机、签收、审批、真实账号、正式数据操作才能标为 BLOCKED。
- 没检查到就是 UNKNOWN，不要写“基本满足”。
- Requirement 必须原子化。若一句需求包含多个 `和/或`、多个负面 case、多个入口、多个角色或多个验收点，必须拆成多条 requirement。

## Layer 2：Decomposer

Decomposer 是强制层。它把 `execution_plan.md` 改写为 `decomposed_plan.md`。

必须使用 `$subagent-plan-decomposer` 的逻辑：

- 标记 `MAIN_AGENT_ONLY`。
- 标记 `SUBAGENT_REQUIRED`。
- 标记 `SPLIT_TO_SUBAGENTS`。
- 标记 `PARALLEL_GROUP`。
- 标记 `JOIN_GATE`。
- 标记 `DEPENDENCY`。
- 标记 `REVIEW_AGENT`。

Decomposer 输出必须通过以下门禁，否则不得进入 Scheduler：

- 每个 requirement 已原子化。
- 每条依赖都有 rationale，说明为什么必须等待。
- 所有共享瓶颈文件或资源已形成 shared foundation / ownership stage。
- QA 被拆成 early test-plan / fixture / guardrail 准备和 final coverage gate。
- human-only checklist 任务与 human-only evidence requirement 分开。
- 每个 `SUBAGENT_REQUIRED` task 都有完整 brief；不能只给示例。
- 调度表必须包含 `dependency_rationale` 字段；没有该字段时视为 decomposer 输出不合格。
- human-only readiness checklist 的“生成任务”不得标成 `HUMAN_ONLY`；它应是 `MAIN_AGENT_ONLY` 或 `SUBAGENT_REQUIRED`。只有真实证据采集项才标 `HUMAN_ONLY` / `BLOCKED`。

### 2.1 拆分粒度规则

默认按业务模块、用户路径、数据契约或安全边界拆分。

禁止默认按函数、局部 helper、小脚本、小 UI 片段拆分。

合格模块示例：首次试玩、报告房间生命周期、分享卡/海报、数据权利、内容题库、集成 QA。  
不合格拆分示例：给 5 行函数单独派 worker、给普通空值判断单独派 reviewer、为局部 helper 写大量与用户路径无关的兜底测试。

除非该小函数是安全、加密、权限、删除、支付、发布等高风险边界，否则不得为微小实现单独开 loop。

### 2.2 Subagent brief 必须完整

每个 subagent brief 必须包含：角色、目标、输入范围、允许读集、允许写集、禁止事项、输出格式、验收标准、依赖项、可并行条件、是否允许外部副作用、压缩等级。

brief 写不清，就不能派发。

### 2.3 Reviewer 规则

Reviewer 只能审稳定输入：

- 已冻结方案。
- 已完成模块。
- 已生成 artifact。
- 已完成测试计划。

不得审查尚未写出来或正在写的实现。

Reviewer 的 revise 必须绑定以下至少一项：

- requirement。
- 用户路径。
- 数据 contract。
- 安全 / 隐私 / 权限风险。
- 发布或破坏性操作风险。

不得因为纯理论输入组合要求增加兜底代码。

## Layer 3：Scheduler

Scheduler 按 `decomposed_plan.md` 派发任务，写入 `scheduler_state.md`。

主 agent 不是同步调用器，而是调度器。

每轮调度步骤：

1. 读取 DAG。
2. 找出所有依赖已满足的 ready tasks。
3. 排除写集冲突、外部副作用冲突、输入未稳定的任务。
4. 一次性派发所有可并行 ready tasks。
5. 没有 ready task 时才等待 running tasks。
6. 每个 join gate 完成后，重新计算 ready tasks，继续派发。

禁止：

- 派一个 subagent 后无理由干等。
- 审计尚未完成的代码。
- 让 reviewer 和 implementation worker 同时改同一文件。
- 在写集冲突未解除时并行执行。
- 把“提高并行度”理解成对未产出的代码做审查。

允许并行：

- 不同模块实现，且写集隔离。
- 内容审查、文档对照、测试计划、fixture 准备。
- 方案审查和实现无依赖的静态检查。
- 一个模块完成后，独立 reviewer 审该模块，同时其他无冲突模块继续实现。

如果存在共享瓶颈文件，例如公共 client service、adapter、cloudfunction entry、schema 或 action registry，Scheduler 必须先派 shared foundation / ownership 任务；其他模块不得并行修改这些瓶颈，除非 decomposed plan 已划分明确接入区。

Shared foundation 要拆成两层：`MAIN_AGENT_ONLY` 负责冻结 ownership、接口边界和写集规则；若需要写代码实现 foundation，必须是独立 `SUBAGENT_REQUIRED` worker，除非用户明确要求主 agent 亲自实现。

## Layer 4：Module Execution

执行以模块为单位，不以函数为单位。

每个模块必须有：module id、owner requirement ids、输入、输出、touched files、validation commands、artifacts、reviewer、scope boundary。

模块 worker 输出必须写到：

```text
loop_state/artifacts/<module-id>-implementation-results.md
```

输出格式：

```text
Conclusion: accept-candidate / revise-needed / blocked
Scope:
Requirements covered:
Files touched:
Implementation summary:
Verification:
Known gaps:
Out-of-scope failures:
No-go claims:
```

### 4.1 复杂度预算

每个模块在执行前声明预算：预期改动文件数、预期代码规模、预期测试深度、是否需要 reviewer。

明显超预算时，worker 必须停止并报告：

```text
block: module-scope-expanded
```

不要连续写数小时后再声称完成。

### 4.2 防过度兜底

新增兜底、防御、fallback、兼容逻辑必须说明：保护哪个 requirement、服务哪个真实用户路径、避免哪个明确风险、是否改变业务语义。

不能为了让 reviewer 满意而堆代码。

## Layer 5：Coverage Gate

最终完成度只来自 `requirement_ledger.md`。

模块完成后，更新相关 requirement：

- `PASS`：有实现证据和验证证据。
- `FAIL`：明确未满足。
- `UNKNOWN`：未检查或证据不足。
- `BLOCKED`：依赖 human-only 或外部条件。

### 5.1 Join gate

每个并行组或模块组必须有 join gate：

- 等待哪些输出。
- 哪些 requirement 被覆盖。
- 哪些 requirement 未覆盖。
- 冲突如何处理。
- 是否允许进入下一阶段。

### 5.2 Gate 决策

只允许：

```text
accept
revise
block
```

可附加 scope：

```text
final | module | local | readiness | artifact | partial
```

`accept` 必须说明接受范围。

`revise` 必须指出返工 requirement 和返工模块。

`block` 必须说明阻塞原因和需要用户做什么。

### 5.3 禁止性结论

禁止使用：

- 基本完成。
- 差不多完成。
- 大部分完成。
- 应该没问题。
- 只剩小问题。

必须使用：

```text
PASS: x
FAIL: x
UNKNOWN: x
BLOCKED: x
```

并列出 FAIL / UNKNOWN / BLOCKED 的 requirement id。

## Revise 与 Regate

高风险 revise 必须 regate：

- 安全。
- 隐私。
- 权限。
- 数据权利。
- 删除、退出、清理、迁移。
- 远端部署。
- 发布 readiness。
- 用户主路径。

Regate 必须检查：原失败项、返工证据、新验证、是否引入新风险、ledger 是否更新。

没有 regate，不得把高风险 revise 改成 accept。

## Human-only 与外部副作用

默认 human-only：真机微信群、真实账号、真实 `OPENID/opengid`、内容人工签收、微信审核材料、发布、回滚、正式清理、正式数据删除/迁移/修复。

默认需要用户明确授权：远端 secret、部署、正式数据库写入、权限规则变更、删除或清理、真实用户数据读取。

mock、多用户模拟、DevTools smoke、local test、code-only deploy、readiness readback 都不能替代 human-only 证据。

human-only readiness checklist 可以由 agent 自动生成；human-only evidence 必须由人类或真实外部环境提供。二者必须分开建 task / requirement。

Checklist generation 不得被标为 blocked；只有需要真实人类或真实环境提供的 evidence requirement 才能标为 BLOCKED。

## 状态更新顺序

固定顺序：

```text
subagent output / artifact
-> join_report
-> gate_report
-> decision_log
-> requirement_ledger
-> scheduler_state
-> current_state
```

`current_state.md` 必须回答：

- 当前阶段。
- 当前 DAG 状态。
- ready tasks。
- running tasks。
- blocked tasks。
- accepted modules。
- failed requirements。
- unknown requirements。
- next dispatch。
- 是否需要用户输入。

## 用户汇报格式

每轮汇报：

```text
当前阶段：
调度状态：
已派发：
已完成模块：
Ledger：
  PASS:
  FAIL:
  UNKNOWN:
  BLOCKED:
需要返工：
阻塞项：
下一步：
状态文件：
```

最终汇报：

```text
结论：
完成度账本：
模块结果：
未完成项：
未验证项：
human-only 项：
证据位置：
不代表什么：
```

## 质量底线

- 没有详细方案，不执行。
- 没有 Decomposer Layer，不执行。
- 没有 requirement ledger，不声称完成。
- Subagent 默认启用。
- 主 agent 必须调度所有 ready 且不冲突的任务。
- Reviewer 不审未完成代码。
- 默认模块级拆分，不做函数级 loop。
- Revise 必须服务 requirement、用户路径、contract 或真实风险。
- 不用大量兜底代码掩盖方案不清。
- 不把局部通过、local-only、readiness、mock、scout 当作最终完成。
- 不把自动可修复缺口藏成人类阻塞。

## 边界

本 skill 定义的是控制协议。实际 subagent 创建、并行执行、远端部署、真实测试和 token 成本统计，取决于当前环境可用工具和用户授权。
