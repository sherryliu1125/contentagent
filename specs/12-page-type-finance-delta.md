# Page Type 金融子类变更节选

本文档只承载本次金融相关 `page_type` 调整的节选。完整枚举和字段边界见 `specs/12-page-type-spec.md`。

## 1. 本次变更

| 金融诈骗子类 | page_type | 解释 |
| --- | --- | --- |
| mall 商城诈骗 | `mall` | 商城、购物、商品、订单、交易展示页 |
| pay 支付诈骗 | `payment` | 支付、充值、提现、余额、红包、收益页 |
| 刷单诈骗 | `rebate` | 刷单、任务返佣、点赞赚钱、垫付返利页 |
| 投资诈骗 | `investment` | 投资理财、股票、基金、黄金、能源投资、证券开户页 |
| 虚拟币 | `crypto` | 虚拟币、数字货币、钱包、交易所、挖矿、空投页 |

## 2. 命名理由

- `investment_task` 改为 `investment`：`task` 容易误导为必须存在明确任务流程；投资页面可以是行情、开户、产品介绍、登录或交易入口。
- `payment_cash` 改为 `payment`：一个词已经能表达支付、充值、提现等资金操作页面。
- `normal_trade` 改为 `mall`：金融诈骗子类里的商城诈骗需要更直观地表示商城/购物页面。
- 刷单使用 `rebate`：当前口径先强调返佣、返利、垫付收益，不把它泛化成所有任务页面。
- 虚拟币使用 `crypto`：覆盖 Bitcoin/BTC、USDT、ETH、钱包、交易所、挖矿和空投等场景。

## 3. 字段边界

`page_type` 只表示页面业务场景。是否诈骗、是否仿冒、是否诱导支付，不写进 `page_type`，由 `visual_signals`、`risk_behavior` 和 `advice` 表达。
