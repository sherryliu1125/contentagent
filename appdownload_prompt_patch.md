# App Download Fake Prompt Patch

本文档只承载本次针对 `online-prompt-v3.10.md` 的修改片段，便于复制粘贴。不要把本文整体粘入 prompt；按各节标注的位置替换或插入。

## 1. Risk Type Definitions

位置：`# Risk Type Definitions` 下第 13 条 `fake`。

操作：替换原 `fake` 定义。

```md
13. **fake**：冒充知名品牌、机构、平台、政府或官方实体作为主要风险，包括以官方或授权身份承载下载、安装、打开 App、描述文件安装或证书信任等流程。若仿冒与色情、赌博、金融交易等更高优先级风险同时出现，优先选择更高优先级类型。
```

## 2. Evidence Field Inventory

位置：`# Evidence Field Inventory` 表格中 `login` 和 `fake` 两行。

操作：替换这两行。

```md
| login | known_entity_branding, invitation_code_required, public_registration_entry, customer_service_entry, guest_access_entry |
| fake | suspected_brand_impersonation, known_entity_branding, official_entity_text_claim, app_download_or_install_entry, app_store_page_imitation, app_store_rating_or_review_display, official_app_store_or_apple_claim, ios_profile_or_trust_instruction, app_identity_mismatch |
```

## 3. Evidence Field Guide - fake

位置：`# Evidence Field Guide` 下的 `## fake` 小节。

操作：替换整个 `## fake` 小节内容。

```md
## fake

- `known_entity_branding`：可见知名品牌、官方样式、机构 Logo 或认证标识。单独出现不构成 `fake`。
- `suspected_brand_impersonation`：页面用知名或官方实体身份承载登录、支付、交易、投资、公共服务、下载、安装、打开 App、描述文件安装或证书信任等流程。不要判断实体是否真实官方。
- `official_entity_text_claim`：页面文字声明知名品牌、机构、平台、政府或官方实体身份，例如官方平台名、机构名、品牌名、政府/公共服务名、开发者身份或官方认证/授权声明。单独出现不构成 `fake`，不判断实体是否真实官方。
- `app_download_or_install_entry`：可见下载、立即下载、免费安装、安装中、打开 App 等 App 获取或安装入口。单独出现不构成 `fake`。
- `app_store_page_imitation`：页面呈现类似 App Store 详情页结构，例如 App 图标、名称、评分、排名、评论数、年龄分级、大小、版本、安装按钮、评论分布等；建议至少同时出现 3 类模块。单独出现不构成 `fake`。
- `app_store_rating_or_review_display`：可见评分、星级、排名、评论数、评论分布或用户评论等应用商店背书信息。单独出现不构成 `fake`。
- `official_app_store_or_apple_claim`：可见 Apple、苹果、App Store 级别的官方、授权、推荐、认证或开发者身份声明，例如 `Apple授权App`、`苹果官方推荐App`、`Apple Inc.`、`App Store官方推荐`。普通“某某官方App”不算该字段。
- `ios_profile_or_trust_instruction`：可见 iOS 描述文件、设备管理、去信任、信任企业级开发者、企业证书、安装描述文件等手动信任或侧载安装指引。单独出现不构成 `fake`。
- `app_identity_mismatch`：App 名称、图标、开发者身份、Apple/App Store 声明之间存在可见不匹配或异常，例如普通陌生 App 显示开发者为 `Apple Inc.`，或 App 名称/图标与 Apple 官方身份明显不对应。

### fake boundary

通用仿冒页面必须同时满足：

- `suspected_brand_impersonation`
- `known_entity_branding` 或 `official_entity_text_claim`

`known_entity_branding` 或 `official_entity_text_claim` 单独出现，不构成 `fake`。

### fake app download boundary

App 下载页不新增 `risk_type`。下载、安装、评分、仿 App Store 页面本身都不是违规结论。

仅当截图可见以下矛盾组合时，可选择 `risk_type="fake"`：

- `app_download_or_install_entry`
- `official_app_store_or_apple_claim`
- `ios_profile_or_trust_instruction`

若同时命中 `app_store_page_imitation`、`app_identity_mismatch` 或 `app_store_rating_or_review_display`，应一并输出对应 evidence。

以下情况不得单独判为 `fake`：

- 只有下载或安装入口
- 只有评分、评论、排名
- 只有仿 App Store 页面结构
- 只有 Apple、苹果、App Store、官方、授权等字样
- 只有描述文件或企业证书信任指引
- 普通品牌官网提供 App 下载
```

## 4. Notes

- 不新增 `appdownload` risk_type。
- 不在 v3.10 输出中增加 `confidence`。
- `site_or_entity_name` 从 v3.10 的 `login` 和 `fake` evidence 中移除。
- 色情 App 下载页仍优先由 `adult_app_or_brand` 触发 `risk_type="porn"`。
