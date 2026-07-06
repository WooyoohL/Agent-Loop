# Agent Loop Skills

这个仓库包含两个 Codex skill，用来把复杂任务变成可拆分、可调度、可验收、可打回的 Agent 工作流。

核心目标不是“多开几个 agent”，而是解决复杂任务里最容易失真的几个问题：

- 没有逐项需求账本，却声称“基本完成”。
- 任务拆分过细，把普通函数级小改也拆成 subagent 和 reviewer。
- 主 agent 不做调度，只是派一个 subagent 后等待。
- reviewer 审查未完成代码，或者没有把问题绑定到具体 requirement。
- 最终结论依赖口头判断，而不是证据和账本。

## 包含的 Skill

### `agent-loop-controller`

主控 skill。负责把用户请求、PRD、实现方案或 QA 计划转成一个完整的 loop：

- `task_contract.md`：任务契约。
- `requirement_ledger.md`：逐项需求账本。
- `execution_plan.md`：可执行方案。
- `decomposed_plan.md`：Subagent-Ready DAG。
- `scheduler_state.md`：调度状态。
- `join_report.md`：并行任务汇合报告。
- `gate_report.md`：验收报告。
- `accept / revise / block` 决策。

最终完成度只能来自 requirement ledger，并且必须用下面四种状态汇报：

```text
PASS / FAIL / UNKNOWN / BLOCKED
```

`agent-loop-controller` 必须实际调用 `subagent-plan-decomposer`。如果 decomposer 无法加载或调用，必须停止：

```text
BLOCKED(reason=decomposer-unavailable)
```

如果 decomposer 可用，但当前环境无法实际创建 subagent，仍然必须先完成方案拆解，然后在调度阶段停止：

```text
BLOCKED(reason=subagent-runtime-unavailable)
```

### `subagent-plan-decomposer`

方案拆解 skill。负责把已有方案、PRD、implementation spec、QA plan 或 Codex 分步计划改写成主 agent 可调度的 Subagent-Ready DAG。

它会输出：

- requirement ledger。
- 模块 DAG。
- 读集和写集。
- 依赖关系和 `dependency_rationale`。
- 并行组。
- join gate。
- reviewer 时机。
- 子 agent brief。
- 调度规则。
- 合并协议。

它不执行方案、不写业务代码、不做最终验收。它只负责把方案改写到可以安全调度的形态。

## 整体流程

```text
用户请求 / PRD / 实现方案
  -> agent-loop-controller
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

主 agent 的角色是调度器，不是同步调用器。它应该一次性派发所有 ready 且写集不冲突的任务，而不是派出一个 subagent 后无理由等待。

## 关键规则

- Subagent 默认启用。
- 非微小任务必须先有明确方案。
- 方案缺少模块边界、验收证据、依赖关系、写集或用户确认点时，不得执行。
- Requirement 必须原子化，不能把多个入口、角色、负面 case 或验收点塞进一条。
- 默认按业务模块、用户路径、数据 contract 或安全边界拆分，不按普通 helper 函数拆分。
- Reviewer 只能审稳定输入，不能审尚未完成的代码。
- QA 要拆成 early preparation 和 final coverage gate。
- human-only checklist generation 和真实 human-only evidence 必须分开。
- 最终完成度不能来自信心、语气或 reviewer 的笼统评价。

## 安装

把 `skills/` 下的两个目录复制到 Codex skills 目录。

Windows 示例：

```powershell
Copy-Item -Recurse .\skills\agent-loop-controller D:\codex\skills\agent-loop-controller
Copy-Item -Recurse .\skills\subagent-plan-decomposer D:\codex\skills\subagent-plan-decomposer
```

WSL 示例：

```bash
cp -r skills/agent-loop-controller ~/.codex/skills/agent-loop-controller
cp -r skills/subagent-plan-decomposer ~/.codex/skills/subagent-plan-decomposer
```

## 仓库结构

```text
skills/
  agent-loop-controller/
    SKILL.md
    agents/openai.yaml
  subagent-plan-decomposer/
    SKILL.md
```

## 验证

当前两个 skill 已使用 `skill-creator` 的基础校验脚本验证：

```powershell
$env:PYTHONUTF8='1'
python D:\codex\skills\.system\skill-creator\scripts\quick_validate.py .\skills\agent-loop-controller
python D:\codex\skills\.system\skill-creator\scripts\quick_validate.py .\skills\subagent-plan-decomposer
```

准备发布前，Windows 与 WSL 中的 skill 文件也已做过 hash 对齐检查。
