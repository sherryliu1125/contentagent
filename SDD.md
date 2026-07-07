# 内容审核 Agent MVP SDD 总览

## 1. 文档定位

本文档是“内容审核 Agent MVP”的轻量系统设计总览，用于说明系统边界、工作流、spec 拆分和研发入口。

本项目采用：

```text
workflow-first, agent-enhanced 的受控审核 Agent
```

核心链路：

```text
Input
  -> VLM Adapter
  -> URL / Resource Classifier
  -> Platform Context Adapter
  -> Rule Engine
  -> Agent Decision Node
  -> Evidence Builder
  -> Audit Logger
  -> ReviewOutput
```

本文档不承载所有字段、规则和异常细节。可实现细节拆分在 `specs/*.md` 中，架构文档到工程 spec 的变化记录在 `DELTA.md` 中。

## 2. 与其他文档的关系

| 文档 | 作用 |
| --- | --- |
| `ARCHITECTURE.md` | 架构方案和 MVP 边界来源 |
| `SDD.md` | 系统总览、spec 索引、研发入口 |
| `DELTA.md` | 相对 `ARCHITECTURE.md` 的新增、明确和约束变化 |
| `specs/*.md` | 可直接拆任务实现的工程 spec |

## 3. MVP 范围

MVP 做：

- Python 主语言。
- LangGraph 工作流。
- Pydantic 核心数据结构。
- Mock VLM Adapter。
- Mock Platform Context Adapter。
- 规则层、Agent 候选决策、Evidence Gate、审计日志。
- 输出 `block`、`need_preview`、`pass` 三类 final_action。

MVP 不做：

- 不接真实 VLM。
- 不接真实平台接口。
- 不接自动封禁、下线、处罚接口。
- 不建设规则后台。
- 不做长期记忆。
- 不做自动学习。
- 不使用开放式多轮 ReAct Agent。

## 4. 核心设计约束

| 约束 | 说明 |
| --- | --- |
| final_action 三值 | 只能是 `block`、`need_preview`、`pass` |
| VLM 原始字段保留 | 必须保留 `advice`、`page_type`、`risk_behavior`、`description` |
| VLM 与 Agent 分离 | `vlm_result.advice` 不得被 Agent 覆盖 |
| Agent 只是候选 | Agent 输出 `final_action_candidate`，最终动作由 Evidence Gate 约束 |
| 不编造平台字段 | 平台字段缺失必须显式记录，Agent 不得补造 |
| block 门槛 | `block` 必须有明确证据或硬规则命中 |
| weak signal 门槛 | weak signal 只能支撑 `need_preview` |
| 纯图片保护 | 纯图片默认低风险，除非命中明确违规 |
| v4/v5 中国站保护 | v4/v5 中国站客户默认保护，除非命中明确违规 |
| 可审计可回放 | 所有关键判断必须记录版本、输入、节点输出和修正原因 |

## 5. 工程 Spec 列表

| Spec | 文件 | 主要内容 |
| --- | --- | --- |
| Interface Spec | `specs/01-interface-spec.md` | 输入输出、Pydantic schema、枚举、空值策略 |
| ReviewState Spec | `specs/02-review-state-spec.md` | `ReviewState` 字段、写入责任、状态流转 |
| LangGraph Workflow Spec | `specs/03-langgraph-workflow-spec.md` | 节点图、条件边、正常与异常路径 |
| Node Spec | `specs/04-node-spec.md` | 每个节点输入输出、核心逻辑、测试重点 |
| Rule Engine Spec | `specs/05-rule-engine-spec.md` | 硬规则、弱信号、保护因子、规则版本 |
| Agent Decision Spec | `specs/06-agent-decision-spec.md` | Agent 输入、输出 schema、prompt 约束、fallback |
| Evidence Gate Spec | `specs/07-evidence-gate-spec.md` | `block`、`need_preview`、`pass` 的证据门槛 |
| Audit Logger Spec | `specs/08-audit-logger-spec.md` | 审计字段、版本、回放策略、敏感字段 |
| Error Handling Spec | `specs/09-error-handling-spec.md` | 异常处理矩阵和 final_action 策略 |
| Test Spec | `specs/10-test-spec.md` | 单元、节点、workflow、回归 case 测试 |

## 6. 推荐实现目录

```text
content_agent/
  models/
  graph/
  adapters/
  rules/
  agent/
  evidence/
  audit/
  config/
tests/
  fixtures/
```

## 7. 推荐开发顺序

1. 实现 `specs/01-interface-spec.md` 中的模型和枚举。
2. 实现 `specs/02-review-state-spec.md` 中的 `ReviewState`。
3. 搭建 `specs/03-langgraph-workflow-spec.md` 的 LangGraph 骨架。
4. 按 `specs/04-node-spec.md` 实现各节点 mock 版本。
5. 按 `specs/05-rule-engine-spec.md` 实现最小规则层。
6. 按 `specs/06-agent-decision-spec.md` 实现 deterministic Agent 或 structured mock Agent。
7. 按 `specs/07-evidence-gate-spec.md` 实现最终约束。
8. 按 `specs/08-audit-logger-spec.md` 写审计日志。
9. 按 `specs/09-error-handling-spec.md` 补齐异常路径。
10. 按 `specs/10-test-spec.md` 建立回归 case。

## 8. 研发评审入口

研发评审时先看本文档确认系统边界，再看：

1. `DELTA.md`：确认相对架构方案新增了哪些工程约束。
2. `specs/01-interface-spec.md`：确认数据契约。
3. `specs/03-langgraph-workflow-spec.md`：确认流程和异常路径。
4. `specs/07-evidence-gate-spec.md`：确认最终动作门槛。
5. `specs/10-test-spec.md`：确认是否可验收。

