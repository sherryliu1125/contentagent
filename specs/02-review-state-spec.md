# 02 ReviewState Spec

## 1. 目的

定义 LangGraph 中共享的 `ReviewState`，包括字段分组、必填策略、节点写入责任和状态流转。

## 2. ReviewState 模型

```python
class ReviewState(BaseModel):
    schema_version: str = "review-schema-0.1"
    graph_version: str = "review-graph-0.1"
    case_id: str
    input: ReviewInput | None = None
    vlm_result: VLMResult | None = None
    url_classification: URLClassification | None = None
    platform_context: PlatformContext | None = None
    rule_result: RuleResult | None = None
    agent_decision: AgentDecision | None = None
    evidence_result: EvidenceResult | None = None
    audit_info: AuditInfo | None = None
    output: ReviewOutput | None = None
    warnings: list[str] = Field(default_factory=list)
    errors: list[str] = Field(default_factory=list)
    node_trace: list[dict[str, Any]] = Field(default_factory=list)
```

## 3. 字段分组

| 分组 | 字段 | 写入节点 |
| --- | --- | --- |
| 元信息 | `schema_version`、`graph_version`、`case_id` | Input Validation |
| 输入 | `input` | Input Validation |
| VLM | `vlm_result` | Mock VLM Adapter |
| URL 分类 | `url_classification` | URL / Resource Classifier |
| 平台上下文 | `platform_context` | Mock Platform Context Adapter |
| 规则 | `rule_result` | Rule Engine |
| Agent | `agent_decision` | Agent Decision |
| 证据 | `evidence_result` | Evidence Builder |
| 审计 | `audit_info` | Audit Logger |
| 输出 | `output` | Output Builder |
| 异常 | `warnings`、`errors` | 所有节点 |
| Trace | `node_trace` | 所有节点 |

## 4. 必填与可空策略

| 字段 | 初始是否必填 | 结束时是否必填 |
| --- | --- | --- |
| `case_id` | 是 | 是 |
| `input` | 输入合法时必填 | 正常路径必填 |
| `vlm_result` | 否 | VLM 成功时必填，失败可空 |
| `url_classification` | 否 | 输入合法时应填 |
| `platform_context` | 否 | 输入合法时应填，字段内部可空 |
| `rule_result` | 否 | 输入合法时应填，异常时 fallback |
| `agent_decision` | 否 | 输入合法时应填，非法时 fallback |
| `evidence_result` | 否 | 输入合法时应填 |
| `output` | 否 | 最终必填 |

## 5. 字段写入责任

| 节点 | 允许写入 | 不允许写入 |
| --- | --- | --- |
| Input Validation | `input`、`case_id`、`errors`、`warnings` | `vlm_result`、`final_action` |
| Mock VLM Adapter | `vlm_result`、`warnings` | `agent_decision`、`evidence_result` |
| URL Classifier | `url_classification`、`warnings` | `platform_context`、`rule_result` |
| Platform Adapter | `platform_context`、`warnings` | `rule_result`、`agent_decision` |
| Rule Engine | `rule_result`、`warnings`、`errors` | `vlm_result`、`evidence_result` |
| Agent Decision | `agent_decision`、`warnings` | `vlm_result`、`rule_result` |
| Evidence Builder | `evidence_result` | `vlm_result`、`platform_context` |
| Audit Logger | `audit_info`、`warnings` | 审核判断字段 |
| Output Builder | `output` | 上游事实字段 |

## 6. VLM 与 Agent 分离

必须保留以下层次：

```text
vlm_result.advice
agent_decision.final_action_candidate
evidence_result.final_action
output.final_action
```

禁止：

- Agent 改写 `vlm_result`。
- Output Builder 直接读取 Agent 候选动作作为最终动作。
- Evidence Builder 删除或重写 VLM 原始字段。

