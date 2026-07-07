# 08 Audit Logger Spec

## 1. 目标

Audit Logger 记录每条 case 的完整审核轨迹，支持误判复盘、规则迭代、离线评测和版本回放。

## 2. 审计字段

| 类别 | 字段 |
| --- | --- |
| 输入 | `case_id`、`image_ref`、`url`、输入平台字段 |
| VLM | `advice`、`page_type`、`risk_behavior`、`description`、`model_version` |
| URL | `domain`、`path`、`suffix`、`is_pure_image`、`url_features`、`parse_error` |
| 平台 | 标准化字段、`missing_fields`、裁剪后的 raw payload |
| 规则 | rule hits、risk_score、risk_level、evidence_level、rule_version |
| Agent | 候选动作、解释、引用证据、prompt_version、model_version |
| Evidence | final_action、证据链、gate_corrections |
| 异常 | `warnings`、`errors` |
| 版本 | schema、graph、rule、prompt、model、adapter、mock data |
| 时间 | started_at、finished_at、节点耗时 |

## 3. AuditInfo

```python
class AuditInfo(BaseModel):
    case_id: str
    trace_id: str
    schema_version: str
    graph_version: str
    prompt_version: str | None = None
    rule_version: str | None = None
    model_version: str | None = None
    adapter_versions: dict[str, str] = Field(default_factory=dict)
    mock_data_versions: dict[str, str] = Field(default_factory=dict)
    started_at: datetime
    finished_at: datetime | None = None
    audit_log_status: Literal["written", "failed", "skipped"] = "skipped"
    audit_log_error: str | None = None
```

## 4. 版本记录

| 版本 | 示例 |
| --- | --- |
| `schema_version` | `review-schema-0.1` |
| `graph_version` | `review-graph-0.1` |
| `rule_version` | `rules-0.1` |
| `prompt_version` | `agent-prompt-0.1` |
| `model_version` | `mock-vlm-0.1` |
| `adapter_version` | `mock-vlm-adapter-0.1` |
| `mock_data_version` | `mock-platform-data-0.1` |

## 5. 存储策略

MVP 使用 JSONL：

```text
logs/audit/review_audit_YYYYMMDD.jsonl
```

一行一条 case。

## 6. 敏感字段策略

| 数据 | 策略 |
| --- | --- |
| 图片二进制 | 不写日志，只写引用或 hash |
| URL query value | 可脱敏 |
| raw_platform_payload | 裁剪保留，敏感字段 mask |
| token、手机号、账号 | 不写明文 |

## 7. 写入失败

Audit Logger 写入失败：

- `audit_log_status = "failed"`
- `audit_log_error` 记录错误摘要。
- `warnings += ["audit_log_write_failed"]`
- 不改变 `final_action`。

