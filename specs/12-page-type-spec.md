# Page Type Spec

本文档定义 `page_type` 的完整枚举和命名口径。`page_type` 只描述页面业务场景，不描述诈骗手法、风险行为或最终处置标签。

## 1. 命名原则

- `page_type` 回答“这是什么页面”。
- 能用一个词表达清楚时，不强制使用下划线。
- 不使用 `fake`、`scam`、`fraud` 表示页面类型；仿冒、诈骗等判断应进入 `risk_behavior`、`visual_signals` 或 `advice`。
- 不使用 `task` 作为泛化后缀；只有页面本身是接任务、做任务、返佣任务时，才允许使用任务语义。
- 金融与交易欺诈页面需要按可见业务场景拆细，避免把商城、支付、刷单、投资、虚拟币混在一个页面类型里。

## 2. 完整枚举

| page_type | 页面场景 |
| --- | --- |
| `mall` | 商城、购物、商品、订单、交易展示页 |
| `login_auth` | 登录、注册、认证、授权、验证页 |
| `payment` | 支付、充值、提现、余额、红包、收益页 |
| `download_install` | 下载、安装、打开 App 引导页 |
| `social_chat` | 客服、私信、加群、加好友、聊天页 |
| `rebate` | 刷单、任务返佣、点赞赚钱、垫付返利页 |
| `investment` | 投资理财、股票、基金、黄金、能源投资、证券开户页 |
| `crypto` | 虚拟币、数字货币、钱包、交易所、挖矿、空投页 |
| `gambling_lottery` | 博彩、棋牌、彩票、转盘、抽奖页 |
| `porn_dating` | 色情、裸聊、约炮、成人交友页 |
| `brand_impersonation` | 官方、品牌、银行、平台、客服等身份包装页 |
| `game_plugin` | 游戏私服、外挂、辅助页 |
| `vpn_proxy` | VPN、代理、翻墙工具页 |
| `politic_sensitive` | 涉政、涉军、暴恐、分裂内容页 |
| `other_unknown` | 无法明确归类的其他页面 |

## 3. 金融与交易欺诈子类

| 金融诈骗子类 | page_type | 判断口径 |
| --- | --- | --- |
| mall 商城诈骗 | `mall` | 页面主体是商品、店铺、订单、购物交易或商城活动 |
| pay 支付诈骗 | `payment` | 页面主体是付款、收银、充值、提现、余额、红包或收益操作 |
| 刷单诈骗 | `rebate` | 页面主体是刷单、任务返佣、点赞赚钱、垫付返利或佣金任务 |
| 投资诈骗 | `investment` | 页面主体是投资理财、证券开户、股票、基金、黄金、能源投资或金融产品 |
| 虚拟币 | `crypto` | 页面主体是虚拟币、数字货币、钱包、交易所、挖矿、空投或币种行情 |

## 4. 旧值替换关系

| 旧 page_type | 新 page_type |
| --- | --- |
| `normal_trade` | `mall` |
| `payment_cash` | `payment` |
| `investment_task` | 按页面主体拆为 `rebate` / `investment` / `crypto` |
| `fake_official` | `brand_impersonation` |

## 5. 与其他字段的边界

`page_type` 不承载违规结论。例如，假交易所投资页的 `page_type` 应为 `investment` 或 `crypto`，仿冒交易所的可见事实进入 `visual_signals`，风险抽象进入 `risk_behavior`，最终处置进入 `advice`。

示例：

```json
{
  "page_type": "investment",
  "risk_behavior": ["identity_impersonation", "payment_inducement"],
  "advice": "finance"
}
```
