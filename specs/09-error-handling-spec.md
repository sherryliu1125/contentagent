# 09 Error Handling Spec

## 1. 原则

- 输入结构错误导致结构化失败。
- 节点降级错误进入 `warnings`。
- 可恢复的内部异常进入 fallback。
- VLM 失败不得默认 `pass`。
- Audit 失败不得改变审核结果。

## 2. 异常矩阵

| 异常 | 记录 | 是否继续 | final_action |
| --- | --- | --- | --- |
| 输入缺 URL | `errors` | 否 | `None` |
| 输入缺 image | `errors` | 否 | `None` |
| VLM mock 非法结构 | `warnings` | 是 | `need_preview` |
| VLM page_type 非法 | `warnings` | 是 | `need_preview` |
| VLM risk_behavior / visual_signals 非对象 | `warnings` | 是 | `need_preview` |
| VLM 调用失败 | `warnings` | 是 | `need_preview` |
| URL 解析失败 | `warnings` | 是 | 通常 `need_preview` |
| 平台字段缺失 | `warnings` | 是 | 按证据门槛 |
| Rule Engine 异常 | `errors` + fallback | 是 | `need_preview` |
| Agent 输出非法 | `warnings` + fallback | 是 | 按 Evidence Gate |
| Evidence Gate 冲突 | `gate_corrections` | 是 | 修正后动作 |
| Audit 写入失败 | `warnings` | 是 | 不改变 |

## 3. 结构化失败

触发：

- raw request 无法解析。
- `url` 缺失。
- `image` 缺失。
- 输出模型无法构造最小错误响应。

输出：

```json
{
  "status": "structured_failed",
  "final_action": null,
  "errors": ["missing_required_input.url"]
}
```

## 4. VLM 失败

处理：

- `vlm_result = None`
- `warnings += ["vlm_failed"]`
- Rule Engine 使用 URL 和平台上下文。
- Evidence Gate 默认转 `need_preview`，除非输入结构化失败。

VLM schema 非法同样按降级处理：

- `page_type` 不在 `change.md` 允许枚举内。
- `risk_behavior` 不是对象。
- `visual_signals` 不是对象。
- 输出旧数组结构但 adapter 未显式转换。

## 5. Rule Engine fallback

```json
{
  "has_explicit_violation": false,
  "has_weak_risk_signal": true,
  "weak_signals": ["Rule Engine 异常，需人工复核"],
  "risk_level": "medium",
  "confidence": "low",
  "evidence_level": "weak"
}
```

## 6. Agent fallback

Agent 输出非法时：

- 丢弃非法输出。
- 基于 `rule_result` 生成 fallback decision。
- 设置 `is_fallback=True`。
- 继续 Evidence Gate。

## 7. Evidence Gate 冲突

冲突不算结构化失败。处理方式是修正最终动作并记录：

```text
gate_corrections
```
