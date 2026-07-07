# 内容审核 Agent MVP 规划文档

## 1. 背景与目标

当前项目目标是建设一个面向图片和网页快照的内容审核 Agent。系统需要先接收图片输入，由 VLM 基于截图可见内容完成视觉初判，再由 Agent 结合 URL、客户等级、资源类型、人工规则和后续平台字段，对风险进行综合复核，最终输出 `block`、`need_preview`、`pass` 三类审核标签和证据说明。

当前处于公司外部开发阶段，暂未对接真实 VLM 接口和平台接口。公司内部已有 VLM 接入路径、图片输入 spec、批量处理 spec 和平台对接接口。MVP 阶段先使用 mock 或占位 adapter 跑通核心审核逻辑，不直接依赖内部真实接口。

MVP 的核心目标不是立即完全替代人工，而是先做成一个可解释的预审助手：

- 对明显违规内容输出 `block`，由人工重点复核。
- 对疑难 case 输出 `need_preview`，汇总弱信号，辅助人工判断。
- 对低风险内容输出 `pass`，降低人工关注优先级。
- 输出清晰的证据链，避免只有标签、没有依据。
- 为后续补充人工经验、硬规则、平台字段和真实接口预留扩展位置。

## 2. MVP 定位

MVP 定位为：

```text
Workflow-first, Agent-enhanced 的视觉内容风控预审系统。
```

它不是完全开放式自主 Agent，也不是单纯的 if-else 脚本。它采用受控审核流程：

```text
图片/网页快照
  -> VLM 视觉初判
  -> URL 与资源类型识别
  -> 客户上下文读取
  -> 保护因子与风险因子并行评估
  -> 硬规则 + 风险评分 + Agent 复核
  -> 输出人工复核建议与证据链
```

其中 VLM 和 Agent 共同组成整体审核 Agent：

- VLM 负责看图，输出视觉初判。
- Agent 负责综合复核，结合 URL、客户等级、规则和人工经验，给出最终审核建议。

## 3. MVP 核心业务原则

MVP 阶段优先聚焦“带 URL 的网页/落地页风险审核”。

当前业务判断原则如下：

1. 纯图片风险默认较低。
   - 除非 VLM 命中明确违规特征，否则不建议处置。

2. v4/v5 中国站客户默认保护。
   - 除非命中明确违规特征，否则不建议处置。

3. 带 URL 且非 v4/v5 中国站客户是重点审核对象。
   - 这类 case 进入更完整的 Agent 复核流程。

4. 纯图片保护和客户等级保护不是严格先后顺序。
   - 它们作为并行保护因子共同参与决策。

5. MVP 输出三类最终标签。
   - `block`：Agent 判断应拦截，MVP 阶段仍由人工复核。
   - `need_preview`：Agent 判断需要人工复核。
   - `pass`：Agent 判断可放行，MVP 阶段可抽检。

## 4. MVP 输入

MVP 输入包括：

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

字段说明：

- `case_id`：审核任务 ID，MVP 可由外部传入或本地生成。
- `image`：图片文件路径、图片 URL 或内部图片引用，MVP 阶段可 mock。
- `url`：必填字段。平台侧确认 URL 一定可以拿到。
- `customer_level`：客户等级，可为空。重点识别 v4/v5。
- `site_region`：站点区域，可为空。重点识别中国站。
- `resource_type`：资源类型，可为空。MVP 可先由 URL 和图片上下文推断。

## 5. VLM 初判输出

VLM 仍然按照当前 `online-prompt.txt` 的结构输出，不改字段名。

```json
{
  "advice": "标签",
  "page_type": "页面场景枚举",
  "risk_behavior": ["风险行为枚举"],
  "description": "简短证据"
}
```

字段含义：

- `advice`：VLM 基于截图可见内容给出的视觉初判标签。
- `page_type`：页面场景。
- `risk_behavior`：截图中可见的风险行为。
- `description`：基于截图可见内容的简短证据说明。

MVP 中需要在内部区分：

- `vlm_result.advice`：VLM 初判标签。
- `final_advice`：Agent 综合复核后的建议标签。

这样后续排查误判时，可以区分是 VLM 初判问题，还是 Agent 复核逻辑问题。

## 6. 并行评估因子

MVP 不将纯图片、客户等级、URL 判断写成简单线性流程，而是先并行计算一组 gating factors。

```text
has_url
is_pure_image
is_website_or_landing_page
is_v4_v5_china_customer
has_explicit_violation
has_weak_risk_signal
```

字段说明：

- `has_url`：是否存在 URL。MVP 中理论上始终为 true，但仍保留该因子。
- `is_pure_image`：是否为纯图片内容，而非网页/落地页快照。
- `is_website_or_landing_page`：是否为网站、H5、App 下载页、登录页、交易页等可交互页面。
- `is_v4_v5_china_customer`：是否为 v4/v5 中国站客户。
- `has_explicit_violation`：是否命中明确违规证据。
- `has_weak_risk_signal`：是否存在弱风险组合，但证据不足以直接定性。

## 7. 决策模型

MVP 采用混合决策模型：

```text
Hard Rule 硬规则
  + Protective Gate 保护因子
  + Risk Score 风险评分
  + Evidence Gate 证据门槛
  + Agent Explanation 解释输出
```

### 7.1 硬规则

硬规则指截图中出现足够明确、可直接引用的违规证据。它不依赖主观感觉，人工审核员看到后也容易认可。

示例硬规则包括：

色情类：

- 明确约炮、裸聊、同城交友、成人直播、无码、激情视频等文字。
- 明显裸露、三点暴露、强性暗示画面。
- 色情内容同时伴随下载、加好友、充值、提现入口。

赌博类：

- 澳门、新葡京、威尼斯人、银河娱乐等赌场或博彩实体。
- 棋牌、捕鱼、彩票、体育投注、真人视讯等博彩场景。
- 彩金、返水、流水、赔率、投注、首存、包赔等博彩黑话。
- 澳门荷官、真人荷官、百家乐、老虎机等明确博彩元素。

金融诈骗类：

- 提现、红包、收益、返现、余额等强诱导。
- 刷单、任务返佣、点赞赚钱、垫付等任务诈骗特征。
- 虚拟币空投、稳赚理财、高收益承诺。
- 异常支付、充值、转账、私下交易。
- 虚假商城价格或销量明显异常。

仿冒类：

- 冒充银行、App Store、官方客服、企业认证。
- 冒充知名品牌官方旗舰店。
- 官方公告、认证标识、证书、Logo 被异常使用。

其他明确违规：

- 未成年人色情。
- 明确涉政、涉军、暴恐、分裂内容。
- 明确售卖外挂、辅助、私服。
- 明确推广 VPN、代理、翻墙节点、机场。

硬规则在 MVP 中先以人工配置和代码规则形式维护。后续可以将已总结的难例和规则通过配置文件、grep 检索、RAG 知识库或规则管理平台逐步补充。

### 7.2 保护因子

保护因子用于降低弱风险 case 的处置强度。

MVP 中主要保护因子：

- `is_pure_image = true`
- `is_v4_v5_china_customer = true`

保护因子不覆盖明确违规。也就是说：

```text
如果命中明确违规，仍然需要进入高优先级复核。
如果没有明确违规，保护因子可以将 case 降为 pass 或低优先级。
```

### 7.3 风险评分

风险评分用于处理疑难 case 和弱信号组合。

典型场景：

```text
普通用户看像正常登录页；
风控审核员会注意到邀请码、客服入口、验证码收集、下载诱导、URL 异常等组合信号。
```

这类 case 通常没有一锤定音证据，但值得人工复核。

MVP 的风险评分不直接对人工展示具体分数，避免产生假精确感。系统内部可以计算分数，最终对外输出：

```text
risk_level: low / medium / high
confidence: low / medium / high
evidence_level: none / weak / decisive
```

### 7.4 证据门槛

MVP 要求不同最终标签具备不同证据强度：

```text
block 必须有 decisive_evidence 或强硬规则命中。
need_preview 可以只有 weak_signals。
pass 需要说明保护因子或低风险原因。
```

MVP 阶段输出 `block` 不等于平台自动封禁，而是表示 Agent 模拟人工判断后认为应拦截。早期仍由人工复核 `block` 和 `pass`；后续准确率稳定后，可以逐步减少对 `block` 和 `pass` 的人工复核。

## 8. 输出结果

MVP 输出面向人工复核的三类最终标签，不直接调用平台自动封禁接口。

建议输出结构：

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
    "model_version": "string|null"
  }
}
```

字段说明：

- `final_advice`：Agent 综合复核后的建议标签。
- `final_action`：Agent 模拟人工判断后的最终标签，只能为 `block`、`need_preview`、`pass`。
- `risk_level`：风险优先级。
- `confidence`：Agent 对当前判断的把握程度。
- `evidence_level`：证据强度。
- `agent_reason`：给人工看的综合解释。
- `decisive_evidence`：一锤定音证据列表。
- `weak_signals`：弱风险信号列表。
- `protective_factors`：导致降低处置强度的保护因子。
- `audit`：审计信息，记录版本，便于后续复盘。

## 9. final_action 定义

MVP 支持以下三类最终标签：

```text
block
need_preview
pass
```

含义：

- `block`：Agent 判断应拦截。MVP 阶段仍进入人工重点复核；后期准确率稳定后，可减少人工查看。
- `need_preview`：Agent 判断证据不足以直接拦截或放行，需要人工复核。
- `pass`：Agent 判断可放行。MVP 阶段可人工抽检；后期准确率稳定后，可减少人工查看。

MVP 暂不接入平台自动处置动作：

```text
takedown
auto_reject
```

这些自动处置动作留到后续阶段，在证据门槛、误杀评估和平台接口稳定后再接入。

## 10. 典型决策路径

### 10.1 明确违规

```text
VLM 命中 gamble / porn / finance / fake 等明确证据
  -> has_explicit_violation = true
  -> evidence_level = decisive
  -> final_action = block
```

即使是纯图片或 v4/v5 中国站客户，也不直接 pass。MVP 阶段 `block` 仍由人工重点复核。

### 10.2 纯图片弱风险

```text
资源类型为纯图片
VLM 未命中明确违规
仅有轻微异常或弱信号
  -> protective_factors 包含 pure_image
  -> final_action = pass
```

### 10.3 v4/v5 中国站客户弱风险

```text
客户为 v4/v5 中国站
VLM 未命中明确违规
  -> protective_factors 包含 v4_v5_china_customer
  -> final_action = pass
```

### 10.4 带 URL 的疑似登录页

```text
页面为 login_auth
出现邀请码、客服入口、验证码收集
无明确色情/赌博/金融/仿冒证据
  -> evidence_level = weak
  -> risk_level = medium
  -> final_action = need_preview
```

### 10.5 带 URL 且非保护客户

```text
存在 URL
非 v4/v5 中国站客户
VLM 初判可疑或存在多个弱信号
  -> 进入完整 Agent 复核
  -> 根据风险等级输出 need_preview 或 block
```

## 11. MVP 暂不做的事情

为了尽快形成可展示成果，MVP 暂不做以下内容：

- 不接真实 VLM 接口。
- 不接真实平台回写接口。
- 不调用平台自动封禁或下线接口。
- 不建设完整人工规则后台。
- 不做长期记忆或自动学习。
- 不做复杂多轮 Agent 自主规划。
- 不要求一次性穷尽全部平台字段。
- 不要求一次性补齐全部难例。

这些能力后续可以逐步接入。

## 12. 后续演进方向

MVP 跑通后，可以按以下方向演进：

1. 接入真实 VLM adapter。
2. 接入平台字段 adapter。
3. 将硬规则和难例沉淀为配置文件或规则库。
4. 将人工复核结果回流为评测集。
5. 建立难例检索能力，例如 grep、关键词索引或 RAG。
6. 建立 batch 处理能力。
7. 建立审计日志和版本追踪。
8. 在误杀率和漏放率稳定后，再评估自动处置能力。

## 13. 当前待团队确认问题

后续需要团队补充或确认：

- 平台除 URL 外还可以稳定提供哪些字段。
- v4/v5 中国站客户的准确判断字段是什么。
- 纯图片和网页快照的资源类型是否平台已有字段。
- 当前人工总结的硬规则和难例清单。
- 哪些风险标签必须高优先级人审。
- 哪些 case 可以只记录不触发人工。
- 后续是否需要自动处置，以及自动处置的证据门槛。
