# 01 Interface Spec

## 1. 目的

定义内容审核 Agent MVP 的输入输出、Pydantic 模型、枚举限制和空值策略。本文档是模型实现和接口联调的依据。

## 2. 枚举

### 2.1 final_action

```python
class FinalAction(str, Enum):
    BLOCK = "block"
    NEED_PREVIEW = "need_preview"
    PASS = "pass"
```

不得增加 `auto_ban`、`takedown`、`reject` 等动作。

### 2.2 风险与证据枚举

```python
class RiskLevel(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    UNKNOWN = "unknown"

class Confidence(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    UNKNOWN = "unknown"

class EvidenceLevel(str, Enum):
    NONE = "none"
    WEAK = "weak"
    DECISIVE = "decisive"
    UNKNOWN = "unknown"
```

### 2.3 VLM advice

必须与 `online-prompt.txt` 对齐：

```python
class VLMAdvice(str, Enum):
    GOOD = "good"
    CHILD_PORN = "child_porn"
    PORN = "porn"
    GAMBLE = "gamble"
    FINANCE = "finance"
    FAKE = "fake"
    OTHER_FRAUD = "other_fraud"
    POLITIC = "politic"
    GAME = "game"
    VPN = "vpn"
```

### 2.4 page_type

必须与 `change.md` 和 `specs/12-page-type-spec.md` 对齐：

```python
class PageType(str, Enum):
    MALL = "mall"
    PAYMENT = "payment"
    REBATE = "rebate"
    LOGIN = "login"
    INVESTMENT = "investment"
    CRYPTO = "crypto"
    DIGITAL_GOODS = "digital_goods"
    VPN = "vpn"
    PORN = "porn"
    GAMBLING = "gambling"
```

不得输出 `login_auth`、`gambling_lottery`、`porn_dating`、`vpn_proxy`、`brand_impersonation`、`other_unknown` 等旧值。

## 3. ReviewInput

```python
class ReviewInput(BaseModel):
    case_id: str | None = None
    image: str
    url: str
    customer_level: str | None = None
    site_region: str | None = None
    resource_type: str | None = None
    request_meta: dict[str, Any] = Field(default_factory=dict)
```

| 字段 | 必填 | 空值策略 |
| --- | --- | --- |
| `case_id` | 否 | 空则系统生成 |
| `image` | 是 | 为空则结构化失败 |
| `url` | 是 | 为空则结构化失败 |
| `customer_level` | 否 | 空值进入平台缺失字段 |
| `site_region` | 否 | 空值进入平台缺失字段 |
| `resource_type` | 否 | 可由 URL 和 VLM page_type 推断 |

## 4. VLMResult

```python
class VLMResult(BaseModel):
    advice: VLMAdvice
    page_type: PageType
    risk_behavior: dict[str, bool] = Field(default_factory=dict)
    visual_signals: dict[str, bool | list[str]] = Field(default_factory=dict)
    description: str
    raw_output: dict[str, Any] | None = None
    model_version: str = "mock-vlm-0.1"
    adapter_version: str = "mock-vlm-adapter-0.1"
```

约束：

- `advice`、`page_type`、`risk_behavior`、`visual_signals`、`description` 必须保留原始语义。
- Agent 不得覆盖此模型。
- `risk_behavior` 必须是对象，不允许为 `None`，不得再使用数组。
- `visual_signals` 必须是对象，不允许为 `None`。
- `risk_behavior` 和 `visual_signals` 只能使用 `change.md` 已定义字段；`payment` 当前字段待补充，可输出空对象。
- 未命中的布尔字段应输出 `false`；数组字段无内容时输出空数组。

## 5. URLClassification

```python
class URLClassification(BaseModel):
    has_url: bool
    normalized_url: str | None = None
    domain: str | None = None
    path: str | None = None
    suffix: str | None = None
    query_keys: list[str] = Field(default_factory=list)
    is_pure_image: bool = False
    is_website_or_landing_page: bool = False
    resource_type_inferred: str | None = None
    url_features: list[str] = Field(default_factory=list)
    parse_error: str | None = None
```

URL 解析失败不导致结构化失败，但要记录 `parse_error`。

## 6. PlatformContext

```python
class PlatformContext(BaseModel):
    customer_level: str | None = None
    site_region: str | None = None
    resource_type: str | None = None
    account_status: str | None = None
    business_category: str | None = None
    historical_risk: str | None = None
    platform_labels: list[str] = Field(default_factory=list)
    is_v4_v5_china_customer: bool = False
    missing_fields: list[str] = Field(default_factory=list)
    raw_platform_payload: dict[str, Any] | None = None
    adapter_version: str = "mock-platform-adapter-0.1"
    mock_data_version: str = "mock-platform-data-0.1"
```

平台字段缺失时不得由 Agent 补造。

## 7. RuleResult

```python
class RuleHit(BaseModel):
    rule_id: str
    rule_name: str
    rule_type: Literal["hard", "weak", "protective"]
    evidence: str
    source: Literal["vlm", "url", "platform", "combined"]
    severity: Literal["low", "medium", "high"]

class RuleResult(BaseModel):
    hard_rule_hits: list[RuleHit] = Field(default_factory=list)
    weak_signal_hits: list[RuleHit] = Field(default_factory=list)
    protective_factor_hits: list[RuleHit] = Field(default_factory=list)
    has_explicit_violation: bool = False
    has_weak_risk_signal: bool = False
    decisive_evidence: list[str] = Field(default_factory=list)
    weak_signals: list[str] = Field(default_factory=list)
    protective_factors: list[str] = Field(default_factory=list)
    risk_score: int = Field(default=0, ge=0, le=100)
    risk_level: RiskLevel = RiskLevel.UNKNOWN
    confidence: Confidence = Confidence.UNKNOWN
    evidence_level: EvidenceLevel = EvidenceLevel.UNKNOWN
    rule_version: str = "rules-0.1"
```

## 8. AgentDecision

```python
class AgentDecision(BaseModel):
    final_advice_candidate: str
    final_action_candidate: FinalAction
    risk_level_candidate: RiskLevel
    confidence_candidate: Confidence
    agent_reason: str
    cited_evidence: list[str] = Field(default_factory=list)
    cited_weak_signals: list[str] = Field(default_factory=list)
    cited_protective_factors: list[str] = Field(default_factory=list)
    prompt_version: str = "agent-prompt-0.1"
    model_version: str = "mock-agent-0.1"
    is_fallback: bool = False
```

## 9. EvidenceResult

```python
class EvidenceResult(BaseModel):
    final_advice: str
    final_action: FinalAction
    risk_level: RiskLevel
    confidence: Confidence
    evidence_level: EvidenceLevel
    decisive_evidence: list[str] = Field(default_factory=list)
    weak_signals: list[str] = Field(default_factory=list)
    protective_factors: list[str] = Field(default_factory=list)
    agent_reason: str
    gate_corrections: list[str] = Field(default_factory=list)
```

## 10. ReviewOutput

```python
class ReviewOutput(BaseModel):
    case_id: str
    status: Literal["ok", "need_preview_by_degrade", "structured_failed"]
    vlm_result: VLMResult | None = None
    final_advice: str | None = None
    final_action: FinalAction | None = None
    risk_level: RiskLevel = RiskLevel.UNKNOWN
    confidence: Confidence = Confidence.UNKNOWN
    evidence_level: EvidenceLevel = EvidenceLevel.UNKNOWN
    agent_reason: str | None = None
    decisive_evidence: list[str] = Field(default_factory=list)
    weak_signals: list[str] = Field(default_factory=list)
    protective_factors: list[str] = Field(default_factory=list)
    audit: dict[str, Any] | None = None
    warnings: list[str] = Field(default_factory=list)
    errors: list[str] = Field(default_factory=list)
```

结构化失败时 `final_action=None`，不返回三值动作。
