# 05 Rule Engine Spec

## 1. 规则层职责

Rule Engine 是 MVP 的确定性审核层，负责：

- 识别硬规则。
- 识别弱信号。
- 识别保护因子。
- 计算内部风险分。
- 输出风险等级、置信度、证据等级。
- 记录规则版本和命中详情。

规则输入必须包含：

```text
VLMResult.page_type
VLMResult.risk_behavior
VLMResult.visual_signals
VLMResult.advice
VLMResult.description
URLClassification
PlatformContext
```

核心组合原则：

```text
page_type + risk_behavior + visual_signals => final_advice / rule hits
```

## 2. 规则类型

| 类型 | 含义 | 是否可支撑 block |
| --- | --- | --- |
| hard | 明确违规，可被人工直接理解 | 是 |
| weak | 可疑但证据不足 | 否 |
| protective | 降低弱风险处置强度 | 否 |

## 3. 硬规则

MVP 不穷尽所有规则，但结构必须支持扩展。

典型 hard rule 来源：

- VLM advice 为 `child_porn`、`porn`、`gamble`、`finance`、`fake`、`politic`、`game`、`vpn`，且 `description` 有可见证据。
- `risk_behavior` 对象包含明确命中的高风险行为，例如 `gambling_or_betting`、`pornographic_content`、`virtual_currency_trading`、`vpn_proxy_service_provision`。
- `visual_signals` 对象包含明确可见证据，例如 `explicit_nudity_visible`、`gambling_slang_visible`、`vpn_proxy_service_provider_visible`。
- URL 或页面文本命中已配置强硬规则。

输出：

```json
{
  "rule_id": "hard.vlm.gamble",
  "rule_type": "hard",
  "evidence": "VLM 判断为 gamble，页面存在博彩诱导",
  "source": "vlm",
  "severity": "high"
}
```

## 4. 弱信号

弱信号只支持 `need_preview`：

- 登录页出现邀请码、验证码、客服入口。
- URL 出现短链、跳转、推广码、异常 query。
- 页面类型和行为组合可疑但无 decisive evidence。

弱信号不得直接升级为 `block`。

## 5. 保护因子

MVP 保护因子：

```text
pure_image
v4_v5_china_customer
```

保护因子应用条件：

```python
if not has_explicit_violation:
    apply_protective_factors()
```

明确违规时，保护因子不能输出 `pass`。

## 6. 风险分

建议分值：

| 因子 | 分值 |
| --- | --- |
| hard rule | +80 到 +100 |
| VLM 明确违规 advice | +70 |
| weak signal | +15 到 +30 |
| URL 异常 | +10 到 +25 |
| 平台历史风险 | +10 到 +30 |
| pure image | -20 |
| v4/v5 中国站客户 | -30 |

风险分只用于排序和风险等级，不得单独制造 `block`。

## 7. 证据等级映射

| 条件 | evidence_level |
| --- | --- |
| hard rule 或 decisive evidence | `decisive` |
| weak signal | `weak` |
| 无风险或只有保护因子 | `none` |
| 关键节点失败 | `unknown` 或 `weak` |

## 8. 规则版本

默认：

```text
rules-0.1
```

以下变更必须升级 `rule_version`：

- 新增或删除规则。
- 修改 hard/weak/protective 分类。
- 修改规则权重。
- 修改证据等级映射。
- 修改保护因子逻辑。

## 9. 配置化策略

推荐目录：

```text
config/rules/
  hard_rules.yaml
  weak_signals.yaml
  protective_factors.yaml
```

规则配置示例：

```yaml
- rule_id: hard.vlm.gamble
  type: hard
  enabled: true
  source: vlm
  match:
    advice_in: ["gamble"]
  evidence_template: "VLM 判断为赌博风险：{description}"
  severity: high
```

## 10. change.md 场景规则方向

以下规则方向来自 `change.md`，用于后续配置化落地。

| page_type | 条件摘要 | 建议 final_advice | 规则类型 |
| --- | --- | --- | --- |
| `investment` | `fund_operation = true` | `finance` | hard 或 weak，按证据强度配置 |
| `investment` | `trading = true` + `institutional_branding = true` | `finance` | hard 或 weak，按证据强度配置 |
| `investment` | `login&register = true` + `customer_service_entry = true` + `institutional_branding = true` | `finance` | weak |
| `investment` | `invitation_code_required = true` + (`login&register` 或 `open_account`) | `finance` | weak |
| `mall` | `mall_transaction = true` + `mall_brand_visible = true` + `transaction_entry_visible = true` | `finance` | weak，通常 `need_preview` |
| `rebate` | `task_commission_rebate = true` + `task_or_order_rebate_visible = true` | `finance` | hard 或 weak，按证据强度配置 |
| `rebate` | `advance_payment_or_deposit = true` + `advance_payment_or_recharge_visible = true` | `finance` | hard |
| `rebate` | `commission_or_cashout_visible = true` + 任务或先付费信号 | `finance` | hard 或 weak |
| `login` | `restricted_access_registration = true` + `invitation_code_required = true` + `public_registration_entry = false` | `finance` 或 `other_fraud` | weak |
| `login` | `customer_service_assisted_login = true` + `customer_service_entry = true` | `finance` 或 `other_fraud` | weak |
| `crypto` | `virtual_currency_trading = true` + 中国站或中文/语言切换信号 | `finance` | hard 或 weak |
| `digital_goods` | `virtual_goods_trading = true` + `virtual_goods_or_account_visible = true` | `other_fraud` | hard 或 weak |
| `digital_goods` | `vpn_or_proxy_service_visible = true` | `vpn` | hard 或 weak |
| `vpn` | VPN 使用或服务提供行为 + 对应可见信号 | `vpn` | hard |
| `porn` | `pornographic_content = true` + 明确裸露或色情服务文本 | `porn` | hard |
| `porn` | 艺术/医学/情趣用品/擦边但无明确裸露或色情服务文本 | 非 `porn` | protective 或 no hit |
| `gambling` | `gambling_or_betting = true` | `gamble` | hard |
| `gambling` | 棋牌赌博行为 + 棋牌/捕鱼/电子游艺可见 | `gamble` | hard |
| `gambling` | 博彩黑话、博彩实体、赌场、体育博彩、彩票抽奖任一可见 | `gamble` | hard |

`payment` 当前字段待补充，Rule Engine 只保留 page type 分支，不新增未定义字段。
