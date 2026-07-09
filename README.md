# Agent Loop Skills

这是一个用于约束 Codex 长任务执行的 skill 组合。目标不是“多开几个 agent”，而是让任务有清楚的范围、证据、验收状态和可追责的结论，避免 AI 在长任务里说“基本完成”但实际漏掉关键项。

当前提供两种主控入口：

- `$agent-loop-lite`：轻量闭环。适合小到中等任务、参数实验、本地脚本、有限代码修改。它强制做 `Subagent Check`，但默认不进入完整 DAG / scheduler / reviewer 流程。
- `$agent-loop-full`：完整闭环。适合复杂 PRD、多模块并行开发、长链路验收、高风险改动。它会强制进入方案重写、模块 DAG、并行调度、join gate 和 requirement ledger 验收。

## 怎么选

| 场景 | 建议入口 |
|---|---|
| 单轮或少量步骤可以完成，但需要留证据 | `$agent-loop-lite` |
| 参数 sweep、本地实验、脚本运行、有限代码修改 | `$agent-loop-lite` |
| 用户明确要 subagent，但任务规模不大 | `$agent-loop-lite` 先做 Subagent Check |
| 已有 PRD / implementation spec，需要多模块并行执行 | `$agent-loop-full` |
| 需要 requirement ledger、DAG、join gate、独立 reviewer | `$agent-loop-full` |
| 只想把一个方案改写成 subagent 可调度结构 | `$subagent-plan-decomposer` |

## Lite 流程

`agent-loop-lite` 的核心是“保存证据，不保存仪式”：

```text
目标
  -> task.md
  -> plan.md
  -> 必填 Subagent Check
  -> 执行
  -> manifest / logs / outputs
  -> gate_summary.md
  -> PASS / FAIL / UNKNOWN / BLOCKED
```

它默认最多使用三个状态文件：

```text
loop_state/
  task.md
  plan.md
  gate_summary.md
```

Subagent Check 是必填项，但 subagent 不是必用项。只有用户明确要求，或存在独立只读审计、独立证据生成、独立 reviewer 等真实收益时才使用。

## Full 流程

`agent-loop-full` 用于更重的工程任务：

```text
用户请求 / PRD / 实现方案
  -> task_contract.md
  -> execution_plan.md
  -> requirement_ledger.md
  -> subagent-plan-decomposer
  -> decomposed_plan.md
  -> scheduler_state.md
  -> 模块级执行
  -> join_report.md
  -> gate_report.md
  -> requirement ledger 最终汇总
```

Full 版的关键约束：

- Subagent 默认启用。
- 非微小任务必须先有明确方案。
- 方案不够详细时必须打回确认，不能直接执行。
- 方案通过后必须调用 `$subagent-plan-decomposer` 改写成 Subagent-Ready DAG。
- 主 agent 是调度器，不是同步等待器；应该派发所有 ready 且写集不冲突的任务。
- Reviewer 只能审稳定输入，不能审尚未完成的代码。
- 最终完成度只能来自 `requirement_ledger.md`，不能来自口头信心。

## Subagent Plan Decomposer

`subagent-plan-decomposer` 是 full 流程里的方案重写器，也可以单独使用。它负责把 PRD、实现方案、QA plan 或 Codex 分步计划改写成可调度的 Subagent-Ready DAG。

它输出：

- requirement ledger
- 模块 DAG
- 读集和写集
- 依赖关系和 `dependency_rationale`
- 并行组
- join gate
- reviewer 时机
- 子 agent brief
- 调度规则
- 合并协议

它不执行业务代码，也不做最终验收。

## 安装

把需要的目录复制到 Codex skills 目录。

Windows 示例：

```powershell
Copy-Item -Recurse .\skills\agent-loop-lite D:\codex\skills\agent-loop-lite
Copy-Item -Recurse .\skills\agent-loop-full D:\codex\skills\agent-loop-full
Copy-Item -Recurse .\skills\subagent-plan-decomposer D:\codex\skills\subagent-plan-decomposer
```

WSL 示例：

```bash
cp -r skills/agent-loop-lite ~/.codex/skills/agent-loop-lite
cp -r skills/agent-loop-full ~/.codex/skills/agent-loop-full
cp -r skills/subagent-plan-decomposer ~/.codex/skills/subagent-plan-decomposer
```

## 仓库结构

```text
skills/
  agent-loop-lite/
    SKILL.md
    agents/openai.yaml
  agent-loop-full/
    SKILL.md
    agents/openai.yaml
  subagent-plan-decomposer/
    SKILL.md
```

## 验证

使用 `skill-creator` 的基础校验脚本：

```powershell
$env:PYTHONUTF8='1'
python D:\codex\skills\.system\skill-creator\scripts\quick_validate.py .\skills\agent-loop-lite
python D:\codex\skills\.system\skill-creator\scripts\quick_validate.py .\skills\agent-loop-full
python D:\codex\skills\.system\skill-creator\scripts\quick_validate.py .\skills\subagent-plan-decomposer
```

最终汇报必须使用明确状态：

```text
PASS / FAIL / UNKNOWN / BLOCKED
```
