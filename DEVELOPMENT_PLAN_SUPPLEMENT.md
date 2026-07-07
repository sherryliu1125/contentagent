# 内容审核 Agent MVP 开发规划补充说明

## 1. 当前前提

当前公司内部已经具备以下能力：

1. 已有 VLM 读取/调用接口。
2. 已有通过 OBS 批量读取检测图片的逻辑。
3. 暂未完全明确平台侧真实字段，包括客户、站点、资源、账号、历史风险等字段。

因此，本阶段开发重点不是重建 VLM 或 OBS 批处理能力，而是将已有能力适配进统一审核 workflow，并在平台字段未完全明确的情况下，保证主流程可运行、可审计、可扩展。

## 2. 总体开发原则

MVP 必须遵守以下原则：

1. 不改变主流程。

核心链路固定为：

```text
BatchImageReader / Input
  -> VLMAdapter
  -> URLResourceClassifier
  -> PlatformContextAdapter
  -> RuleEngine
  -> AgentDecisionService
  -> EvidenceGate
  -> AuditLogger
  -> OutputBuilder
```

2. 不改变核心数据模型。

已有 VLM、OBS、平台接口只能通过 adapter 接入，不直接侵入 `RuleEngine`、`AgentDecisionService`、`EvidenceGate`。

3. VLM 结果、Agent 候选结果、最终审核结果必须分离：

```text
vlm_result.advice
agent_decision.final_action_candidate
evidence_result.final_action
output.final_action
```

4. 平台字段未明确时，不阻塞主流程。

缺失字段必须进入：

```python
platform_context.missing_fields
state.warnings
```

5. 规则层只能使用已确认存在的字段。

不得在规则中假设平台字段一定存在，也不得由 Agent 补造平台字段。

## 3. 推荐开发顺序

### 第一阶段：统一输入适配

目标：把 OBS 批量图片读取结果转换为统一审核输入。

建议接口：

```python
BatchImageReader.read_from_obs(...) -> list[ReviewInput]
```

统一输入模型：

```python
ReviewInput
```

字段包括：

```python
case_id
image
url
customer_level
site_region
resource_type
request_meta
```

限制条件：

- `image`、`url` 必须存在。
- `case_id` 可为空，系统生成。
- 平台字段允许为空。
- OBS 批处理逻辑不得直接进入 LangGraph 节点内部。

### 第二阶段：接入已有 VLM 接口

目标：用现有 VLM 能力实现统一 adapter。

建议接口：

```python
VLMAdapter.analyze_image(image_ref: str, case_id: str | None = None) -> VLMResult
```

统一输出模型：

```python
VLMResult
```

必须保留字段：

```python
advice
page_type
risk_behavior
description
raw_output
model_version
adapter_version
```

限制条件：

- 不得修改 VLM 原始字段语义。
- `risk_behavior` 不允许为 `None`，无风险时为空数组。
- VLM 失败不得默认 `pass`，应记录 warning，并由后续 Evidence Gate 转入 `need_preview`。

### 第三阶段：URL 与资源类型识别

目标：基于 URL、外部 `resource_type`、VLM `page_type` 判断资源特征。

建议接口：

```python
URLResourceClassifier.classify(
    url: str,
    resource_type: str | None,
    vlm_page_type: str | None
) -> URLClassification
```

重点输出：

```python
is_pure_image
is_website_or_landing_page
resource_type_inferred
url_features
parse_error
```

限制条件：

- URL 解析失败不导致结构化失败。
- URL 异常只能形成 weak signal，不能单独支撑 `block`。

### 第四阶段：平台上下文占位接入

目标：在真实平台字段未完全明确前，先实现可空、可扩展的上下文 adapter。

建议接口：

```python
PlatformContextAdapter.get_context(input: ReviewInput) -> PlatformContext
```

第一版可用字段：

```python
customer_level
site_region
resource_type
account_status
business_category
historical_risk
platform_labels
is_v4_v5_china_customer
missing_fields
raw_platform_payload
```

限制条件：

- 缺失字段写入 `missing_fields`。
- Agent 和规则层不得编造缺失的平台字段。
- v4/v5 中国站客户保护因子只在无明确违规时生效。
- 平台字段补齐后，只扩展 `PlatformContextAdapter` 和规则配置，不改变主 workflow。

### 第五阶段：规则层实现

目标：先基于已确定字段实现最小可用规则层。

建议接口：

```python
RuleEngine.evaluate(
    vlm_result: VLMResult | None,
    url_classification: URLClassification,
    platform_context: PlatformContext
) -> RuleResult
```

第一版规则只依赖：

```python
vlm_result
url_classification
customer_level
site_region
resource_type
```

限制条件：

- hard rule 才能支撑 `block`。
- weak signal 只能支撑 `need_preview`。
- 保护因子不能覆盖明确违规。
- 风险分只能用于排序和风险等级，不能单独制造 `block`。

### 第六阶段：Evidence Gate 实现

目标：统一约束最终动作。

建议接口：

```python
EvidenceGate.apply(state: ReviewState) -> EvidenceResult
```

限制条件：

- `final_action` 最终来源必须是 `EvidenceResult.final_action`。
- Agent 候选 `block` 但证据不足时，降级为 `need_preview`。
- Agent 候选 `pass` 但命中 hard rule 时，升级为 `block`。
- VLM 失败时不得输出默认 `pass`。
- 所有修正原因写入 `gate_corrections`。

### 第七阶段：Agent Decision

目标：先实现 deterministic mock Agent，后续再接真实 LLM。

建议接口：

```python
AgentDecisionService.decide(state: ReviewState) -> AgentDecision
```

MVP 逻辑：

```python
if rule_result.has_explicit_violation:
    return block
elif rule_result.has_weak_risk_signal:
    return need_preview
else:
    return pass
```

限制条件：

- Agent 只是候选决策，不是最终裁决。
- Agent 不得覆盖 `vlm_result`。
- Agent 不得读取未输入的平台字段。
- Agent 输出非法时必须 fallback。

### 第八阶段：审计与输出

建议接口：

```python
AuditLogger.write(state: ReviewState) -> AuditInfo
OutputBuilder.build(state: ReviewState) -> ReviewOutput
```

限制条件：

- Audit 写入失败不改变审核结果。
- 输出必须透传 warnings/errors。
- 结构化失败时 `final_action=None`。
- 正常路径下 `output.final_action` 必须来自 `evidence_result.final_action`。

## 4. 内部开发限制条件汇总

1. 不重写已有 VLM 接口，只通过 `VLMAdapter.analyze_image` 适配。
2. 不重写 OBS 批量读取逻辑，只通过 `BatchImageReader.read_from_obs` 适配。
3. 平台字段未明确前，不阻塞主流程。
4. 规则层不得依赖未确认字段。
5. Agent 不得补造平台字段。
6. VLM 原始判断不得被 Agent 或规则层覆盖。
7. `block` 必须有 hard rule 或 decisive evidence。
8. weak signal 不得直接升级为 `block`。
9. 保护因子不覆盖明确违规。
10. 所有关键判断必须可审计、可回放、可定位来源。

## 5. 当前最优落地策略

当前最优策略是：

```text
先接 OBS 输入和已有 VLM，
再用可空 PlatformContext 跑通 workflow，
第一版规则只使用已确认字段，
Evidence Gate 先做强约束，
Agent 先 mock，
平台字段明确后再扩展 adapter 和规则配置。
```

这样可以避免因为平台字段未定而阻塞开发，同时保证后续真实平台能力接入时，不需要推翻主流程。
