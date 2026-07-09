# 07 Evidence Gate Spec

## 1. 目标

Evidence Gate 是 `final_action` 的最终约束层。它接收 Rule Engine 和 Agent Decision 的结果，输出最终 `EvidenceResult`。

## 2. 输入

- `rule_result`
- `agent_decision`
- `vlm_result`
- `warnings`
- `errors`

## 3. 输出

- `final_advice`
- `final_action`
- `risk_level`
- `confidence`
- `evidence_level`
- `decisive_evidence`
- `weak_signals`
- `protective_factors`
- `agent_reason`
- `gate_corrections`

## 4. 证据门槛

| final_action | 门槛 |
| --- | --- |
| `block` | 必须存在 hard rule 或 decisive evidence |
| `need_preview` | 可由 weak signal、关键节点失败、证据不足触发 |
| `pass` | 无明确违规，且无显著弱信号，或保护因子成立 |

`Evidence Gate` 不直接解释 `risk_behavior` 或 `visual_signals` 的业务含义；这些含义必须先由 Rule Engine 转换为 hard / weak / protective 命中。Evidence Gate 只负责执行最终证据门槛。

## 5. 修正规则

| Agent 候选 | 证据状态 | 最终动作 |
| --- | --- | --- |
| `block` | 无 hard rule，无 decisive evidence | `need_preview` |
| `pass` | hard rule 命中 | `block` |
| `pass` | VLM 失败且 URL 可疑 | `need_preview` |
| `block` | 只有 weak signals | `need_preview` |
| `pass` | 纯图片，无明确违规 | `pass` |
| `pass` | v4/v5 中国站客户，无明确违规 | `pass` |

## 6. 伪代码

```python
def apply_evidence_gate(state):
    if state.errors and has_structured_error(state.errors):
        return structured_failed()

    rule = state.rule_result
    agent = state.agent_decision
    corrections = []

    if rule.has_explicit_violation or rule.decisive_evidence:
        return block(rule, agent, corrections)

    if agent.final_action_candidate == "block":
        corrections.append("block_downgraded_due_to_insufficient_evidence")
        return need_preview(rule, agent, corrections)

    if has_vlm_failure(state.warnings):
        return need_preview(rule, agent, corrections)

    if rule.has_weak_risk_signal:
        return need_preview(rule, agent, corrections)

    return pass_result(rule, agent, corrections)
```

## 7. 保护因子规则

保护因子只在无明确违规时生效：

```python
if rule.has_explicit_violation:
    ignore_protective_pass()
```

## 8. gate_corrections

修正原因必须记录，例如：

- `agent_block_downgraded_to_need_preview_due_to_weak_evidence`
- `agent_pass_upgraded_to_block_due_to_hard_rule`
- `vlm_failure_forced_need_preview`

## 9. page_type 字段约束

当 VLM 输出出现以下问题时，Evidence Gate 不得默认 `pass`：

- `page_type` 不在 `change.md` 允许枚举内。
- `risk_behavior` 不是对象。
- `visual_signals` 不是对象。
- 关键布尔字段无法判断且 Rule Engine 已记录 schema warning。

这些情况通常应输出 `need_preview`，除非输入结构化失败需要返回 `final_action=None`。
