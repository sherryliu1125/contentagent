# 06 Agent Decision Spec

## 1. 目标

Agent Decision Node 负责在受控上下文中输出候选动作和解释。Agent 不是最终裁决源，最终动作由 Evidence Gate 约束。

## 2. Agent 输入上下文

Agent 只允许读取：

- `vlm_result`
- `url_classification`
- `platform_context`
- `rule_result`
- `warnings` 摘要

禁止读取：

- 外部网页内容。
- 未提供的平台字段。
- 历史 case 记忆。
- 自动处置接口。

## 3. 输出 schema

```json
{
  "final_advice_candidate": "string",
  "final_action_candidate": "block|need_preview|pass",
  "risk_level_candidate": "low|medium|high|unknown",
  "confidence_candidate": "low|medium|high|unknown",
  "agent_reason": "string",
  "cited_evidence": [],
  "cited_weak_signals": [],
  "cited_protective_factors": []
}
```

## 4. Prompt 约束

Prompt 必须包含：

```text
1. 不得改写 VLM 原始字段。
2. 不得编造平台字段。
3. 不得把 weak signal 描述成明确违规。
4. final_action_candidate 只能是 block、need_preview、pass。
5. block 只能在硬规则或 decisive evidence 存在时建议。
6. 纯图片默认低风险，除非命中明确违规。
7. v4/v5 中国站客户默认保护，除非命中明确违规。
8. 证据不足时输出 need_preview。
9. 必须输出合法 JSON。
```

## 5. 防编造机制

| 风险 | 机制 |
| --- | --- |
| 编造平台字段 | 输出引用字段必须存在于 `platform_context` |
| 覆盖 VLM | Agent schema 不包含写回 VLM 的字段 |
| 过度推理 | Prompt 限制只引用输入上下文 |
| 弱信号升级 | Evidence Gate 校验 `block` 门槛 |
| 非法动作 | Pydantic 枚举校验 |

## 6. Fallback

Agent 输出非法时：

```python
if rule_result.has_explicit_violation:
    action = "block"
elif rule_result.has_weak_risk_signal or has_degrade_warning:
    action = "need_preview"
else:
    action = "pass"
```

并设置：

```python
AgentDecision(is_fallback=True)
warnings.append("agent_output_invalid")
```

## 7. MVP Agent 实现

MVP 可先实现 deterministic mock agent，不接真实 LLM：

```python
def decide(rule_result):
    if rule_result.has_explicit_violation:
        return "block"
    if rule_result.has_weak_risk_signal:
        return "need_preview"
    return "pass"
```

后续接真实 LLM 时，输入输出 schema 不变。

## 8. 不使用开放式 ReAct Agent 的原因

- 审核流程需要可回放。
- 工具调用可能引入截图外信息。
- 自主规划路径不稳定。
- 难以定位误判来自规则、VLM 还是 Agent。

