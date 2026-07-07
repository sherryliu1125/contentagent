# 04 Node Spec

## 1. Input Validation Node

| 项 | 内容 |
| --- | --- |
| 目标 | 校验输入，初始化 `ReviewState` |
| 输入 | raw request |
| 输出 | `case_id`、`input`、`warnings`、`errors` |
| 核心逻辑 | `image` 和 `url` 必填；`case_id` 为空则生成 |
| 异常 | 缺少必填字段则结构化失败 |
| 测试 | URL 缺失、image 缺失、case_id 自动生成 |

## 2. Mock VLM Adapter Node

| 项 | 内容 |
| --- | --- |
| 目标 | 返回 mock VLM 视觉初判 |
| 输入 | `input.image`、`case_id` |
| 输出 | `vlm_result` |
| 核心逻辑 | 按 case_id 查 mock，未命中返回低风险 mock |
| 异常 | mock schema 非法则 `vlm_result=None` 并 warning |
| 测试 | VLM 四字段保留、非法 advice、risk_behavior 长度 |

默认 mock：

```json
{
  "advice": "good",
  "page_type": "other_unknown",
  "risk_behavior": [],
  "description": "未见明显异常"
}
```

## 3. URL / Resource Classifier Node

| 项 | 内容 |
| --- | --- |
| 目标 | 解析 URL 和资源类型 |
| 输入 | `input.url`、`input.resource_type`、`vlm_result.page_type` |
| 输出 | `url_classification` |
| 核心逻辑 | 提取 domain/path/suffix/query，判断纯图片和落地页 |
| 异常 | URL parse 失败记录 warning |
| 测试 | 图片后缀、登录页、下载页、短链、非法 URL |

## 4. Mock Platform Context Adapter Node

| 项 | 内容 |
| --- | --- |
| 目标 | 构造标准化平台上下文 |
| 输入 | `customer_level`、`site_region`、`resource_type` |
| 输出 | `platform_context` |
| 核心逻辑 | 判断 `is_v4_v5_china_customer`，记录缺失字段 |
| 异常 | mock 未命中不失败 |
| 测试 | v4/v5 + 中国站、字段缺失、raw payload 保留 |

## 5. Rule Engine Node

| 项 | 内容 |
| --- | --- |
| 目标 | 输出规则命中和风险等级 |
| 输入 | `vlm_result`、`url_classification`、`platform_context` |
| 输出 | `rule_result` |
| 核心逻辑 | 硬规则、弱信号、保护因子、风险分 |
| 异常 | 异常时 fallback 到 `need_preview` 弱信号 |
| 测试 | 明确违规、弱信号、保护因子不覆盖明确违规 |

## 6. Agent Decision Node

| 项 | 内容 |
| --- | --- |
| 目标 | 输出候选动作和解释 |
| 输入 | VLM、URL、平台、规则结果 |
| 输出 | `agent_decision` |
| 核心逻辑 | 按 schema 输出，不编造字段，不覆盖 VLM |
| 异常 | 输出非法则 fallback |
| 测试 | 非法 final_action、空解释、编造平台字段、覆盖 VLM |

## 7. Evidence Builder Node

| 项 | 内容 |
| --- | --- |
| 目标 | 执行最终证据门槛 |
| 输入 | `rule_result`、`agent_decision`、`warnings`、`errors` |
| 输出 | `evidence_result` |
| 核心逻辑 | block 门槛、weak signal 限制、保护因子限制 |
| 异常 | Agent 缺失时使用规则 fallback |
| 测试 | block 降级、pass 升级、VLM 失败 need_preview |

## 8. Audit Logger Node

| 项 | 内容 |
| --- | --- |
| 目标 | 写入审计日志 |
| 输入 | 全量 `ReviewState` |
| 输出 | `audit_info` |
| 核心逻辑 | 记录输入、节点输出、版本、异常、最终动作 |
| 异常 | 写入失败只 warning |
| 测试 | 审计失败不改变 final_action、版本完整 |

## 9. Output Builder Node

| 项 | 内容 |
| --- | --- |
| 目标 | 构造对外输出 |
| 输入 | `evidence_result`、`vlm_result`、`audit_info` |
| 输出 | `ReviewOutput` |
| 核心逻辑 | 正常输出 Evidence Gate 结果，结构化失败输出错误 |
| 异常 | 输出构造失败返回最小结构化错误 |
| 测试 | final_action 来源、结构化失败、warnings/errors 透传 |

