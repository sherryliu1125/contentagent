# 内容审核 Agent MVP 设计 Delta

本文档记录相对 `ARCHITECTURE.md` 的新增、明确和工程化约束。它用于研发评审时快速确认：从架构方案到可实现 spec，哪些地方被进一步收敛。

## 1. 文档结构 Delta

| 项 | `ARCHITECTURE.md` | 当前 SDD / Spec |
| --- | --- | --- |
| 文档定位 | 架构方案 | `SDD.md` 作为轻量总览，细节拆分到 `specs/*.md` |
| 工程入口 | 未拆 spec | 新增 10 份工程 spec |
| 变更说明 | 无单独 delta | 新增 `DELTA.md` |

## 2. 决策权 Delta

| 设计点 | 新增或明确 |
| --- | --- |
| Agent 决策权 | Agent 输出只是候选，不是最终裁决 |
| Evidence Gate | 明确 Evidence Gate 是 `final_action` 的最终来源 |
| `block` 门槛 | `block` 必须有硬规则命中或 decisive evidence |
| weak signal | weak signal 只能支撑 `need_preview`，不能包装成明确违规 |
| 保护因子 | 纯图片和 v4/v5 中国站客户只保护弱风险，不覆盖明确违规 |

## 3. 状态与字段 Delta

| 设计点 | 新增或明确 |
| --- | --- |
| VLM 原始结果 | 必须独立保存为 `vlm_result` |
| VLM 必留字段 | `advice`、`page_type`、`risk_behavior`、`visual_signals`、`description` |
| `page_type` | 收敛为 `change.md` 中已定义的 10 个值 |
| `risk_behavior` | 从旧数组结构收敛为对象结构 |
| `visual_signals` | 新增为 VLM 原始结果中的一等字段 |
| Agent 结果 | 保存为 `agent_decision`，字段名使用 candidate 语义 |
| 最终结果 | 保存为 `evidence_result` 和 `ReviewOutput` |
| 字段写入责任 | 明确每个节点只能写入自己负责的字段 |
| 平台字段缺失 | 显式进入 `missing_fields` 和 `warnings` |

## 3.1 Page Type Field Delta

本轮相对旧 spec 的核心变化：

- `risk_behavior` 不再是 `list[str]`，必须是 `dict[str, bool]`。
- `visual_signals` 必须作为对象输出，字段只记录截图中可见事实。
- Rule Engine 使用 `page_type + risk_behavior + visual_signals` 作为稳定规则输入。
- 旧 page type `login_auth`、`gambling_lottery`、`porn_dating`、`vpn_proxy` 等不再作为正式枚举。
- `payment` 当前只保留 page type 名称，字段待后续补充，不新增未定义字段。
- `change.md` 未覆盖的标签和页面字段暂不在本轮补充。

## 4. 异常处理 Delta

| 异常 | 新增或明确 |
| --- | --- |
| 输入校验失败 | 结构化失败，`final_action=None` |
| VLM 失败 | 不默认 `pass`，降级为 `need_preview` |
| VLM schema 非法 | 不中断流程，记录 warning，最终至少 `need_preview` |
| VLM page_type 非法 | 不中断流程，记录 warning，最终至少 `need_preview` |
| VLM page 字段结构非法 | `risk_behavior` / `visual_signals` 非对象时记录 warning，最终至少 `need_preview` |
| URL 解析失败 | 记录 parse error，通常进入 `need_preview` |
| Rule Engine 异常 | fallback 为弱信号复核路径 |
| Agent 输出非法 | 丢弃 Agent 输出，使用规则层 fallback |
| Audit 写入失败 | 不改变审核结果，只进入 warning |

## 5. 版本与审计 Delta

| 版本字段 | 新增或明确 |
| --- | --- |
| `schema_version` | Pydantic 字段和枚举变化时升级 |
| `graph_version` | LangGraph 节点和条件边变化时升级 |
| `rule_version` | 规则、权重、证据门槛变化时升级 |
| `prompt_version` | Agent prompt 或输出约束变化时升级 |
| `model_version` | VLM 或 Agent 模型变化时记录 |
| `adapter_version` | VLM / 平台 adapter 映射变化时升级 |
| `mock_data_version` | mock 数据变化时升级 |

## 6. 测试 Delta

新增明确测试范围：

- Pydantic schema 测试。
- 节点级输入输出测试。
- LangGraph workflow 集成测试。
- Evidence Gate 冲突修正测试。
- Agent 输出非法 fallback 测试。
- VLM 失败降级测试。
- v4/v5 中国站客户保护测试。
- 纯图片保护测试。
- 明确违规不被保护因子覆盖测试。
- 审计版本字段完整性测试。

## 7. 最重要的工程结论

最终实现必须遵守：

```text
VLM 原始判断 != Agent 候选判断 != Evidence Gate 最终动作
```

对应字段：

```text
vlm_result.advice
agent_decision.final_action_candidate
evidence_result.final_action
output.final_action
```
