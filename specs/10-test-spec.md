# 10 Test Spec

## 1. 测试层级

| 类型 | 覆盖 |
| --- | --- |
| 单元测试 | Pydantic model、URL parser、规则函数、Evidence Gate |
| 节点测试 | 每个 LangGraph 节点输入输出 |
| Workflow 集成测试 | 正常路径和异常路径 |
| 回归 case 测试 | 典型风险、弱信号、保护因子 |
| Adapter 测试 | VLM mock、平台 mock schema |
| Agent schema 测试 | 非法输出、fallback |
| Audit 测试 | 版本字段、写入失败 |

## 2. 必测断言

所有 workflow 测试必须断言：

- `final_action` 只可能是 `block`、`need_preview`、`pass`。
- 结构化失败时 `final_action=None`。
- `vlm_result.advice` 原样保留。
- `vlm_result.page_type` 只使用 `change.md` 允许枚举。
- `vlm_result.risk_behavior` 是对象，不是数组。
- `vlm_result.visual_signals` 是对象。
- Agent 不覆盖 VLM 原始字段。
- Evidence Gate 是最终动作来源。
- 只有 weak signal 时不得输出 `block`。
- 明确违规时保护因子不得输出 `pass`。
- VLM 失败不得默认 `pass`。
- 审计信息包含版本字段。

## 3. 典型测试 case

| 编号 | 场景 | 输入特征 | 预期 |
| --- | --- | --- | --- |
| TC01 | 纯图片低风险 | 图片后缀，VLM good，无弱信号 | `pass` |
| TC02 | 纯图片明确违规 | 图片后缀，VLM porn/gamble | `block` |
| TC03 | v4/v5 中国站客户弱风险 | v4 + CN，登录页弱信号，无硬规则 | `pass` |
| TC04 | v4/v5 中国站客户明确违规 | v5 + CN，VLM gamble | `block` |
| TC05 | 登录页弱信号 | login，邀请码，客服入口 | `need_preview` |
| TC06 | 商城交易弱信号 | mall，商品交易入口，品牌可见 | `need_preview` |
| TC07 | 明确赌博 | VLM gamble，博彩诱导 | `block` |
| TC08 | 明确色情 | VLM porn，成人诱导 | `block` |
| TC09 | VLM 失败 | VLM adapter 抛异常 | `need_preview` |
| TC10 | Agent 输出非法 | Agent 输出 `auto_ban` | fallback 后按 Evidence Gate |
| TC11 | URL 解析失败 | 非标准 URL | `need_preview` |
| TC12 | 平台字段缺失 | customer_level/site_region 空 | 不失败，confidence 降低 |
| TC13 | 证据不足 block | Agent 候选 block，只有 weak signals | 降为 `need_preview` |
| TC14 | 明确违规被 Agent pass | hard_rule_hits 非空，Agent pass | 改为 `block` |
| TC15 | 输入缺 URL | url 为空 | `structured_failed` |
| TC16 | VLM 旧 risk_behavior 数组 | `risk_behavior=[]` | `need_preview` 并 warning |
| TC17 | 虚拟币中文站 | crypto，virtual_currency_trading，中文或语言切换 | `block` 或 `need_preview`，按规则强度 |
| TC18 | 刷单垫付 | rebate，advance_payment_or_deposit，充值/垫付可见 | `block` 或 `need_preview`，按规则强度 |
| TC19 | 数字商品售卖 VPN | digital_goods，vpn_or_proxy_service_visible | final_advice=`vpn`，动作按证据门槛 |
| TC20 | payment 字段待补充 | payment，risk_behavior/visual_signals 空对象 | 不因字段缺失失败 |

## 4. 回归测试要求

每次以下版本变化必须跑全量典型 case：

- `rule_version`
- `graph_version`
- `schema_version`
- `prompt_version`

Adapter 或 mock 数据变化时，至少运行相关 adapter 测试和 workflow 冒烟测试。

## 5. 最小验收

MVP 最小验收：

- 15 个典型 case 全部通过。
- 审计日志每条 case 可回放。
- Evidence Gate 冲突修正测试通过。
- VLM 失败、Agent 非法、平台缺失三类降级路径通过。
