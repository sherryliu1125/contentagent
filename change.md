# Page Type Field Change Log

## Purpose

This file records the process of fixing fields for each page type.

Goal:

- Fix the `page_type` value.
- Fix the `risk_behavior` fields for this page type.
- Fix the `visual_signals` fields for this page type.
- Use these fields as stable inputs for the rule engine.

Current design principle:

- `page_type` defines the page scenario.
- `risk_behavior` defines what user action or task the page is guiding.
- `visual_signals` defines directly visible risk signals.
- The rule engine combines `page_type + risk_behavior + visual_signals` to decide the final advice.

## Allowed Page Types

Current allowed `page_type` values:

- `mall`
- `payment`
- `rebate`
- `login`
- `investment`
- `crypto`
- `digital_goods`
- `vpn`
- `porn`
- `gambling`
- `game`
- `politic`
- `general`
- `other`

## Page Type: investment

### Final Field Structure

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
  }
}
```

### Field Meaning

#### page_type

`investment`

The page is an investment-related service page, including stock trading, fund investment, commodity or spot trading, virtual asset investment, securities account opening, financial product investment, or investment platforms using company or institution packaging.

#### risk_behavior

`trading`

Whether the page shows trading, market quotes, prices, yield, status such as "trading", project lists, holdings, or other investment transaction elements.

`fund_operation`

Whether the page shows fund-related operations, such as deposit, recharge, withdrawal, buy, subscribe, wallet, assets, balance, or holdings.

`login&register`

Whether the page shows login or registration entry, such as login button, register button, account input, phone number input, password input, or similar account access elements.

`open_account`

Whether the page shows account opening entry, such as open account, online account opening, create trading account, or similar onboarding elements.

#### visual_signals

`institutional_branding`

Whether the page uses institution or company packaging, such as exchange names, financial institution names, listed company names, energy company names, state-owned enterprise names, logos, official-looking buildings, or brand-style presentation.

`customer_service_entry`

Whether the page shows a clear customer service or support entry, such as customer service, online customer service, contact customer service, support chat, consultation, or advisor entry.

`invitation_code_required`

Whether the page requires or displays an invitation code, referral code, agent code, invite-only code, or similar access requirement.

### Rule Direction

The rule engine should not rely on a single visual signal alone. It should combine the page type, behavior, and signals.

Example rule patterns:

```text
page_type = investment
AND risk_behavior.fund_operation = true
=> finance
```

```text
page_type = investment
AND risk_behavior.trading = true
AND visual_signals.institutional_branding = true
=> finance
```

```text
page_type = investment
AND risk_behavior.login&register = true
AND visual_signals.customer_service_entry = true
AND visual_signals.institutional_branding = true
=> finance
```

```text
page_type = investment
AND visual_signals.invitation_code_required = true
AND (
  risk_behavior.login&register = true
  OR risk_behavior.open_account = true
)
=> finance
```

## Page Type: mall

### Final Field Structure

```json
{
  "page_type": "mall",
  "risk_behavior": {
    "mall_transaction": true
  },
  "visual_signals": {
    "mall_brand_visible": true,
    "visible_brand_names": [],
    "product_info_visible": true,
    "transaction_entry_visible": true
  }
}
```

### Field Meaning

#### page_type

`mall`

The page is a mall, shopping, store, product, order, or goods transaction page.

#### risk_behavior

`mall_transaction`

Whether the page shows or guides goods transaction behavior, such as browsing products, placing an order, buying, adding to cart, submitting an order, contacting seller, or paying.

#### visual_signals

`mall_brand_visible`

Whether the page visibly shows a mall, store, platform, brand, logo, trademark, or name.

This field only records visible brand or platform presentation. It does not determine whether the brand is impersonated.

`visible_brand_names`

The readable brand, platform, mall, or store names visible in the screenshot.

This is an open list extracted from the screenshot, not a fixed whitelist or blacklist.

`product_info_visible`

Whether the page visibly shows product information, such as product image, product name, price, specification, inventory, sales volume, or product details.

`transaction_entry_visible`

Whether the page visibly shows a transaction entry, such as buy, place order, add to cart, submit order, pay, contact seller, or similar shopping action.

### Rule Direction

The rule engine should not ask VLM to decide whether the mall is impersonated. VLM should only extract visible mall, brand, product, and transaction signals.

Example rule pattern:

```text
page_type = mall
AND risk_behavior.mall_transaction = true
AND visual_signals.mall_brand_visible = true
AND visual_signals.transaction_entry_visible = true
=> possible_mall_fraud / need_preview
```

## Page Type: rebate

### Final Field Structure

```json
{
  "page_type": "rebate",
  "risk_behavior": {
    "task_commission_rebate": true,
    "advance_payment_or_deposit": true
  },
  "visual_signals": {
    "task_or_order_rebate_visible": true,
    "commission_or_cashout_visible": true,
    "advance_payment_or_recharge_visible": true,
    "agent_recruitment_visible": true,
    "supply_or_substation_entry": true
  }
}
```

### Field Meaning

#### page_type

`rebate`

The page is a task scam, fake order scam, fake job, task commission, order brushing, paid like/follow, rebate, cashout, or commission task page.

#### risk_behavior

`task_commission_rebate`

Whether the page shows or guides users to earn commission, rebate, cashback, reward, or income by completing tasks, brushing orders, taking orders, grabbing orders, liking, following, placing orders, or similar task actions.

`advance_payment_or_deposit`

Whether the page shows or guides users to pay first before receiving commission, rebate, cashout, or higher task income, such as recharge, advance payment, deposit, prepaid balance, task unlock, order supplement, membership upgrade, or guarantee money.

#### visual_signals

`task_or_order_rebate_visible`

Whether the page visibly shows task, order brushing, taking orders, grabbing orders, paid like/follow, order task, task hall, part-time task, or task earning elements.

`commission_or_cashout_visible`

Whether the page visibly shows commission, rebate, cashback, reward, income, balance, withdraw, cashout, available withdrawal, arrival, daily earning, or stable earning elements.

`advance_payment_or_recharge_visible`

Whether the page visibly shows recharge, advance payment, prepaid balance, deposit, order supplement, task purchase, task unlock, membership upgrade, guarantee money, or similar pay-first elements.

`agent_recruitment_visible`

Whether the page visibly shows agent recruitment, invite friends, team income, promotion earning, substation, subordinate, referral reward, or similar agent/distribution information.

`supply_or_substation_entry`

Whether the page visibly shows supply, supply login, goods source, source station, substation, substation backend, agent backend, or similar supply/distribution entries.

### Rule Direction

Example rule patterns:

```text
page_type = rebate
AND risk_behavior.task_commission_rebate = true
AND visual_signals.task_or_order_rebate_visible = true
=> final_advice = finance
```

```text
page_type = rebate
AND risk_behavior.advance_payment_or_deposit = true
AND visual_signals.advance_payment_or_recharge_visible = true
=> final_advice = finance
```

```text
page_type = rebate
AND visual_signals.commission_or_cashout_visible = true
AND (
  visual_signals.task_or_order_rebate_visible = true
  OR visual_signals.advance_payment_or_recharge_visible = true
)
=> final_advice = finance
```

```text
page_type = rebate
AND (
  visual_signals.agent_recruitment_visible = true
  OR visual_signals.supply_or_substation_entry = true
)
=> strengthen finance risk
```

### Open Issue

`agent_recruitment` is not included in `risk_behavior` for the first version because it overlaps with `visual_signals.agent_recruitment_visible`. Confirm with operations later whether agent recruitment should become an independent risk behavior.

## Page Type: login

### Final Field Structure

```json
{
  "page_type": "login",
  "risk_behavior": {
    "restricted_access_registration": true,
    "customer_service_assisted_login": true
  },
  "visual_signals": {
    "customer_service_entry": true,
    "invitation_code_required": true,
    "guest_access_entry": true,
    "public_registration_entry": false
  }
}
```

### Field Meaning

#### page_type

`login`

The page is a login, registration, authentication, authorization, account joining, or access verification page.

#### risk_behavior

`restricted_access_registration`

Whether the formal registration, joining, account opening, superior binding, team binding, or core system access flow requires a non-public access code, such as invitation code, team code, referral code, superior code, agent code, internal code, organization code, or channel code.

This behavior can still be true when the page provides guest access, because guest access only means temporary browsing or trial access. It does not mean the user can publicly create an account or formally join the system.

`customer_service_assisted_login`

Whether the login, registration, joining, or authentication area prominently provides a customer service entry, making customer service an assisted or parallel path for account access.

This is suspicious but should be treated carefully. A customer service entry alone does not prove fraud.

#### visual_signals

`customer_service_entry`

Whether the page visibly shows a customer service or support entry, such as customer service, online customer service, contact customer service, support chat, consultation, or advisor entry.

`invitation_code_required`

Whether the page visibly requires or displays a non-public access code for registration, joining, or system access. This includes invitation code, team code, referral code, superior code, agent code, internal code, organization code, or channel code.

Team code is treated as a variant of invitation code.

`guest_access_entry` **[TBD]**

Whether the page visibly provides a guest access, skip login, trial, preview, or temporary no-login entry.

Guest access is not the same as public registration.

`public_registration_entry` **[TBD]**

Whether the page visibly provides a public registration entry that allows the user to create a formal account or formally join the system without an invitation code, team code, referral code, superior code, agent code, or similar non-public access code.

Guest access does not count as public registration.

### Rule Direction

The rule engine should keep visual facts and risk behavior separate.

Example rule patterns:

```text
page_type = login
AND risk_behavior.restricted_access_registration = true
AND visual_signals.invitation_code_required = true
AND visual_signals.public_registration_entry = false
=> suspicious / need_preview
```

```text
page_type = login
AND risk_behavior.customer_service_assisted_login = true
AND visual_signals.customer_service_entry = true
=> suspicious / need_preview
```

```text
page_type = login
AND visual_signals.guest_access_entry = true
AND risk_behavior.restricted_access_registration = true
=> keep restricted_access_registration true
```

## Page Type: crypto

### Final Field Structure

```json
{
  "page_type": "crypto",
  "risk_behavior": {
    "virtual_currency_trading": true
  },
  "visual_signals": {
    "crypto_asset_visible": true,
    "primary_language_chinese": true,
    "language_switch_entry": true
  }
}
```

### Field Meaning

#### page_type

`crypto`

The page is a virtual currency or digital asset page, including Bitcoin, BTC, USDT, ETH, crypto exchanges, wallets, token prices, buy/sell crypto, contract trading, crypto recharge, or crypto withdrawal.

#### risk_behavior

`virtual_currency_trading`

Whether the page shows or guides virtual currency trading or asset operations, such as buying, selling, exchange, trading, recharge, withdrawal, wallet transfer, asset management, contract trading, or token market trading.

#### visual_signals

`crypto_asset_visible`

Whether the page visibly shows virtual currency names, symbols, products, or trading elements, such as Bitcoin, BTC, USDT, ETH, TRX, BNB, crypto, token, wallet, exchange, buy crypto, sell crypto, recharge crypto, withdraw crypto, contract, trading pair, or K-line chart.

`primary_language_chinese`

Whether the main visible page language is Chinese.

This is only a visual language signal. It does not determine whether the site is a China site.

`language_switch_entry`

Whether the page visibly provides a language switch entry, such as a language dropdown, Language, Chinese/English toggle, globe icon, country flag icon, or multilingual menu.

### Rule Direction

The rule engine should use platform context for China-site judgment. Whether the page belongs to a China site is not a VLM visual field.

Example rule patterns:

```text
page_type = crypto
AND risk_behavior.virtual_currency_trading = true
AND platform_context.is_china_site = true
=> finance
```

```text
page_type = crypto
AND risk_behavior.virtual_currency_trading = true
AND visual_signals.primary_language_chinese = true
=> finance
```

```text
page_type = crypto
AND risk_behavior.virtual_currency_trading = true
AND visual_signals.language_switch_entry = true
=> finance
```

```text
page_type = crypto
AND risk_behavior.virtual_currency_trading = true
AND platform_context.is_china_site = false
AND visual_signals.primary_language_chinese = false
AND visual_signals.language_switch_entry = false
=> good
```

## Page Type: digital_goods

### Final Field Structure

```json
{
  "page_type": "digital_goods",
  "risk_behavior": {
    "virtual_goods_trading": true,
    "agent_recruitment": true
  },
  "visual_signals": {
    "virtual_goods_or_account_visible": true,
    "agent_recruitment_visible": true,
    "customer_service_entry": true,
    "supply_or_substation_entry": true,
    "order_query_entry": true,
    "vpn_or_proxy_service_visible": true
  }
}
```

### Field Meaning

#### page_type

`digital_goods`

The page is a virtual goods, account, code, key, membership, digital service, supply, reseller, or distribution platform page.

This page type covers what operations may call "card/key/code" scams. It does not mean bank card fraud.

#### risk_behavior

`virtual_goods_trading`

Whether the page shows or guides trading of virtual goods, accounts, card/key/code goods, redemption codes, license keys, memberships, digital services, VPN accounts, Telegram accounts, overseas platform accounts, or similar non-physical goods.

`agent_recruitment`

Whether the page shows agent recruitment, substation joining, reseller recruitment, supply agent recruitment, promotion earning, daily income claims, referral distribution, or similar agent/distribution behavior.

#### visual_signals

`virtual_goods_or_account_visible`

Whether the page visibly shows virtual goods, accounts, card/key/code goods, redemption codes, license keys, memberships, digital services, VPN accounts, Telegram accounts, overseas platform accounts, auto-delivery, supply, goods source, or related keywords/products.

`agent_recruitment_visible`

Whether the page visibly shows agent recruitment, join agent, substation, franchise, promotion earning, daily income, referral distribution, or similar agent/distribution information.

`customer_service_entry`

Whether the page visibly shows a customer service or support entry, such as manual customer service, online customer service, contact customer service, support chat, or consultation.

`supply_or_substation_entry`

Whether the page visibly shows supply, supply login, goods source, source station, substation, substation backend, agent backend, or similar supply/distribution entries.

`order_query_entry`

Whether the page visibly shows order, place order, order query, query, tutorial, after-sales query, or similar transaction support entries.

`vpn_or_proxy_service_visible`

Whether the page visibly shows VPN, proxy, node, circumvention, accelerator, airport, Proxy, V2Ray, Clash, Shadowrocket, or similar VPN/proxy goods or services.

This signal does not change `page_type` to `vpn`. It only allows the rule engine to assign `final_advice = vpn` when VPN/proxy goods or services are visible inside another page type.

### Rule Direction

The default advice for digital goods/account/code marketplace risk is `other_fraud`, unless a higher-priority category is visible.

Example rule patterns:

```text
page_type = digital_goods
AND risk_behavior.virtual_goods_trading = true
AND visual_signals.virtual_goods_or_account_visible = true
=> final_advice = other_fraud
```

```text
page_type = digital_goods
AND risk_behavior.agent_recruitment = true
AND visual_signals.agent_recruitment_visible = true
=> final_advice = other_fraud
```

```text
page_type = digital_goods
AND visual_signals.supply_or_substation_entry = true
AND visual_signals.order_query_entry = true
=> final_advice = other_fraud
```

```text
page_type = digital_goods
AND visual_signals.vpn_or_proxy_service_visible = true
=> final_advice = vpn
```

VPN remains an independent page type and advice category. If the page subject is VPN, proxy, circumvention tools, nodes, airports, Clash/V2Ray configuration, or VPN subscription services, use `page_type = vpn`. If the page subject is a virtual goods marketplace that happens to sell VPN/proxy goods, keep `page_type = digital_goods` and let the rule engine output `final_advice = vpn`.

## Page Type: vpn

### Final Field Structure

```json
{
  "page_type": "vpn",
  "risk_behavior": {
    "vpn_proxy_usage": true,
    "vpn_proxy_service_provision": true
  },
  "visual_signals": {
    "vpn_proxy_usage_state_visible": true,
    "vpn_proxy_service_provider_visible": true
  }
}
```

### Field Meaning

#### page_type

`vpn`

The page subject is VPN, proxy, circumvention tools, nodes, airports, accelerators, Proxy, V2Ray, Clash, Shadowrocket, VPN/proxy configuration, VPN/proxy usage, or VPN/proxy service provision.

#### risk_behavior

`vpn_proxy_usage`

Whether the page shows or guides use, enabling, switching, or configuration of VPN/proxy/circumvention tools, such as node switching, global proxy, system proxy, routing rules, latency testing, real-time speed, or connection status.

`vpn_proxy_service_provision`

Whether the page shows or guides provision, sale, subscription, management, recharge, purchase, renewal, support, or tutorial of VPN/proxy/circumvention services, such as node services, airport services, traffic packages, proxy accounts, subscription records, purchase records, usage tutorials, or ticket support.

#### visual_signals

`vpn_proxy_usage_state_visible`

Whether the page visibly shows VPN/proxy/circumvention tools in use or configuration state, such as outbound mode, global proxy, system proxy, enhanced mode, real-time speed, node switching, node list, region nodes, DIRECT/Proxy status, latency testing, routing rules, or Telegram/Youtube/Netflix routing items.

`vpn_proxy_service_provider_visible`

Whether the page visibly shows VPN/proxy/circumvention service provider, seller, or management evidence, such as purchase records, subscription records, recharge, traffic records, node list, relay rules, usage tutorials, ticket system, website announcement, plans, traffic quota, price, renewal, airport service, or proxy service.

### Rule Direction

Example rule pattern:

```text
page_type = vpn
AND (
  risk_behavior.vpn_proxy_usage = true
  OR risk_behavior.vpn_proxy_service_provision = true
)
AND (
  visual_signals.vpn_proxy_usage_state_visible = true
  OR visual_signals.vpn_proxy_service_provider_visible = true
)
=> final_advice = vpn
```

## Page Type: porn

### Final Field Structure

```json
{
  "page_type": "porn",
  "risk_behavior": {
    "pornographic_content": true
  },
  "visual_signals": {
    "explicit_nudity_visible": true,
    "sexual_service_text_visible": true,
    "sexualized_but_non_explicit_visible": false,
    "sex_toy_visible": false,
    "artistic_nudity_visible": false,
    "medical_or_educational_sexual_content_visible": false
  }
}
```

### Field Meaning

#### page_type

`porn`

The page subject is pornographic content, adult sexual content, nude chat, hookup, adult live streaming, adult videos, adult images, or adult services.

If another page type contains pornographic content, keep the original page type when it better describes the page subject, and let the rule engine output `final_advice = porn`.

#### risk_behavior

`pornographic_content`

Whether the page shows or guides pornographic content, adult sexual services, nude chat, hookup, adult live streaming, adult videos, explicit sexual text, incest content, or similar pornographic behavior/services.

#### visual_signals

`explicit_nudity_visible`

Whether the page visibly shows explicit pornographic nudity or sexual acts, such as visible genitals, visible female nipples/areola, three-point nudity, intercourse, masturbation, or other explicit sexual behavior.

`sexual_service_text_visible`

Whether the page visibly shows sexual service or pornographic inducement text, such as hookup, nude chat, porn live streaming, adult videos, same-city hookup, incest, mature woman, door-to-door service, or erotic video.

`sexualized_but_non_explicit_visible`

Whether the page visibly shows sexy, suggestive, swimsuit, underwear, body display, or soft sexualized content without explicit nudity, explicit sexual acts, or sexual service text.

`sex_toy_visible`

Whether the page visibly shows sex toys or adult products, such as vibrators, adult toys, condoms, lubricants, or similar adult goods.

This signal alone does not mean pornographic violation.

`artistic_nudity_visible`

Whether the page visibly shows artistic nudity, such as sculpture, painting, museum artwork, or body art.

This signal alone should be treated as a protective factor when there is no sexual service text.

`medical_or_educational_sexual_content_visible`

Whether the page visibly shows sexual organs or reproductive content in a medical, educational, scientific, health, or anatomy context, such as uterus diagrams, reproductive anatomy, medical textbooks, or health education.

This signal alone should be treated as a protective factor when there is no sexual service text.

### Rule Direction

Example rule patterns:

```text
risk_behavior.pornographic_content = true
AND (
  visual_signals.explicit_nudity_visible = true
  OR visual_signals.sexual_service_text_visible = true
)
=> final_advice = porn
```

```text
visual_signals.artistic_nudity_visible = true
AND visual_signals.sexual_service_text_visible = false
=> not porn
```

```text
visual_signals.medical_or_educational_sexual_content_visible = true
AND visual_signals.sexual_service_text_visible = false
=> not porn
```

```text
visual_signals.sex_toy_visible = true
AND visual_signals.explicit_nudity_visible = false
AND visual_signals.sexual_service_text_visible = false
=> not porn
```

```text
visual_signals.sexualized_but_non_explicit_visible = true
AND visual_signals.explicit_nudity_visible = false
AND visual_signals.sexual_service_text_visible = false
=> not porn
```

## Page Type: gambling

### Final Field Structure

```json
{
  "page_type": "gambling",
  "risk_behavior": {
    "gambling_or_betting": true,
    "lottery_activity": true
  },
  "visual_signals": {
    "gambling_entity_visible": true,
    "gambling_slang_visible": true,
    "gambling_icon_visible": true,
    "gambling_entry_visible": true,
    "lottery_text_visible": true,
    "prize_draw_visible": true,
    "chess_card_game_visible": true,
    "fishing_game_visible": true,
    "electronic_game_visible": true,
    "sports_content_visible": true,
    "odds_or_handicap_visible": true,
    "cash_recharge_or_withdraw_visible": true,
    "casino_or_live_dealer_visible": true,
    "sports_betting_visible": true,
    "lottery_or_draw_visible": true
  }
}
```

### Field Meaning

#### page_type

`gambling`

The page subject is gambling, betting, lottery, draw, chess/card games, fishing games, electronic gaming, live dealer casino, sports, sports betting, casino, odds, or betting platform.

`page_type = gambling` only describes the visible page scenario. It does not mean the final decision must be `gamble`. The rule engine should make the final decision from `risk_behavior + visual_signals + platform_context`.

#### risk_behavior

`gambling_or_betting`

Whether the page shows or guides an obvious gambling economic system, such as betting, wagering, odds, casino bonus, gambling rebate, wagering turnover, first-deposit bonus, guaranteed compensation, live dealer casino, casino games, sports betting entry, or similar betting activity.

This field is for clearly gambling-oriented behavior. It can be triggered by obvious betting text, gambling entities, casino/live dealer content, betting entries, odds/handicap, or other visible gambling economic signals. It should not be triggered by ordinary sports content, ordinary games, chess/card games, fishing games, electronic games, or prize draw visuals alone.

`lottery_activity`

Whether the page shows lottery behavior, such as lottery, Mark Six, lottery platform, lottery purchase, lottery numbers, draw results, number betting, or online lottery sales.

Lottery is a hard gambling behavior. Ordinary marketing prize draws, roulette-style promotions, golden egg promotions, scratch cards, or lucky draws are not lottery unless they show lottery sales, number betting, gambling return, cashout, odds, betting, or similar gambling economic signals.

#### visual_signals

`gambling_entity_visible`

Whether the page visibly shows gambling entities, scenes, brands, or platform names, such as gambling, betting, casino, entertainment city, Macau, Grand Lisboa, Galaxy Entertainment, Macau Sands, Wynn Macau, MGM, Suncity Group, Crown Sports, Kaiyuan, live dealer, sports betting, electronic gaming, lottery platform, or similar gambling platform identity.

`gambling_slang_visible`

Whether the page visibly shows gambling text, slang, or betting terms, such as casino bonus, rebate, wagering turnover, first-deposit bonus, guaranteed compensation, bet, wager, odds, handicap, banker/player, points up/down, rolling commission, prize return, line check, line entry, or gambling promotional slogans.

Chinese examples include: 六合彩、线上博彩、线上赌场、彩金、返水、反水、流水、首存赠送、注册送28元、包赔、投注、下注、赔率、盘口、水位、庄闲、上分、下分、洗码、返奖、真人视讯、线路检测、线路入口、娱乐是一种态度.

`gambling_icon_visible`

Whether the page visibly shows gambling-oriented icons or visual symbols, such as casino chips, playing-card betting icons, dice, roulette, slot machine icons, dealer avatars, Macau casino icons, betting ticket icons, lottery-ball icons, or sportsbook betting icons.

`gambling_entry_visible`

Whether the page visibly shows a gambling entry, such as enter casino, enter lobby, bet now, start betting, sports betting entry, live dealer entry, lottery entry, line entry, line check, register to receive casino bonus, first-deposit bonus entry, or similar call-to-action.

`lottery_text_visible`

Whether the page visibly shows lottery-specific text or numbers, such as lottery, Mark Six, lottery platform, lottery purchase, lottery numbers, draw results, number betting, or online lottery sales.

`prize_draw_visible`

Whether the page visibly shows ordinary prize draw visuals or text, such as roulette draw, golden egg draw, scratch card, lucky draw, spin wheel, lottery-style marketing promotion, or probability reward.

`chess_card_game_visible`

Whether the page visibly shows chess/card game elements or text, such as chess/card games, mahjong, poker, Dou Dizhu, Texas Hold'em, Niu Niu, baccarat, Golden Flower, or similar card/table games.

This is a visual fact only. It should be combined with gambling text, betting entry, cash recharge/withdrawal, or other gambling evidence before deciding `final_advice = gamble`.

`sports_content_visible`

Whether the page visibly shows sports content, such as football, basketball, World Cup, sports teams, match schedules, scoreboards, sports news, match banners, or player/team visuals.

This is a visual fact only. It should be combined with odds, handicap, betting entry, or gambling text before deciding `final_advice = gamble`.

`fishing_game_visible`

Whether the page visibly shows fishing game visuals or text.

This is a visual fact only. It should be combined with gambling text, betting entry, cash recharge/withdrawal, or other gambling evidence before deciding `final_advice = gamble`.

`electronic_game_visible`

Whether the page visibly shows electronic game hall, slot-machine-style game, arcade game, or electronic entertainment visuals/text.

This is a visual fact only. It should be combined with gambling text, betting entry, cash recharge/withdrawal, or other gambling evidence before deciding `final_advice = gamble`.

`odds_or_handicap_visible`

Whether the page visibly shows odds, handicap, water level, betting lines, match odds, point spread, over/under, parlay, or similar sports/betting odds information.

`cash_recharge_or_withdraw_visible`

Whether the page visibly shows recharge, deposit, withdrawal, cashout, wallet, balance, cash exchange, points up/down, buy chips, or other money-in/money-out entries.

`casino_or_live_dealer_visible`

Whether the page visibly shows casino or live dealer elements, such as casino, dealer, live dealer, baccarat table, roulette, slot machine, Macau casino, or similar casino scenes.

`sports_betting_visible`

Whether the page visibly shows sports betting, sports wager, odds board, match betting, football/basketball odds, handicap, or odds lines.

`lottery_or_draw_visible`

Whether the page visibly shows lottery or draw evidence.

Lottery examples include: lottery, Mark Six, lottery platform, lottery purchase, lottery numbers, draw results, number betting, online lottery sales.

Prize draw examples include: roulette draw, golden egg draw, scratch card, lucky draw, prize return, or similar probability-based draw/promotion activity. These should be treated as `visual_signals.prize_draw_visible`, not `risk_behavior.lottery_activity`, unless lottery sales, number betting, gambling return, cashout, odds, betting, or similar gambling economic signals are visible.

### Rule Direction

Gambling hard signals override finance and fake. If hard gambling behavior and evidence are visible, the rule engine should output `final_advice = gamble` even when the page also shows cash reward, rebate, investment, financial return, official branding, or impersonation.

VLM should extract visible facts into `risk_behavior` and `visual_signals`; Rule Engine is responsible for the final `final_advice` and `final_action`.

Example rule patterns:

```text
page_type = gambling
AND risk_behavior.gambling_or_betting = true
=> final_advice = gamble
```

```text
risk_behavior.lottery_activity = true
AND visual_signals.lottery_text_visible = true
=> final_advice = gamble
```

```text
visual_signals.chess_card_game_visible = true
AND (
  visual_signals.cash_recharge_or_withdraw_visible = true
  OR visual_signals.gambling_slang_visible = true
  OR visual_signals.gambling_entry_visible = true
  OR visual_signals.odds_or_handicap_visible = true
)
=> final_advice = gamble
```

```text
visual_signals.sports_content_visible = true
AND (
  visual_signals.sports_betting_visible = true
  OR visual_signals.odds_or_handicap_visible = true
  OR visual_signals.gambling_slang_visible = true
  OR visual_signals.gambling_entry_visible = true
)
=> final_advice = gamble
```

```text
visual_signals.fishing_game_visible = true
AND (
  visual_signals.cash_recharge_or_withdraw_visible = true
  OR visual_signals.gambling_slang_visible = true
  OR visual_signals.gambling_entry_visible = true
)
=> final_advice = gamble
```

```text
visual_signals.electronic_game_visible = true
AND (
  visual_signals.cash_recharge_or_withdraw_visible = true
  OR visual_signals.gambling_slang_visible = true
  OR visual_signals.gambling_entry_visible = true
)
=> final_advice = gamble
```

```text
visual_signals.gambling_entity_visible = true
OR visual_signals.gambling_icon_visible = true
OR visual_signals.gambling_entry_visible = true
OR visual_signals.casino_or_live_dealer_visible = true
=> final_advice = gamble
```

```text
visual_signals.prize_draw_visible = true
AND (
  visual_signals.cash_recharge_or_withdraw_visible = true
  OR visual_signals.gambling_slang_visible = true
  OR visual_signals.gambling_entry_visible = true
  OR visual_signals.odds_or_handicap_visible = true
)
=> final_advice = gamble or need_preview, depending on evidence strength
```

```text
(
  visual_signals.chess_card_game_visible = true
  OR visual_signals.sports_content_visible = true
  OR visual_signals.fishing_game_visible = true
  OR visual_signals.electronic_game_visible = true
  OR visual_signals.prize_draw_visible = true
)
AND no gambling text, no betting entry, no odds/handicap, no cash recharge/withdrawal
=> not hard gamble
```

```text
hard gambling evidence
AND visual_signals.primary_language_chinese = false
AND visual_signals.language_switch_entry = false
AND platform_context.site_region is Brazil, South Africa, or another regulated gambling market
=> final_action = need_preview
```

## Page Type: game

### Draft Field Structure

This is a transition draft for VLM testing.

```json
{
  "page_type": "game",
  "risk_behavior": {
    "game_play_or_promotion": true,
    "private_server_or_cheat": true
  },
  "visual_signals": {
    "game_content_visible": true,
    "private_server_text_visible": true,
    "cheat_tool_visible": true,
    "game_account_or_item_trade_visible": true
  }
}
```

### Field Meaning

#### page_type

`game`

The page subject is ordinary games, mobile games, web games, game promotion, game download, game play, game private servers, game cheats, game plugins, game acceleration tools, or similar game-related content.

If the page clearly shows gambling economic systems, betting, casino, lottery, odds, recharge/withdrawal cashout, points up/down, or gambling slang, use `page_type = gambling` or keep the visible gambling signals for Rule Engine escalation.

#### risk_behavior

`game_play_or_promotion`

Whether the page shows or guides ordinary game play, game download, game promotion, game registration, game lobby, game task, or game participation.

`private_server_or_cheat`

Whether the page shows or guides private server, unauthorized server, cheat, plugin, hack, modifier, script, accelerator abuse, or similar game-violation behavior.

#### visual_signals

`game_content_visible`

Whether the page visibly shows game title, game character, game UI, game lobby, gameplay screenshot, download button, start game button, or game promotion content.

`private_server_text_visible`

Whether the page visibly shows private server, 私服, 变态服, GM服, 公益服, 怀旧服, unauthorized server, or similar private-server text.

`cheat_tool_visible`

Whether the page visibly shows cheat, plugin, hack, modifier, script, 辅助, 外挂, 自动脚本, 透视, 加速, or similar cheat/tool text or UI.

`game_account_or_item_trade_visible`

Whether the page visibly shows game account, item, equipment, currency, recharge card,代练,陪玩, or similar game goods/service trading. This signal alone does not determine final advice; Rule Engine should decide whether it belongs to `digital_goods`, `game`, or other risk.

### Rule Direction

```text
page_type = game
AND visual_signals.private_server_text_visible = true
=> final_advice = game
```

```text
page_type = game
AND visual_signals.cheat_tool_visible = true
=> final_advice = game
```

```text
page_type = game
AND visual_signals.game_content_visible = true
AND no private server, no cheat, no gambling, no cashout/betting evidence
=> not hard violation
```

## Page Type: politic

### Draft Field Structure

This is a transition draft for VLM testing.

```json
{
  "page_type": "politic",
  "risk_behavior": {
    "political_sensitive_content": true,
    "terrorism_or_separatism_content": true
  },
  "visual_signals": {
    "china_political_sensitive_visible": true,
    "military_or_police_sensitive_visible": true,
    "terrorism_or_extremism_visible": true,
    "separatism_symbol_visible": true
  }
}
```

### Field Meaning

#### page_type

`politic`

The page subject is China political sensitive content, military/police sensitive content, international terrorism, extremism, separatism, or related political risk content.

#### risk_behavior

`political_sensitive_content`

Whether the page shows or promotes political sensitive content, especially China-related political sensitive content, leaders, party/state organs, slogans, protests, petitions, political attacks, military/police sensitive content, or related content.

`terrorism_or_separatism_content`

Whether the page shows or promotes terrorism, extremism, violent organization, separatism, national split, terrorist symbols, extremist propaganda, or related content.

#### visual_signals

`china_political_sensitive_visible`

Whether the page visibly shows China political sensitive names, slogans, leaders, party/state organs, protest/petition content, political attack text, or related visuals.

`military_or_police_sensitive_visible`

Whether the page visibly shows military, police, weapons, uniforms, official insignia, troops, armed action, military/police sensitive text, or related visuals in a sensitive political context.

`terrorism_or_extremism_visible`

Whether the page visibly shows terrorism, extremist organization, violent extremist propaganda, terrorist flags, weapons with terror context, execution/attack propaganda, or related content.

`separatism_symbol_visible`

Whether the page visibly shows separatist slogans, flags, maps, symbols, independence claims, national split content, or related visuals.

### Rule Direction

```text
page_type = politic
AND (
  risk_behavior.political_sensitive_content = true
  OR risk_behavior.terrorism_or_separatism_content = true
)
=> final_advice = politic
```

## Page Type: general

### Draft Field Structure

This is a transition draft for VLM testing. `general` is the default container for safe or ordinary pages outside current risk scenarios.

```json
{
  "page_type": "general",
  "risk_behavior": {
    "ordinary_information_or_service": true
  },
  "visual_signals": {
    "ordinary_content_visible": true,
    "ordinary_game_or_sports_visible": true,
    "ordinary_brand_or_corporate_visible": true,
    "education_news_tool_visible": true
  }
}
```

### Field Meaning

#### page_type

`general`

The page subject is ordinary, safe, or currently unsupported non-risk content, such as news, articles, education, tools, corporate websites, brand display, normal product/service introduction without transaction risk, normal social content, normal sports information, or ordinary game entertainment without private server/cheat/gambling evidence.

Use `general` when the page does not fit the existing risk page types and there is no clear suspicious or violating scenario. This page type is mainly for `good/pass` fallback after Rule Engine checks.

#### risk_behavior

`ordinary_information_or_service`

Whether the page shows ordinary information browsing, normal brand/service introduction, education, news, tool usage, entertainment browsing, or similar non-risk activity.

#### visual_signals

`ordinary_content_visible`

Whether the page visibly shows ordinary content such as article text, news content, navigation, profile, informational page, gallery, community post, or normal website layout.

`ordinary_game_or_sports_visible`

Whether the page visibly shows ordinary game entertainment or sports information without betting, odds, gambling slang, recharge/withdrawal cashout, private server, or cheat evidence.

`ordinary_brand_or_corporate_visible`

Whether the page visibly shows normal brand, company, product, official-looking corporate site, business introduction, contact/about page, or similar non-risk brand/corporate content.

`education_news_tool_visible`

Whether the page visibly shows education, news, learning, documentation, calculator, editor, utility, productivity tool, or similar ordinary tool/content service.

### Rule Direction

```text
page_type = general
AND no high-risk cross-page visual_signals
=> final_advice = good
```

```text
page_type = general
AND gambling/porn/politic/game/vpn/finance/fake strong signals are visible
=> Rule Engine should use those signals to correct final_advice or send need_preview
```

For speed, `general` should not require full extraction of every risk category. In the transition prompt, it should only focus on several false-positive-prone areas:

- ordinary game/sports vs gambling
- ordinary brand/corporate page vs fake
- ordinary game page vs private server/cheat
- ordinary information/tool/news/education page vs unsupported risk

## Page Type: other

### Draft Field Structure

This is a transition draft for VLM testing.

```json
{
  "page_type": "other",
  "risk_behavior": {
    "unclassified_suspicious_activity": true
  },
  "visual_signals": {
    "unclassified_risk_visible": true,
    "abnormal_contact_or_redirect_visible": true,
    "suspicious_service_or_claim_visible": true
  }
}
```

### Field Meaning

#### page_type

`other`

The page appears suspicious, abnormal, violating, or risky, but cannot be clearly classified as mall, payment, rebate, login, investment, crypto, digital_goods, vpn, porn, gambling, game, politic, or general.

Use `other` for risk fallback, not for ordinary safe pages. Safe unsupported pages should use `general`.

#### risk_behavior

`unclassified_suspicious_activity`

Whether the page shows suspicious or abnormal behavior that does not fit existing page types, such as unclear risk service, abnormal solicitation, suspicious recruitment, unexplained high reward, unusual contact guidance, or other unclassified risk activity.

#### visual_signals

`unclassified_risk_visible`

Whether the page visibly shows abnormal, suspicious, or risk-looking content that cannot be assigned to an existing field.

`abnormal_contact_or_redirect_visible`

Whether the page visibly emphasizes unusual contact, QR code, external app redirect, private chat, group join, or off-platform communication in a suspicious context.

`suspicious_service_or_claim_visible`

Whether the page visibly shows suspicious service claims, abnormal guarantees, black/grey market wording, exaggerated promise, or hard-to-classify risk service.

### Rule Direction

```text
page_type = other
AND risk_behavior.unclassified_suspicious_activity = true
=> final_advice = other_fraud or need_preview, depending on evidence strength
```

```text
page_type = other
AND evidence is weak or unclear
=> final_action = need_preview
```
