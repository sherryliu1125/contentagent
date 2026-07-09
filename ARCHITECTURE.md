# 内容审核 Agent MVP 架构方案

## 1. 文档目的

本文档用于定义图片/网页快照内容审核 Agent 的 MVP 架构方案，供产品、算法、后端、风控策略和审核团队评审。

本方案基于当前 MVP 目标与 `online-prompt.txt` 中的 VLM 输出约束，重点说明系统定位、技术选型、模块设计、核心数据流、决策模型、开发方案、技术要求和 MVP 边界。

本文档不是最终接口 spec，也不是开发任务拆分文档。后续应在本架构方案确认后，再拆分 SDD、接口 spec、规则 spec 和实现任务。

## 2. 背景与目标

项目目标是建设一个面向图片和网页快照的内容审核 Agent。

审核链路分为两层：

1. VLM 基于图片或网页快照可见内容进行视觉初判。
2. Agent 结合 URL、客户等级、资源类型、人工规则和后续平台字段进行综合复核。

VLM 必须先输出以下字段：

```json
{
  "advice": "string",
  "page_type": "string",
  "risk_behavior": ["string"],
  "description": "string"
}
```

Agent 最终输出三类标签：

```text
block
need_preview
pass
```

MVP 阶段不直接调用平台自动封禁、下线或处罚接口。`block` 表示 Agent 判断应拦截，但早期仍进入人工重点复核；`pass` 早期也可以人工抽检。后续在准确率、误杀率和漏放率稳定后，再逐步减少人工查看。

## 3. 架构定位

本系统定位为：

```text
workflow-first, agent-enhanced 的受控审核 Agent
```

含义如下：

- `workflow-first`：核心审核链路由明确节点组成，流程稳定、可回放、可观测。
- `agent-enhanced`：Agent 只在受控节点内做综合复核、证据解释和疑难 case 归纳，不做开放式自主规划。
- `受控审核`：最终动作、证据门槛、保护规则和硬规则由系统约束，避免 Agent 直接成为不可控裁决器。

该定位决定了 MVP 不采用完全开放式多轮 Agent，也不把所有判断写成简单 if-else 脚本。系统应以确定性 workflow 承载业务边界，以规则层承载审核经验，以 Agent 承载综合解释和弱信号复核。

## 4. 核心业务原则

MVP 阶段遵循以下原则：

1. URL 一定可以拿到。
2. 其他平台字段可能暂时为空，系统必须兼容。
3. 纯图片默认低风险，除非命中明确违规特征。
4. v4/v5 中国站客户默认保护，除非命中明确违规特征。
5. 重点审核对象是带 URL 且非 v4/v5 中国站客户。
6. 硬规则和难例后续会持续补充，因此规则层必须配置化、可检索、可扩展。
7. `description` 保持为 VLM 视觉证据字段，不改名为其他字段。
8. VLM 的 `advice` 必须保留，Agent 不应删除或覆盖其原始语义。
9. `final_action` 只能输出 `block`、`need_preview`、`pass`。

## 5. 技术选型

### 5.1 Python

建议使用 Python 作为 MVP 主语言。

原因：

- 与 VLM、LLM、LangGraph、Pydantic、规则处理和批处理生态兼容性好。
- 适合快速实现 mock adapter、规则配置、审核工作流和离线评测。
- 后续接入真实 VLM、平台接口和批量任务时成本较低。

### 5.2 LangGraph

建议使用 LangGraph 管理审核工作流。

使用 LangGraph 的原因：

- 当前业务是有状态审核流程，不是单次 prompt 调用。
- 每个节点职责清晰：Input、VLM Adapter、URL/Resource Classifier、Platform Context Adapter、Rule Engine、Agent Decision Node、Evidence Builder、Audit Logger。
- 可以保留完整 `ReviewState`，便于排查误判来自 VLM、规则、平台上下文还是 Agent 综合判断。
- 支持条件边和节点编排，适合表达硬规则优先、保护因子降级、疑难 case 进入人工复核等流程。
- 后续可以扩展 checkpoint、batch、重试、人工反馈回流和版本化审计。

不建议在 MVP 中使用复杂多轮 ReAct Agent。内容审核场景需要可控性、可解释性和一致性，应该使用确定性 workflow + 受控 Agent 决策节点。

### 5.3 Pydantic

建议使用 Pydantic 定义所有核心数据结构。

Pydantic 的作用：

- 校验输入字段，例如 `url` 必填。
- 固定 VLM 输出字段：`advice`、`page_type`、`risk_behavior`、`description`。
- 约束 `final_action` 只能为 `block`、`need_preview`、`pass`。
- 定义 `ReviewState`、`ReviewInput`、`VLMResult`、`PlatformContext`、`RuleResult`、`ReviewOutput`。
- 为 mock adapter 和真实 adapter 提供统一数据契约。
- 支持 schema 版本化，便于后续兼容字段演进。

Pydantic 不负责业务决策，只负责数据边界、字段校验和结构一致性。

### 5.4 自研规则层

规则层建议自研，不建议完全交给 Agent 判断。

自研规则层负责：

- 硬规则命中。
- 保护因子判断。
- URL 风险特征识别。
- 弱信号组合。
- 内部风险评分。
- 证据等级计算。
- 规则版本记录。

MVP 可先使用 Python 规则函数 + 配置文件实现。后续可以逐步演进为规则管理平台、关键词索引、grep 检索或 RAG 难例库。

## 6. 总体架构设计

整体链路如下：

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

模块职责如下：

```text
Input:
  接收审核任务，完成基础字段校验，初始化 ReviewState。

VLM Adapter:
  基于图片或网页快照输出 advice、page_type、risk_behavior、description。

URL / Resource Classifier:
  识别 URL 特征、资源类型、是否纯图片、是否网页或落地页。

Platform Context Adapter:
  读取客户等级、站点区域、资源类型和后续平台字段。

Rule Engine:
  运行硬规则、保护因子、弱信号规则和风险评分。

Agent Decision Node:
  在受控上下文中进行综合复核，输出 final_action 候选和 agent_reason。

Evidence Builder:
  汇总 decisive_evidence、weak_signals、protective_factors，并校验证据门槛。

Audit Logger:
  记录输入、节点输出、版本信息、最终动作和异常信息。
```

## 7. 模块设计

### 7.1 Input

Input 模块负责接收外部审核任务，并转换为内部 `ReviewState`。

MVP 输入字段：

```json
{
  "case_id": "string",
  "image": "image_file_or_image_url",
  "url": "string",
  "customer_level": "string|null",
  "site_region": "string|null",
  "resource_type": "string|null"
}
```

技术要求：

- `url` 必填。
- `case_id` 可由外部传入；若为空，MVP 可生成本地 ID。
- `customer_level`、`site_region`、`resource_type` 允许为空。
- 不因平台字段缺失中断审核。
- 输入校验失败时应返回结构化错误，不进入 Agent 决策。

### 7.2 VLM Adapter

VLM Adapter 负责调用 VLM 或 mock VLM，并输出视觉初判。

输出字段必须为：

```json
{
  "advice": "good",
  "page_type": "other_unknown",
  "risk_behavior": [],
  "description": "未见明显异常"
}
```

技术要求：

- `advice` 必须保留。
- `description` 必须保留，不改名。
- `page_type` 应遵循 `specs/12-page-type-spec.md` 中的枚举；`risk_behavior`、`advice` 应遵循线上审核 prompt 中的枚举。
- VLM 只能基于截图可见内容判断，不使用 URL、客户等级或平台字段。
- VLM 失败时不得默认 `pass`，应进入 `need_preview` 或错误复核路径。

MVP 阶段先使用 mock adapter。真实 VLM 接入后，应保持 adapter 输入输出结构不变。

### 7.3 URL / Resource Classifier

该模块负责识别 URL 和资源类型相关信号。

核心输出：

```text
has_url
is_pure_image
is_website_or_landing_page
resource_type_inferred
url_features
```

判断来源：

- 外部传入的 `resource_type`。
- URL path、query、host、suffix。
- VLM 输出的 `page_type`。
- 图片内容是否呈现网页结构、按钮、表单、导航、下载入口。

MVP 应重点识别：

- 纯图片。
- 登录/认证页。
- 下载/安装页。
- 支付/充值/提现页。
- 落地页/H5。
- 私聊客服或加群入口。
- 邀请码、推广码、短链、跳转页等 URL 弱信号。

### 7.4 Platform Context Adapter

该模块负责读取平台上下文。

MVP 阶段字段可以为空，但应预留结构：

```text
customer_level
site_region
resource_type
account_status
business_category
historical_risk
platform_labels
raw_platform_payload
```

技术要求：

- 支持 mock 数据。
- 支持真实平台字段后续替换。
- 保留 raw payload，便于排查字段映射问题。
- 标准化字段与原始字段分开存储。
- 平台字段缺失时降低置信度，但不直接失败。

### 7.5 Rule Engine

Rule Engine 是 MVP 的核心可控决策层。

职责：

- 识别明确违规。
- 识别保护因子。
- 汇总弱信号。
- 计算内部风险分。
- 输出证据等级。
- 记录规则命中详情。

核心输出：

```text
hard_rule_hits
has_explicit_violation
has_weak_risk_signal
weak_signals
decisive_evidence
protective_factors
risk_score
risk_level
confidence
evidence_level
rule_version
```

硬规则示例：

- 色情：约炮、裸聊、成人直播、无码、同城交友、明显裸露。
- 赌博：棋牌、捕鱼、百家乐、体育投注、真人视讯、彩金、返水、流水、赔率、投注。
- 金融诈骗：提现、红包、收益、返现、刷单、任务返佣、垫付、虚拟币空投、高收益承诺。
- 仿冒：冒充银行、App Store、官方客服、企业认证、官方旗舰店、官方公告。
- 其他违规：未成年人色情、涉政涉军暴恐、外挂、私服、VPN、代理、翻墙节点。

保护因子：

- `is_pure_image = true`
- `is_v4_v5_china_customer = true`

保护因子不覆盖明确违规。命中明确违规时，即使存在保护因子，也不能直接输出 `pass`。

### 7.6 Agent Decision Node

Agent Decision Node 负责综合复核。

输入：

- VLM 输出。
- URL / Resource Classifier 输出。
- Platform Context Adapter 输出。
- Rule Engine 输出。
- 当前 `ReviewState`。

输出：

```text
final_advice
final_action
agent_reason
confidence
```

技术要求：

- `final_action` 只能为 `block`、`need_preview`、`pass`。
- Agent 不得删除或改写原始 `vlm_result.advice`。
- Agent 不得编造平台字段。
- Agent 不得把弱信号描述成明确违规。
- Agent 应解释为什么选择当前 `final_action`。
- Agent 输出必须经过 Evidence Builder 校验。

Agent 的定位是受控综合判断，不是唯一裁决源。

### 7.7 Evidence Builder

Evidence Builder 负责构造面向人工复核的证据链。

输出字段：

```text
decisive_evidence
weak_signals
protective_factors
agent_reason
evidence_level
```

证据门槛：

- `block` 必须有 `decisive_evidence` 或强硬规则命中。
- `need_preview` 可以只有 `weak_signals`。
- `pass` 需要说明低风险原因或保护因子。

如果 Agent 给出的 `final_action` 与证据门槛冲突，Evidence Builder 应按规则修正或打回。例如证据不足的 `block` 应降为 `need_preview`。

### 7.8 Audit Logger

Audit Logger 负责记录完整审核轨迹。

建议记录：

```text
case_id
input_snapshot
vlm_result
url_features
platform_context_snapshot
rule_hits
agent_reason
final_action
prompt_version
rule_version
model_version
adapter_version
graph_version
timestamp
errors
warnings
```

审计日志用于：

- 人工复盘。
- 误判分析。
- 规则迭代。
- 模型版本对比。
- 后续评测集构建。

## 8. 核心数据流

完整流程：

```text
1. 接收输入
2. 校验 ReviewInput
3. 初始化 ReviewState
4. 调用 VLM Adapter，得到 vlm_result
5. 解析 URL 和资源类型
6. 获取平台上下文
7. 执行规则引擎
8. 执行 Agent 综合复核
9. 构建证据链并校验 final_action
10. 写入审计日志
11. 输出 ReviewOutput
```

关键数据关系：

```text
vlm_result.advice:
  VLM 基于截图可见内容给出的视觉初判。

final_advice:
  Agent 综合复核后的风险类别建议。

final_action:
  最终动作标签，只能为 block、need_preview、pass。
```

VLM 初判和 Agent 最终动作必须分开保存。这样后续排查时可以区分：

- VLM 是否看错图。
- 规则是否误命中。
- 平台字段是否缺失或映射错误。
- Agent 是否过度推理。

## 9. 决策模型

MVP 采用以下混合决策模型：

```text
Hard Rule
+ Protective Gate
+ Risk Score
+ Evidence Gate
+ Agent Explanation
```

### 9.1 Hard Rule

Hard Rule 用于处理明确违规。

典型逻辑：

```text
if has_explicit_violation:
  evidence_level = decisive
  final_action = block
```

硬规则应基于截图可见证据、URL 特征和人工规则，不依赖 Agent 主观发挥。

### 9.2 Protective Gate

Protective Gate 用于保护低风险 case。

典型逻辑：

```text
if no explicit violation and is_pure_image:
  final_action = pass

if no explicit violation and is_v4_v5_china_customer:
  final_action = pass
```

保护因子只降低弱风险 case 的处置强度，不覆盖明确违规。

### 9.3 Risk Score

Risk Score 用于处理疑难 case 和弱信号组合。

内部可以计算数值分：

```text
visual_risk_score
url_risk_score
platform_risk_score
weak_signal_score
protection_score
```

对外不建议展示具体分数，避免假精确感。对外展示：

```text
risk_level: low / medium / high
confidence: low / medium / high
evidence_level: none / weak / decisive
```

### 9.4 Evidence Gate

Evidence Gate 是最终动作的硬约束。

```text
block:
  必须有 decisive_evidence 或强硬规则命中。

need_preview:
  可以只有 weak_signals。
  适合证据不足但风险组合明显的 case。

pass:
  必须无明确违规。
  应有低风险原因或保护因子。
```

### 9.5 Agent Explanation

Agent Explanation 用于给人工复核员提供清晰解释。

要求：

- 引用可见证据、规则命中或保护因子。
- 不输出无法验证的推断。
- 不把 VLM 没看到的内容写成事实。
- 不用营销化语言。
- 不只输出标签，必须说明理由。

## 10. final_action 设计

### 10.1 block

定义：

Agent 判断应拦截。MVP 阶段仍进入人工重点复核，后续准确率稳定后可减少人工查看。

证据门槛：

- 命中明确硬规则。
- 或存在 `decisive_evidence`。
- 证据能被人工复核员直接理解。

典型场景：

- 明确色情。
- 明确赌博。
- 明确金融诈骗。
- 明确仿冒。
- 明确涉政涉军暴恐。
- 明确外挂、私服、VPN、代理推广。

### 10.2 need_preview

定义：

Agent 判断证据不足以直接拦截或放行，需要人工复核。

证据门槛：

- 存在 `weak_signals`。
- 未达到 `block` 的证据强度。
- 不满足明确 `pass` 条件。

典型场景：

- 带 URL 且非 v4/v5 中国站客户。
- 登录认证页出现邀请码、客服入口、验证码收集。
- 下载页存在诱导安装但缺少明确违规证据。
- URL 异常但截图证据较弱。
- VLM `advice` 可疑，但 `description` 证据不足。

### 10.3 pass

定义：

Agent 判断可放行。MVP 阶段可人工抽检，后续准确率稳定后减少人工查看。

证据门槛：

- 无明确违规。
- 无明显弱信号组合。
- 或保护因子成立。

典型场景：

- 纯图片且未命中明确违规。
- v4/v5 中国站客户且未命中明确违规。
- 普通内容页、正常商品页、正常品牌展示页。

## 11. ReviewState 状态对象设计

LangGraph 中建议维护统一 `ReviewState`。

字段建议：

```text
case_id
image
url

customer_level
site_region
resource_type
platform_context
raw_platform_payload

vlm_result.advice
vlm_result.page_type
vlm_result.risk_behavior
vlm_result.description

has_url
is_pure_image
is_website_or_landing_page
is_v4_v5_china_customer

url_features
resource_type_inferred

hard_rule_hits
has_explicit_violation
has_weak_risk_signal
weak_signals
decisive_evidence
protective_factors

risk_score
risk_level
confidence
evidence_level

final_advice
final_action
agent_reason

audit
errors
warnings
```

设计要求：

- `vlm_result` 必须保留原始 VLM 输出。
- `final_action` 只能为 `block`、`need_preview`、`pass`。
- `errors` 与 `warnings` 不应直接覆盖最终结果，应由 Evidence Builder 或异常策略统一处理。
- 所有关键节点输出都应能从 `ReviewState` 中追踪。

## 12. 接口占位策略

### 12.1 VLM 接口 mock

MVP 阶段使用 mock VLM Adapter。

mock 策略：

- 从固定 JSON 样例读取。
- 或根据测试 case 返回预设 VLM 结果。
- 输出严格符合 `advice`、`page_type`、`risk_behavior`、`description`。

替换真实接口时：

- Adapter 对外接口不变。
- 增加模型版本记录。
- 增加超时、重试和异常处理。
- VLM 调用失败时进入 `need_preview`，不默认 `pass`。

### 12.2 平台接口 mock

MVP 阶段使用 mock Platform Context Adapter。

mock 策略：

- URL 必填。
- `customer_level`、`site_region`、`resource_type` 可为空。
- 可通过本地 JSON 模拟 v4/v5 中国站客户、普通客户和未知客户。

替换真实接口时：

- 保持标准化输出结构不变。
- 保留原始平台字段。
- 明确字段映射版本。
- 平台接口失败时不直接中断，可降级为字段缺失并降低置信度。

### 12.3 自动处置接口

MVP 不接自动处置接口。

以下动作不在 MVP 中实现：

```text
takedown
auto_reject
auto_ban
auto_freeze
```

后续只有在证据门槛、误杀评估、人工复核回流和平台接口稳定后，再评估自动处置能力。

## 13. 开发方案

建议按四个阶段开发。

### 阶段一：基础链路

目标：跑通单条 case 从输入到输出的完整链路。

内容：

- 定义 Pydantic 数据结构。
- 实现 `ReviewState`。
- 实现 mock VLM Adapter。
- 实现 mock Platform Context Adapter。
- 搭建 LangGraph 基础节点。
- 输出基础 `ReviewOutput`。

验收标准：

- 能输入一张图片或图片引用和 URL。
- 能得到 VLM mock 结果。
- 能输出 `final_action`。
- 输出结构稳定。

### 阶段二：规则与保护因子

目标：实现可控审核逻辑。

内容：

- 实现硬规则。
- 实现弱信号规则。
- 实现纯图片保护。
- 实现 v4/v5 中国站客户保护。
- 实现风险等级和证据等级。

验收标准：

- 明确违规 case 输出 `block`。
- 纯图片弱风险 case 输出 `pass`。
- v4/v5 中国站客户弱风险 case 输出 `pass`。
- 带 URL 且非保护客户的弱信号 case 输出 `need_preview`。

### 阶段三：Agent 综合复核

目标：引入受控 Agent 决策节点。

内容：

- 设计 Agent 输入上下文。
- 约束 Agent 输出 schema。
- 实现 `agent_reason`。
- 防止 Agent 编造字段。
- 将 Agent 输出接入 Evidence Builder。

验收标准：

- Agent 能解释 `final_action`。
- Agent 输出不违反证据门槛。
- Agent 不覆盖原始 `vlm_result.advice`。

### 阶段四：审计与评测

目标：支持人工复盘和后续迭代。

内容：

- 实现 Audit Logger。
- 记录 prompt、规则、模型和 graph 版本。
- 准备典型 case 测试集。
- 对 `block`、`need_preview`、`pass` 做回归验证。

验收标准：

- 每条 case 可回放。
- 能定位 VLM、规则、平台字段或 Agent 的判断来源。
- 能基于人工反馈沉淀规则和难例。

## 14. 技术要求

### 14.1 数据结构要求

- 所有核心输入输出必须 Pydantic 化。
- 枚举字段必须限制取值。
- `final_action` 必须限制为 `block`、`need_preview`、`pass`。
- VLM 字段必须保持 `advice`、`page_type`、`risk_behavior`、`description`。
- 空平台字段必须被显式表示，不允许隐式丢失。

### 14.2 可观测性要求

- 每个节点应记录输入摘要和输出摘要。
- 每个最终结果应包含规则版本和 prompt 版本。
- 每个 `block` 应能追溯到明确证据。
- 每个 `pass` 应能说明低风险或保护原因。
- 每个 `need_preview` 应能说明弱信号。

### 14.3 稳定性要求

- VLM 失败不得默认放行。
- 平台字段缺失不得导致系统崩溃。
- Agent 输出不合法时应回退到规则层结果。
- 证据不足时不得输出高置信 `block`。
- 明确违规时保护因子不得直接覆盖为 `pass`。

### 14.4 可扩展性要求

- 规则可配置。
- 难例可沉淀。
- Adapter 可替换。
- Schema 可版本化。
- LangGraph 节点可新增。
- 审计日志可用于离线评测。

## 15. MVP 边界

### 15.1 MVP 做什么

MVP 实现：

- 图片/网页快照审核主链路。
- VLM mock 初判。
- 平台上下文 mock。
- URL 和资源类型基础识别。
- 硬规则与弱信号规则。
- 纯图片保护。
- v4/v5 中国站客户保护。
- Agent 综合复核。
- Evidence Builder。
- Audit Logger。
- `block`、`need_preview`、`pass` 三类输出。

### 15.2 MVP 不做什么

MVP 不实现：

- 不接真实 VLM 接口。
- 不接真实平台回写接口。
- 不自动封禁、下线或处罚。
- 不建设完整人工规则后台。
- 不做长期记忆。
- 不做自动学习。
- 不做复杂多轮 Agent 自主规划。
- 不一次性穷尽全部平台字段。
- 不一次性补齐全部难例。
- 不把内部风险分数作为主要人工展示依据。

## 16. 输出结构建议

MVP 输出建议如下：

```json
{
  "case_id": "string",
  "vlm_result": {
    "advice": "fake",
    "page_type": "login_auth",
    "risk_behavior": [
      "credential_collection",
      "private_contact_inducement"
    ],
    "description": "登录页要求邀请码\\n客服入口突出"
  },
  "final_advice": "fake",
  "final_action": "need_preview",
  "risk_level": "medium",
  "confidence": "low",
  "evidence_level": "weak",
  "agent_reason": "该页面缺少明确违规证据，但登录认证页出现邀请码和客服入口，符合疑似诈骗登录页弱信号组合，建议人工复核。",
  "decisive_evidence": [],
  "weak_signals": [
    "登录页要求邀请码",
    "客服入口突出",
    "疑似账号/验证码收集"
  ],
  "protective_factors": [],
  "audit": {
    "prompt_version": "string",
    "rule_version": "string",
    "model_version": "string|null",
    "graph_version": "string"
  }
}
```

## 17. 后续文档拆分建议

本架构方案确认后，建议拆分为以下正式文档。

### 17.1 SDD：系统设计文档

内容：

- LangGraph 节点设计。
- `ReviewState` 详细字段。
- 状态流转。
- 条件边设计。
- 异常处理。
- 审计日志。
- 版本管理。

### 17.2 Interface Spec：接口与数据结构规范

内容：

- `ReviewInput`。
- `VLMResult`。
- `PlatformContext`。
- `RuleResult`。
- `ReviewOutput`。
- 枚举定义。
- 空值策略。
- mock 与真实接口替换约定。

### 17.3 Rule Spec：规则与证据规范

内容：

- 硬规则分类。
- 弱信号分类。
- 保护因子定义。
- 风险等级定义。
- 证据等级定义。
- `block`、`need_preview`、`pass` 的证据门槛。
- 难例沉淀规范。

### 17.4 Development Plan：开发任务拆分

内容：

- 里程碑。
- 模块任务。
- 测试任务。
- 评测样例。
- 验收标准。
- 后续真实接口接入计划。

## 18. 待团队确认事项

后续需要团队确认：

- v4/v5 中国站客户的准确判断字段。
- 平台侧稳定可提供的字段列表。
- 平台已有资源类型枚举。
- 真实 VLM 接口输入输出约束。
- 人工审核已有硬规则和难例清单。
- 哪些风险类型必须高优先级人审。
- 哪些 case 可进入抽检而非全量人工查看。
- 自动处置能力的后续接入条件。
