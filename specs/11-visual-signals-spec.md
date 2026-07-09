# 11 Visual Signals Spec

## 1. 目的

本文档定义 `VisualSignals` 的设计骨架，用于补足 `VLMResult.advice`、`VLMResult.page_type`、`VLMResult.risk_behavior` 颗粒度不足的问题。

`VisualSignals` 的目标不是让 VLM 直接做最终审核判断，而是把截图中可见的视觉事实结构化，供 `RuleEngine` 组合判断。

核心原则：

```text
VLM / VisualSignals 负责：看到什么
RuleEngine 负责：这些信号按规则意味着什么
AgentDecision 负责：基于已有证据给出候选解释
EvidenceGate 负责：最终 final_action
```

因此，`VisualSignals` 不得直接输出 `final_action`，不得替代 `RuleResult`，也不得替代 `EvidenceResult`。

## 2. 与现有 Spec 的关系

本文档作为独立补充 spec 存在，不重排 `specs/01-10` 的编号。

短期内，`VisualSignals` 可以先作为规则设计文档和后续开发依据，不强制阻塞 Phase 2 基础框架开发。

后续接入代码时，建议修改点如下：

| Spec | 后续建议修改 |
| --- | --- |
| `01-interface-spec.md` | 新增 `VisualSignals`、`SignalEvidence` 等 Pydantic 模型 |
| `02-review-state-spec.md` | 在 `ReviewState` 中新增 `visual_signals: VisualSignals | None` |
| `03-langgraph-workflow-spec.md` | 在 `VLM Adapter` 后增加 `visual_signal_extractor` 节点，或说明由 VLM Adapter 一次性返回 |
| `04-node-spec.md` | 新增 `Visual Signal Extractor Node` 的输入、输出、异常和测试重点 |
| `05-rule-engine-spec.md` | 说明 `RuleEngine` 如何消费 `visual_signals` 生成 hard / weak / protective 命中 |
| `10-test-spec.md` | 增加视觉信号提取和场景规则测试 case |

## 3. 接入方式

`VisualSignals` 有两种实现方式：

### 3.1 一次 VLM 输出

VLM Adapter 在一次模型调用中同时返回：

```text
VLMResult
VisualSignals
```

优点：

- 延迟低。
- 不需要二次模型调用。
- 适合 MVP。

限制：

- 需要调整 VLM prompt 和输出 schema。
- 字段过多时，需要控制输出长度。

### 3.2 独立 Visual Signal Extractor

在 `VLMResult` 后增加逻辑节点：

```text
VLMResult -> VisualSignalExtractor -> VisualSignals
```

优点：

- 架构边界清晰。
- 后续可以独立优化视觉信号提取。

限制：

- 如果需要再次调用模型，会增加延迟。

MVP 建议：

```text
逻辑上保留 VisualSignalExtractor 概念；
实现上优先允许 VLM Adapter 一次性返回 VisualSignals；
不得为了 VisualSignals 强制增加二次模型调用。
```

## 4. 字段设计原则

`VisualSignals` 字段必须满足：

1. 只记录截图中可见或可读文本中的事实。
2. 不直接判断最终违规动作。
3. 不编造截图中不存在的信息。
4. 不从平台字段、历史 case、外部网页内容中补充信息。
5. 不确定时应使用 `unknown` 或写入 `extraction_warnings`。
6. 关键布尔信号应尽量附带 `evidence_texts`，便于人工复核。

不推荐把 prompt 写成：

```text
如果 page_type == xxx，则从以下关键词中选择...
```

推荐方式是：

```text
固定一组通用视觉信号字段；
RuleEngine 再用 page_type + VisualSignals + URLClassification + PlatformContext 组合判断。
```

## 5. 建议数据模型

以下为建议 schema，后续可在 `01-interface-spec.md` 中落地为 Pydantic 模型。

```python
class SignalEvidence(BaseModel):
    field: str
    value: str | bool | None
    evidence_texts: list[str] = Field(default_factory=list)
    confidence: Confidence = Confidence.UNKNOWN
```

```python
class VisualSignals(BaseModel):
    # language and text
    primary_language: str | None = None
    has_language_switch: bool | None = None
    supported_languages: list[str] = Field(default_factory=list)
    visible_keywords: list[str] = Field(default_factory=list)
    evidence_texts: list[str] = Field(default_factory=list)

    # brand and impersonation
    visible_brand_names: list[str] = Field(default_factory=list)
    has_official_claim: bool | None = None
    has_certification_or_badge: bool | None = None
    impersonated_brand_suspected: bool | None = None

    # login and identity
    has_login_form: bool | None = None
    asks_for_credentials: bool | None = None
    asks_for_real_name: bool | None = None
    asks_for_bank_card: bool | None = None
    has_invite_code_field: bool | None = None
    registration_requires_invite_code: bool | None = None

    # payment and transaction
    has_payment_entry: bool | None = None
    has_recharge_entry: bool | None = None
    has_withdraw_entry: bool | None = None
    visible_currency_symbols: list[str] = Field(default_factory=list)

    # contact and private guidance
    has_customer_service_entry: bool | None = None
    has_private_contact_entry: bool | None = None
    has_group_chat_entry: bool | None = None

    # download and inducement
    has_download_button: bool | None = None
    has_install_inducement: bool | None = None
    has_reward_claim: bool | None = None
    has_task_reward_claim: bool | None = None

    # finance / crypto
    has_crypto_trading: bool | None = None
    has_fiat_exchange: bool | None = None
    has_investment_return_claim: bool | None = None

    # region and audience hints
    visible_country_or_region_terms: list[str] = Field(default_factory=list)

    # traceability
    signal_evidence: list[SignalEvidence] = Field(default_factory=list)
    extraction_warnings: list[str] = Field(default_factory=list)
    schema_version: str = "visual-signals-0.1"
```

字段说明：

- `None` 表示不确定或未提取，不等于 `False`。
- `False` 表示截图中明确未发现该信号。
- `evidence_texts` 用于保存可见文字证据，例如“邀请码”“联系客服”“充值”“提现”。
- `signal_evidence` 用于精确记录某个字段对应的证据文本和置信度。

## 6. 场景信号清单补充格式

业务团队后续应按以下格式补充不同 `page_type` 或审核场景需要关注的视觉信号。

### 6.1 模板

```md
### 场景：<page_type 或业务场景名>

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  | 待确认 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
|  |  | hard/weak/protective | block/need_preview/pass |  | 待确认 |
```

规则类型说明：

- `hard`：明确违规，可支撑 `block`。
- `weak`：可疑但证据不足，只能支撑 `need_preview`。
- `protective`：保护因子，只能降低弱风险，不覆盖明确违规。

## 7. 场景示例

以下示例用于说明规则表达方式，不代表最终业务口径。最终规则应由公司内部业务团队确认。

### 7.1 login_auth

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 是否有登录表单 | `has_login_form` | bool | 页面可见账号、密码、验证码等输入 | 账号、密码、验证码 | 建议保留 |
| 是否要求账号凭证 | `asks_for_credentials` | bool | 页面要求输入账号、密码、验证码等 | 请输入账号、请输入密码 | 建议保留 |
| 是否有邀请码字段 | `has_invite_code_field` | bool | 页面出现邀请码、推荐码、注册码等字段 | 邀请码、推荐码 | 建议保留 |
| 是否无邀请码无法注册 | `registration_requires_invite_code` | bool | 文案显示注册依赖邀请码或客服 | 无邀请码请联系客服 | 待业务确认 |
| 是否有客服入口 | `has_customer_service_entry` | bool | 页面可见在线客服、联系客服等入口 | 在线客服、联系客服 | 建议保留 |
| 是否有私域联系入口 | `has_private_contact_entry` | bool | 页面引导添加微信、Telegram、QQ群等 | 添加客服、进群 | 待业务确认 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `weak.login.invite_customer_service` | `has_login_form` + `has_invite_code_field` + `has_customer_service_entry` | weak | need_preview | 登录页同时出现邀请码和客服入口，疑似封闭式引流或灰产入口 | 待业务确认 |
| `weak.login.invite_required_private_contact` | `registration_requires_invite_code` + `has_private_contact_entry` | weak | need_preview | 无邀请码需私域联系，风险高于普通登录页 | 待业务确认 |

### 7.2 payment

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 是否有支付入口 | `has_payment_entry` | bool | 页面有付款、支付、收银台等入口 | 支付、立即付款 | 建议保留 |
| 是否有充值入口 | `has_recharge_entry` | bool | 页面有充值、入金、首充等入口 | 充值、首充 | 建议保留 |
| 是否有提现入口 | `has_withdraw_entry` | bool | 页面有提现、出金、提款等入口 | 提现、提款 | 建议保留 |
| 是否要求银行卡 | `asks_for_bank_card` | bool | 页面要求填写银行卡或支付账户 | 银行卡、卡号 | 建议保留 |
| 是否要求实名 | `asks_for_real_name` | bool | 页面要求填写姓名、身份证等 | 姓名、实名 | 建议保留 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `weak.payment.cash_sensitive_fields` | `has_payment_entry` + (`asks_for_bank_card` or `asks_for_real_name`) | weak | need_preview | 支付页收集敏感身份或银行卡信息，需要复核 | 待业务确认 |

### 7.3 download_install

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 是否有下载按钮 | `has_download_button` | bool | 页面出现下载、安装、立即体验等按钮 | 下载、安装 | 建议保留 |
| 是否诱导安装 | `has_install_inducement` | bool | 以奖励、福利、解锁等方式诱导安装 | 立即领取、安装后查看 | 待业务确认 |
| 是否有奖励承诺 | `has_reward_claim` | bool | 页面承诺红包、收益、返现等 | 红包、返现 | 待业务确认 |
| 是否有私域联系入口 | `has_private_contact_entry` | bool | 下载页引导联系个人或群组 | 联系客服、加入群 | 待业务确认 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `weak.download.reward_install` | `has_download_button` + (`has_install_inducement` or `has_reward_claim`) | weak | need_preview | 下载页带奖励或诱导安装信号 | 待业务确认 |

### 7.4 mall / brand_impersonation

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 可见品牌名 | `visible_brand_names` | list[str] | 页面可见品牌、平台、店铺名 | Apple、Nike、官方旗舰店 | 建议保留 |
| 是否有官方声明 | `has_official_claim` | bool | 页面宣称官方、认证、旗舰等 | 官方、认证、旗舰店 | 建议保留 |
| 是否有认证标识 | `has_certification_or_badge` | bool | 页面可见认证、证书、官方徽标 | 官方认证、证书 | 待业务确认 |
| 是否疑似仿冒品牌 | `impersonated_brand_suspected` | bool | 视觉上疑似冒用知名品牌或官方身份 | 官方、旗舰、授权 | 待业务确认 |
| 是否有支付入口 | `has_payment_entry` | bool | 页面有购买、下单、支付入口 | 购买、下单、支付 | 建议保留 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `weak.trade.official_claim_payment` | `has_official_claim` + `has_payment_entry` | weak | need_preview | 有官方声明且引导交易，需要结合品牌、URL、平台信息进一步判断 | 待业务确认 |

### 7.5 investment / crypto / rebate

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 页面主语言 | `primary_language` | str | zh/en/mixed/unknown | 中文、English | 建议保留 |
| 是否有语言切换 | `has_language_switch` | bool | 页面有语言切换入口 | CN/EN、Language | 建议保留 |
| 支持语言 | `supported_languages` | list[str] | 可见语言选项 | 中文、English | 建议保留 |
| 是否有虚拟币交易 | `has_crypto_trading` | bool | 页面可见 BTC、ETH、USDT、行情、交易等 | BTC、USDT、Trade | 待业务确认 |
| 是否有法币兑换 | `has_fiat_exchange` | bool | 页面可见 CNY、USD、充值兑换等 | CNY、USD、兑换 | 待业务确认 |
| 是否有充值入口 | `has_recharge_entry` | bool | 页面可见充值、入金、Deposit 等 | 充值、Deposit | 待业务确认 |
| 是否有提现入口 | `has_withdraw_entry` | bool | 页面可见提现、Withdraw 等 | 提现、Withdraw | 待业务确认 |
| 是否有收益承诺 | `has_investment_return_claim` | bool | 页面承诺稳赚、高收益、返利等 | 稳赚、高收益、返利 | 待业务确认 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `weak.crypto.zh_trading_payment` | `primary_language == zh` + `has_crypto_trading` + (`has_recharge_entry` or `has_withdraw_entry`) | weak | need_preview | 中文虚拟币交易及资金出入入口，需按公司策略复核 | 待业务确认 |
| `weak.crypto.en_no_cn_signal` | `primary_language == en` + `has_crypto_trading` + no Chinese audience signal | weak/protective | pass 或 need_preview | 英文站且无中文服务迹象时是否保护，待业务确认 | 待业务确认 |

### 7.6 pure_image

#### 需要提取的 VisualSignals

| 信号 | 字段名 | 类型 | 说明 | evidence_texts 示例 | 状态 |
| --- | --- | --- | --- | --- | --- |
| 是否包含明确违规文字 | `visible_keywords` | list[str] | 图片中出现明确违规词 | 裸聊、博彩、翻墙 | 待业务确认 |
| 是否含交易或客服入口 | `has_payment_entry` / `has_customer_service_entry` | bool | 纯图中仍可能出现引流或交易信号 | 联系客服、充值 | 待业务确认 |

#### 规则组合草案

| 规则 ID | 条件 | 规则类型 | 建议动作 | 说明 | 状态 |
| --- | --- | --- | --- | --- | --- |
| `protective.image.pure_low_risk` | `url_classification.is_pure_image == true` + 无明确违规视觉信号 | protective | pass | 纯图片默认低风险 | 已有原则 |
| `hard.image.explicit_violation` | 纯图片 + 明确色情/赌博/诈骗等证据 | hard | block | 保护因子不覆盖明确违规 | 已有原则 |

## 8. RuleEngine 消费方式

`RuleEngine` 后续应基于以下输入组合判断：

```text
VLMResult
URLClassification
PlatformContext
VisualSignals
warnings/errors
```

判断顺序建议：

1. 识别 hard rule。
2. 识别 weak signal。
3. 识别 protective factor。
4. 计算 risk_score、risk_level、confidence、evidence_level。
5. 输出 `RuleResult`。

示例：

```text
page_type = login_auth
+ has_invite_code_field = true
+ has_customer_service_entry = true
+ no explicit violation
=> weak signal
=> EvidenceGate 最多输出 need_preview
```

示例：

```text
url_classification.is_pure_image = true
+ vlm_result.advice = good
+ no explicit visual signal
=> protective factor
=> EvidenceGate 可输出 pass
```

示例：

```text
vlm_result.advice = gamble
+ description 有可见博彩证据
=> hard rule
=> EvidenceGate 输出 block
```

## 9. URLClassification 的职责边界

`URLClassification` 负责提取 URL 特征：

```text
domain
path
suffix
query_keys
is_pure_image
is_website_or_landing_page
resource_type_inferred
url_features
parse_error
```

`RuleEngine` 负责判断这些特征是否构成风险：

```text
短链 -> weak signal
IP 域名 -> weak signal
深层子域名 -> weak signal
异常 query -> weak signal
图片 URL + 无明确违规 -> protective factor
图片 URL + 明确违规 -> hard rule 仍生效
```

URL 不应单独制造 `block`，除非后续公司规则明确把某些 URL 特征列为 hard rule。

## 10. AgentDecision 的职责边界

Agent 不负责发明视觉信号，也不负责直接执行场景规则。

Agent 只允许读取：

```text
VLMResult
URLClassification
PlatformContext
VisualSignals
RuleResult
warnings/errors 摘要
```

Agent 不得：

- 编造 `VisualSignals`。
- 编造平台字段。
- 覆盖 `vlm_result`。
- 把 weak signal 描述成明确违规。
- 直接决定 `final_action`。

## 11. 延迟控制原则

MVP 不应为了 `VisualSignals` 默认增加二次模型调用。

优先策略：

```text
一次 VLM 输出中包含 VLMResult + VisualSignals
```

如果后续需要按场景补充信号，应优先通过 prompt 和 schema 优化，而不是无条件增加模型调用次数。

可选优化：

- 纯图片场景只提取基础视觉违规信号。
- 网页/落地页场景提取登录、支付、下载、语言、品牌、客服等交互信号。
- 不确定字段允许为空或 unknown，避免为了填满字段增加推理负担。

## 12. 测试建议

后续测试应覆盖以下类型：

| Case | 目标 |
| --- | --- |
| 纯图片 + VLM good | 验证保护因子 |
| 纯图片 + 明确违规文本或画面 | 验证明确违规不被保护覆盖 |
| 登录页 + 邀请码 + 客服入口 | 验证 login_auth 视觉信号 |
| 登录页 + 无邀请码无法注册 | 验证 `registration_requires_invite_code` |
| 下载页 + 奖励诱导 | 验证 download_install 信号 |
| 支付页 + 银行卡/实名字段 | 验证 payment 信号 |
| 商城页 + 官方声明 + 支付入口 | 验证 mall / brand_impersonation 信号 |
| 虚拟币中文站 + 充值/提现 | 验证 crypto 视觉信号 |
| 虚拟币英文站 + 无中文服务 | 验证语言和受众信号 |
| URL 异常 + 无视觉违规 | 验证 URL weak signal 不直接 block |

## 13. 待业务补充区域

公司内部业务团队应补充：

1. 每个 `page_type` 必须提取的视觉信号清单。
2. 每个视觉信号的业务含义。
3. 信号组合到 `hard`、`weak`、`protective` 的规则表。
4. 虚拟币、仿冒商城、支付诈骗、下载诱导等具体规则口径。
5. 哪些规则可以支撑 `block`，哪些只能支撑 `need_preview`。
6. 具体 `rule_id` 命名和 `rule_version` 升级策略。

补充时请优先使用第 6 节模板，避免把业务规则写成自由文本。
