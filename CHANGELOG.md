# 更新日志

## 2026-07-06 - Agent Loop Skills 重构版

本次更新基于旧版 `agent-loop-controller.zip` 做重构。旧版偏向“通用循环协议 + 可选配套 skill”，新版收紧为“`agent-loop-controller` + `subagent-plan-decomposer`”两件套：先建需求账本，再强制拆成 Subagent-Ready DAG，最后按账本验收。

## 总体变化

### 从可选委派改为强制拆解

旧版中，`subagent-plan-decomposer` 是“可引用的配套 Skill”，只有在任务受益于 subagent、并行、上下文隔离或 reviewer 时才调用。

新版改为硬门禁：

```text
必须实际调用 $subagent-plan-decomposer
```

如果无法加载或调用 decomposer，直接停止：

```text
BLOCKED(reason=decomposer-unavailable)
```

如果 decomposer 可用但当前环境无法创建 subagent，必须先完成 decomposer 改写，再在调度阶段停止：

```text
BLOCKED(reason=subagent-runtime-unavailable)
```

### 从“计划 + gate”改为“ledger + DAG + gate”

旧版主要流程是：

```text
需求 -> 任务契约 -> 规划 -> 委派 -> 执行 -> 汇合 -> 门控验收 -> 决策 -> 状态更新
```

新版改为：

```text
需求 / PRD / 方案
  -> task_contract.md
  -> execution_plan.md
  -> requirement_ledger.md
  -> decomposed_plan.md
  -> scheduler_state.md
  -> module execution
  -> join_report.md
  -> gate_report.md
  -> ledger summary
```

最终完成度只能来自 `requirement_ledger.md`，不再允许用“基本完成”“差不多完成”“只剩小问题”这类口头结论。

### 从主 agent 同步等待改为调度器模型

旧版强调 subagent 委派和 join gate，但没有足够明确的调度规则。

新版明确主 agent 是 scheduler：

- 读取 DAG。
- 找出所有依赖已满足的 ready tasks。
- 排除写集冲突、外部副作用冲突和输入未稳定的任务。
- 一次性派发所有可并行 ready tasks。
- 没有 ready task 时才等待 running tasks。
- 每个 join gate 完成后重新计算 ready tasks。

## `agent-loop-controller` 主要更新

### 新增 5 层结构

新版 controller 明确采用 5 层：

```text
1. Controller Layer
2. Decomposer Layer
3. Scheduler Layer
4. Module Execution
5. Coverage Gate
```

这把“控制目标、方案拆解、并行调度、模块执行、最终验收”分开，减少主 agent 自己写、自己审、自己宣布完成的问题。

### 强制先输出执行方案

非微小任务必须先写 `execution_plan.md`。方案至少包含：

- 需求来源和优先级。
- 模块划分。
- 每个模块的业务目标。
- 输入、输出、依赖。
- 允许写集和禁止事项。
- 验收标准。
- 完成证据。
- 用户必须确认的位置。
- 明确不做什么。

如果缺少模块边界、验收证据、依赖关系、写集或用户必须确认的决策，结论必须是：

```text
block: plan-insufficient
```

### 新增 requirement ledger 作为唯一完成度来源

新版要求建立 `requirement_ledger.md`，每条 requirement 包含：

- requirement id。
- 来源。
- 原子化需求文本。
- owner module。
- 状态：`PASS / FAIL / UNKNOWN / BLOCKED`。
- 实现证据。
- 验证证据。
- human-only 原因。

没有证据不得 `PASS`。没检查到就是 `UNKNOWN`。自动可修复缺口不得藏成 `BLOCKED`。

### 明确禁止函数级 loop

新版默认按业务模块、用户路径、数据 contract 或安全边界拆分。

不再允许把普通 5 行 helper、空值判断、小 UI 片段单独拆成 worker 和 reviewer，除非它本身处在安全、权限、删除、支付、发布等高风险边界。

### 新增模块复杂度预算

每个模块执行前要声明预算：

- 预期改动文件数。
- 预期代码规模。
- 预期测试深度。
- 是否需要 reviewer。

明显超预算时，worker 必须停止并报告：

```text
block: module-scope-expanded
```

### 加强 high-risk revise / regate

安全、隐私、权限、数据权利、删除、迁移、远端部署、发布 readiness、用户主路径等高风险返工后必须 regate。

没有 regate，不得把高风险 revise 改成 accept。

### 拆分 human-only checklist 和 human-only evidence

新版明确：

- readiness checklist 可以由 agent 生成。
- 真机、真实账号、签收、审批、发布、正式数据操作等 evidence 必须由人类或真实外部环境提供。
- checklist generation 不得标为 blocked。
- 只有真实 evidence requirement 才能标为 `BLOCKED`。

## `subagent-plan-decomposer` 主要更新

### 从“方案标注器”强化为工程级 DAG 拆解器

新版 decomposer 不只是给原方案加标签，而是必须输出：

- `Requirement Ledger`
- `模块 DAG`
- `调度表`
- `改写后的 Subagent-Ready 方案`
- `子 Agent 任务简报`
- `Reviewer 计划`
- `Join / 合并协议`
- `主 Agent 调度检查清单`

### 新增输入充分性 Gate

方案不够详细时，不生成伪 DAG，而是输出：

```text
Decision: PLAN_INSUFFICIENT
```

并列出必须由用户确认的缺口。

### 强化 requirement 原子化

如果一句需求包含多个入口、角色、负面 case、`和/或` 条件或多个验收点，必须拆成多条 requirement。

例如不能用一条 requirement 覆盖：

```text
外群 / 单聊 / 历史入口 / URL 篡改 / 已退出成员
```

### 强制读写集和 dependency rationale

每个任务必须声明：

- 输入范围。
- 允许读集。
- 允许写集。
- 禁止写集。
- 共享瓶颈文件或资源。
- 是否允许外部副作用。
- 是否需要用户确认。

调度表必须包含 `dependency_rationale`。只写 `deps` 或箭头不合格。

### 新增 shared foundation / ownership stage

如果存在公共 client service、adapter、cloudfunction entry、schema、action registry 等共享瓶颈，必须先建立 shared foundation / ownership stage。

shared foundation 拆成两层：

- `MAIN_AGENT_ONLY`：冻结 ownership、接口边界、写集规则。
- `SUBAGENT_REQUIRED` foundation worker：需要实际代码改动时单独执行。

### 明确 reviewer 时机

Reviewer 只能审稳定输入：

- 已冻结方案。
- 已完成模块。
- 已生成 artifact。
- 已完成测试计划。

不得审查尚未写出或正在写的代码。

### QA 拆成 early preparation 和 final gate

新版要求 QA 拆成两类：

- early QA plan / fixture / guardrail 准备。
- final coverage gate。

不能把所有 QA 都放到最后。

## 移除或不再作为主流程的内容

旧版 `agent-loop-controller.zip` 中还包含：

- `ai-opposes-ai`
- `research-target-reframing`
- `ai-research-orchestrator`

新版发布版暂不把这些纳入主流程。原因是本次核心目标是解决代码任务中的拆分、调度、验收可信度问题，而不是扩展研究型或反方审查型 workflow。

当前主流程只保留两个 skill：

```text
agent-loop-controller
subagent-plan-decomposer
```

其中 `subagent-plan-decomposer` 是硬依赖。

## 迁移说明

旧版使用者迁移时应注意：

- `plan.md` 的职责现在由 `execution_plan.md` 承担。
- 新增 `requirement_ledger.md`，最终完成度必须从这里汇总。
- 新增 `decomposed_plan.md`，来自 `subagent-plan-decomposer`。
- 新增 `scheduler_state.md`，记录 ready/running/blocked/done tasks 和写集锁。
- 不再允许主 agent 手工模拟 decomposer 输出。
- 不再使用“基本完成”作为最终汇报。
- 模块执行结果应写入 `loop_state/artifacts/<module-id>-implementation-results.md`。

## 兼容性说明

这是一次流程语义重构，不是旧版的轻量补丁。

对旧版 prompt 或状态文件最可能产生影响的地方：

- 旧版可选 subagent 逻辑会被新版强制 decomposer 替代。
- 旧版按 `plan.md` 验收的流程需要迁移到 `requirement_ledger.md`。
- 旧版泛化 gate 需要迁移到 module / join / coverage gate。
- 旧版没有 `dependency_rationale` 的调度表需要补齐。

## 当前发布内容

```text
README.md
CHANGELOG.md
skills/agent-loop-controller/SKILL.md
skills/agent-loop-controller/agents/openai.yaml
skills/subagent-plan-decomposer/SKILL.md
```

仓库中保留的 `agent-loop-controller.zip` 是旧版归档，不是当前 active source。
