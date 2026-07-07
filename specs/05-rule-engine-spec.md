# 05 Rule Engine Spec

## 1. 规则层职责

Rule Engine 是 MVP 的确定性审核层，负责：

- 识别硬规则。
- 识别弱信号。
- 识别保护因子。
- 计算内部风险分。
- 输出风险等级、置信度、证据等级。
- 记录规则版本和命中详情。

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
- `risk_behavior` 包含明确诱导行为，例如 `gambling_inducement`、`porn_inducement`。
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
- 下载页诱导安装但无明确违规。
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

