# 12 Page Type Spec

本文档定义 `page_type` 的完整枚举和命名口径。`page_type` 只描述页面业务场景，不描述诈骗手法、风险行为或最终处置标签。

## 1. 命名原则

- `page_type` 回答“这是什么页面”。
- `risk_behavior` 回答“页面正在引导什么行为”。
- `visual_signals` 回答“截图中直接看见了什么”。
- `advice` 回答“截图内风险初判是什么标签”。
- 不使用 `fake`、`scam`、`fraud` 表示页面类型；仿冒、诈骗等判断应进入 `risk_behavior`、`visual_signals` 或 `advice`。
- 不把平台上下文写入 `page_type`；例如是否中国站点必须来自 `PlatformContext`。

## 2. 完整枚举

当前允许的 `page_type` 值以 `change.md` 为准：

| page_type | 页面场景 | 字段状态 |
| --- | --- | --- |
| `mall` | 商城、购物、店铺、商品、订单或商品交易页 | 已定义 |
| `payment` | 支付、充值、提现、余额、红包、收益页 | 待补充 |
| `rebate` | 刷单、任务返佣、点赞关注、兼职任务、提现返利页 | 已定义 |
| `login` | 登录、注册、认证、授权、加入、验证页 | 已定义 |
| `investment` | 投资理财、股票、基金、黄金、能源投资、证券开户页 | 已定义 |
| `crypto` | 虚拟货币、数字资产、交易所、钱包、币种行情页 | 已定义 |
| `digital_goods` | 虚拟商品、账号、卡密、兑换码、授权码、会员、数字服务、供货或分销平台页 | 已定义 |
| `vpn` | VPN、代理、翻墙工具、节点、机场、代理服务页 | 已定义 |
| `porn` | 色情、裸聊、约炮、成人直播、成人视频或成人服务页 | 已定义 |
| `gambling` | 博彩、棋牌、捕鱼、电子游艺、真人视讯、体育博彩、彩票、抽奖、赌场、投注页 | 已定义 |

`payment` 当前仅保留 page type 名称，`risk_behavior` 和 `visual_signals` 字段待后续补充。

## 3. 旧值替换关系

| 旧 page_type | 新 page_type |
| --- | --- |
| `login_auth` | `login` |
| `gambling_lottery` | `gambling` |
| `porn_dating` | `porn` |
| `vpn_proxy` | `vpn` |
| `normal_trade` | `mall` |
| `payment_cash` | `payment` |
| `investment_task` | 按页面主体拆为 `rebate` / `investment` / `crypto` |
| `fake_official` | 保留原页面主体，仿冒信息进入 `visual_signals` 或规则层 |
| `brand_impersonation` | 保留原页面主体，仿冒信息进入 `visual_signals` 或规则层 |

## 4. 与其他字段的边界

`page_type` 不承载违规结论。例如，假交易所投资页的 `page_type` 应为 `investment` 或 `crypto`，仿冒交易所的可见事实进入 `visual_signals`，风险抽象进入 `risk_behavior`，最终风险标签进入 `advice`。

示例：

```json
{
  "page_type": "investment",
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
  },
  "advice": "finance"
}
```
