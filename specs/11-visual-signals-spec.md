# 11 Visual Signals Spec

## 1. 目的

本文档定义 `visual_signals` 的设计口径和字段清单。字段来源以 `change.md` 已补充内容为准。

`visual_signals` 的目标不是让 VLM 直接做最终审核判断，而是把截图中可见的视觉事实结构化，供 Rule Engine 组合判断。

核心原则：

```text
VLM 负责：看到什么
RuleEngine 负责：这些信号按规则意味着什么
AgentDecision 负责：基于已有证据给出候选解释
EvidenceGate 负责：最终 final_action
```

## 2. 输出位置

`visual_signals` 是 `VLMResult` 的一部分：

```python
class VLMResult(BaseModel):
    advice: VLMAdvice
    page_type: PageType
    risk_behavior: dict[str, bool]
    visual_signals: dict[str, bool | list[str]]
    description: str
```

MVP 不新增独立 Visual Signal Extractor 节点。VLM Adapter 可以一次性返回 `page_type`、`risk_behavior`、`visual_signals`、`advice`、`description`。

## 3. 字段原则

- 只记录截图中可见或可读文本中的事实。
- 不直接输出 `final_action`。
- 不编造截图中不存在的信息。
- 不从平台字段、历史 case、外部网页内容中补充信息。
- 布尔字段命中为 `true`，未命中为 `false`。
- 数组字段无法提取时输出空数组 `[]`。
- 只使用当前 `page_type` 在 `change.md` 中已定义的字段。
- `payment` 当前字段待补充，可输出空对象。

## 4. page_type 字段清单

### 4.1 mall

```json
{
  "risk_behavior": {
    "mall_transaction": true
  },
  "visual_signals": {
    "mall_brand_visible": true,
    "visible_brand_names": [],
    "product_info_visible": true,
    "transaction_entry_visible": true
  }
}
```

### 4.2 payment

`payment` 当前仅保留 page type，字段待补充：

```json
{
  "risk_behavior": {},
  "visual_signals": {}
}
```

### 4.3 rebate

```json
{
  "risk_behavior": {
    "task_commission_rebate": true,
    "advance_payment_or_deposit": true
  },
  "visual_signals": {
    "task_or_order_rebate_visible": true,
    "commission_or_cashout_visible": true,
    "advance_payment_or_recharge_visible": true,
    "agent_recruitment_visible": true,
    "supply_or_substation_entry": true
  }
}
```

### 4.4 login

```json
{
  "risk_behavior": {
    "restricted_access_registration": true,
    "customer_service_assisted_login": true
  },
  "visual_signals": {
    "customer_service_entry": true,
    "invitation_code_required": true,
    "guest_access_entry": true,
    "public_registration_entry": false
  }
}
```

### 4.5 investment

```json
{
  "risk_behavior": {
    "trading": true,
    "fund_operation": true,
    "login&register": true,
    "open_account": false
  },
  "visual_signals": {
    "institutional_branding": true,
    "customer_service_entry": true,
    "invitation_code_required": false
  }
}
```

### 4.6 crypto

```json
{
  "risk_behavior": {
    "virtual_currency_trading": true
  },
  "visual_signals": {
    "crypto_asset_visible": true,
    "primary_language_chinese": true,
    "language_switch_entry": true
  }
}
```

### 4.7 digital_goods

```json
{
  "risk_behavior": {
    "virtual_goods_trading": true,
    "agent_recruitment": true
  },
  "visual_signals": {
    "virtual_goods_or_account_visible": true,
    "agent_recruitment_visible": true,
    "customer_service_entry": true,
    "supply_or_substation_entry": true,
    "order_query_entry": true,
    "vpn_or_proxy_service_visible": true
  }
}
```

### 4.8 vpn

```json
{
  "risk_behavior": {
    "vpn_proxy_usage": true,
    "vpn_proxy_service_provision": true
  },
  "visual_signals": {
    "vpn_proxy_usage_state_visible": true,
    "vpn_proxy_service_provider_visible": true
  }
}
```

### 4.9 porn

```json
{
  "risk_behavior": {
    "pornographic_content": true
  },
  "visual_signals": {
    "explicit_nudity_visible": true,
    "sexual_service_text_visible": true,
    "sexualized_but_non_explicit_visible": false,
    "sex_toy_visible": false,
    "artistic_nudity_visible": false,
    "medical_or_educational_sexual_content_visible": false
  }
}
```

### 4.10 gambling

```json
{
  "risk_behavior": {
    "gambling_or_betting": true,
    "chess_card_gambling": true
  },
  "visual_signals": {
    "gambling_entity_visible": true,
    "gambling_slang_visible": true,
    "chess_card_game_visible": true,
    "casino_or_live_dealer_visible": true,
    "sports_betting_visible": true,
    "lottery_or_draw_visible": true
  }
}
```

## 5. RuleEngine 消费方式

Rule Engine 基于以下输入组合判断：

```text
VLMResult.page_type
VLMResult.risk_behavior
VLMResult.visual_signals
VLMResult.advice
VLMResult.description
URLClassification
PlatformContext
warnings/errors
```

判断顺序建议：

1. 识别 hard rule。
2. 识别 weak signal。
3. 识别 protective factor。
4. 计算 risk_score、risk_level、confidence、evidence_level。
5. 输出 `RuleResult`。

## 6. 测试建议

测试至少覆盖：

- `risk_behavior` 为对象，不接受旧数组结构。
- `visual_signals` 为对象，不接受数组或字符串。
- `payment` 字段待补充时允许空对象。
- `login` 邀请码 + 客服入口弱信号。
- `rebate` 任务返佣 + 垫付充值信号。
- `crypto` 虚拟币交易 + 中文或语言切换信号。
- `digital_goods` 数字商品 + VPN 商品信号。
- `porn` 色情硬信号和艺术/医学/情趣用品保护场景。
- `gambling` 博彩黑话、棋牌、彩票抽奖硬信号。
