# Capability: Mainland China Operation

> Shipping to users inside mainland China: the filings that gate your DNS before a single request is served, the identity and payment rails that are not the western ones, and the network assumptions that stop being true at the border.

Last verified: 2026-08-10

## When a project needs this

- The brief names mainland users, a mainland company, or writes 国内 / 大陆 / 境内 / 中国区.
- Hosting is Alibaba Cloud, Tencent Cloud or Huawei Cloud, or the brief says "备案".
- A WeChat Mini Program (微信小程序), Official Account (公众号), WeChat Pay (微信支付) or Alipay (支付宝) appears anywhere.
- Distribution is Huawei / Xiaomi / OPPO / vivo / Tencent MyApp rather than Google Play.
- The interview is being conducted in Chinese **and** the users are not overseas Chinese speakers — ask, do not assume. A Chinese-speaking founder building for a US market needs none of this.

Skip it entirely when the audience is outside the border. Hosting a Chinese-language site in Hong Kong or Singapore is legal, needs no filing, and costs you latency and an occasional unreachable window — a real tradeoff, not a violation.

**Read this capability before the stack table, not after.** It rewrites four rows — hosting, auth, payments, model provider — and it introduces gates measured in calendar weeks that no amount of engineering compresses.

## Decision zero: is the deployment inside the border?

Everything else follows from this one answer. Decide it in DEEP DIVE, in one question.

| | Inside mainland | Outside (HK / SG) | Split: static inside, API outside |
|---|---|---|---|
| Filing | ICP filing mandatory before the host serves the domain | None | Still mandatory for the filed half |
| Latency to mainland users | Good | 80–300 ms, variable, occasionally much worse | Worst of both — the page paints, the data does not |
| WeChat Pay / Alipay | Standard domestic merchant | Cross-border merchant, FX settlement, more paperwork | Domestic merchant, cross-border API calls |
| Domestic CDN, SMS, push | Available | Mostly unavailable or degraded | Partial |
| Time to first deploy | Weeks — the filing is the critical path | Hours | Weeks |

**Recommendation: pick one side and commit.** Inside the border for consumer products aimed at mainland users — pay the filing calendar once and everything downstream is normal. Outside for B2B tools whose buyers are multinationals with VPN-equipped staff. The split architecture is the trap: it carries the full filing burden *and* the full cross-border latency, and teams choose it because it feels like a compromise rather than because it solves anything.

## The compliance gates are calendar time, not engineering time

| Gate | Who needs it | Blocks | Typical elapsed |
|---|---|---|---|
| Domain real-name (域名实名认证) | Any domain to be filed | The filing itself | Days |
| **ICP filing (ICP备案, MIIT)** | Any site served from mainland servers | The host answering for your domain at all | Weeks; varies by province and by hosting provider |
| Commercial ICP licence (经营性ICP许可证) | You charge mainland users for online services or content | Charging, not launching | Months |
| Public-security filing (公安网安备案) | Sites serving mainland users | Follows ICP filing | Days |
| Graded protection (等保2.0 定级备案) | Public internet platforms — commonly Level 3; Level 2 for smaller systems | Sector regulators, enterprise and government procurement | 3–6 months end to end. Level 3 is re-tested annually, Level 2 every two years |
| GenAI filing (生成式AI备案 / 登记) | See below | Public launch of the AI feature | Weeks to months |
| Cross-border transfer (数据出境) | Any personal information leaving the border | The first export | Depends on mechanism — see below |

**None of these belong in the build order.** Every one of them completes when an outside party acts, which is exactly the acceptance criterion `agents/blueprint-validator.md` finding 17 forbids as a build gate: an autonomous builder either stalls forever or self-certifies. They go in the blueprint's §20.1 **manual gates / post-build launch checklist**, named, owned and dated. Writing "Done when: 备案通过" as step 3 of 14 is the single most damaging mistake in this whole capability.

### Filing an AI feature: 备案 versus 登记

Two different filings, and briefs routinely name the wrong one. Under the Interim Measures for Generative AI Services (生成式人工智能服务管理暂行办法):

- **备案** — national CAC filing, for a service with public-opinion or social-mobilization attributes (舆论属性或者社会动员能力) that fields its **own** model.
- **登记** — local CAC registration, for an application that merely calls an **already-filed** model through its API. This is what most product teams actually need, and it is the lighter path.

Either way the operator must display, prominently, which filed service is in use — the model name and its filing number. That is a real, scriptable acceptance criterion; see the build steps below.

### Cross-border personal information: the thresholds that decide the mechanism

Counted cumulatively from 1 January each year, for a processor that is **not** a critical information infrastructure operator:

| Volume of personal information sent abroad | Mechanism |
|---|---|
| Fewer than 100,000 people, no sensitive PI | None — exempt from declaration |
| 100,000 to under 1,000,000 people, or fewer than 10,000 people's **sensitive** PI | Standard contract filing (标准合同备案) or certification |
| 1,000,000+ people, or 10,000+ people's sensitive PI, or any important data (重要数据) | CAC security assessment (安全评估) |

The design consequence, and it is the only one that belongs in §9: **PI leaves the border in your architecture whether or not anyone decided it would.** An error tracker, an analytics pixel, a session-replay tool, a foreign LLM endpoint, a support-desk integration, a CI job that restores a production dump — each is an export. Enumerate them in §7 and pick, per sink, either a mainland region or a field-level redaction that means no PI crosses. National ID numbers (身份证号) are sensitive PI, and one column of them drops your threshold by two orders of magnitude.

## Identity: phone first, email never

| Method | Use for | What it actually costs |
|---|---|---|
| **Phone + SMS code (手机号+短信验证码)** | The default for consumer products | Provider onboarding: a real-name entity, an approved signature (签名) and message template (模板), reviewed over days |
| WeChat authorization (微信授权登录) | Anything inside a Mini Program, Official Account or the WeChat browser | Nothing extra if the app is already there — it is one tap and it converts far better than a form |
| Carrier one-tap (一键登录) | Native apps, over cellular | Per-attempt fee; falls back to SMS on Wi-Fi |
| Alipay authorization | Commerce and payment-led flows | Merchant account first |
| WeCom / DingTalk QR (企业微信/钉钉扫码) | Internal tools, B2B | The org already runs one of them, or this is worthless |
| Email + password | Nothing consumer-facing | Reads as a foreign product and converts badly. Keep it for an admin console |

**Recommendation: phone as the identity key, WeChat authorization as the fast path where the surface allows it.** `users.phone` is the natural key and the unique index; email is an optional profile field.

**The OpenID trap, which is a data-model bug and not a preference.** WeChat issues a different `openid` per surface — Mini Program, Official Account, native app are three. The stable cross-surface identifier is `unionid`, and only if the surfaces are bound to the same Open Platform account. Key the user on `unionid`, store each `openid` beside it, and a person who signs in from the Mini Program on Monday and the app on Tuesday is one row. Key on `openid` and they are two, permanently, and merging them later means merging orders.

## Payments

**Recommendation: WeChat Pay and Alipay, both, from day one.** They are not alternatives to each other in the way two card processors are — users have a strong per-context preference, and shipping one costs conversion. Everything else is a rounding error.

Both require a company: business licence, corporate bank account, and a merchant application reviewed over days to weeks. Neither has a usable personal-merchant path for a product. Budget it as an outside-party gate, exactly like the filings.

Four engineering facts that differ from a card processor, and each one is a build step:

- **`out_trade_no` is yours, and it is the idempotency key.** You mint it, it is short, and it must be unique forever. The channel's `transaction_id` arrives later and is not a substitute.
- **The async notification is not the ledger.** It can arrive twice, out of order, or not at all. Verify the signature, treat it as a hint, and confirm state by querying the channel's order API before you grant anything.
- **Reconciliation is a separate daily job** against the channel's bill file. Settlement is roughly T+1. A product that never reconciles discovers its first mismatch during an audit, not during a deploy.
- **Recurring billing is a signing flow, not a card on file.** WeChat Pay's 委托代扣 and Alipay's 周期扣款 both require the user to sign an agreement, and the agreement can be cancelled inside the wallet app without touching your product. Entitlement must follow the agreement's state, not a subscription row you own.

An aggregator buys one integration in exchange for a cut of every transaction and an extra party in the settlement path. Worth it for a small team shipping both channels plus refunds; not worth it once volume is real.

## The network stops behaving at the border

This is where a blueprint written from western defaults fails silently — every one of these resolves fine on the machine that wrote it.

**Unreachable or unreliable from inside:** anything Google (Fonts, reCAPTCHA, Maps, Analytics, YouTube embeds, **Firebase — including FCM push**), and, intermittently, default package registries, container registries, and GitHub release and raw endpoints.

**So the blueprint states mirrors as environment-conditional configuration, not as a README note.** Package registry, Python index, container registry, and JVM repository each get a mirror value selected by one environment variable. Hardcode the mirror and the project stops building in foreign CI; omit it and it stops building on the developer's machine in Shanghai. Both are real, so the switch is the deliverable.

**Push notifications:** iOS is APNs as usual. Android has no Play Services, so delivery goes through per-vendor channels — Huawei, Xiaomi, OPPO, vivo, Honor — normally via an aggregator SDK. One integration, five review processes, and a per-vendor payload quirk. Never let a blueprint reach §9 with "FCM" in it for a mainland Android app.

**Maps carry a different coordinate system, and this one is a correctness bug.** Chinese basemaps use GCJ-02; a GPS fix is WGS-84. Plot raw device coordinates on a domestic map and everything sits a few hundred metres off — consistently enough to look like a rendering bug and wrong enough to misroute a driver. Convert at exactly one boundary, store one system, and put a landmark fixture in the test suite.

**User-generated content needs moderation before publish**, not after report. Text, images and video all have hosted moderation APIs from the same cloud vendors; the compliance expectation is pre-publish, and the engineering consequence is that "create post" is an asynchronous state machine, not an insert.

## Models and the AI gateway

Direct calls to foreign model providers are not a reliable production path from inside the border — treat them as unavailable rather than slow. Domestic providers largely expose OpenAI-compatible endpoints, which is what makes this survivable: keep the single gateway interface `knowledge/capabilities/ai-llm-integration.md` already mandates, and the provider becomes one configuration value plus one adapter. A project that scattered raw SDK calls across twenty files cannot make this move at all.

Pair every public generative feature with input **and** output moderation, and with the disclosure surface the filing regime requires.

## Data model additions

| Field | Where | Why |
|---|---|---|
| `phone_e164`, unique | users | The identity key. Store normalized; display national |
| `wechat_unionid`, unique nullable | users | The cross-surface identity. Not `openid` |
| `wechat_openid_<surface>` | user_identities | One row per surface, all pointing at one user |
| `realname_verified_at`, `realname_ref` | users | Store the verification *result* and a provider reference — never the raw ID number unless the sector mandates it, because that column moves you into the sensitive-PI threshold |
| `out_trade_no`, unique | payment_orders | Yours, short, permanent. The idempotency key |
| `channel`, `transaction_id`, `settle_date` | payment_orders | Reconciliation joins on these |
| `moderation_status`, `moderation_ref` | any UGC table | Pre-publish gate; `published` is derived from it, never set directly |

## Build steps this adds

1. **Registry mirrors as conditional config.** *Done when:* WHEN `CN_MIRROR=1` is set, the install command resolves against the mirror and exits 0; WHEN it is unset, the same command resolves against the default registry and exits 0. Both paths run in CI.
2. **Phone + SMS login behind a provider interface, with a fake provider.** *Done when:* WHEN `SMS_PROVIDER=fake`, the end-to-end login test completes with no outbound network call and exits 0. The real provider's signature and template approval is an outside-party gate and belongs in §20.1 — the build may not wait on it.
3. **Identity unification on `unionid`.** *Done when:* WHEN the same `unionid` authenticates from two different surface fixtures, THE SYSTEM SHALL resolve both to one `users` row and two `user_identities` rows.
4. **Payment order lifecycle with channel-confirmed state.** *Done when:* WHEN the same async notification is delivered twice for one `out_trade_no`, THE SYSTEM SHALL respond success both times and leave the count of paid orders unchanged. WHEN a notification arrives whose signature does not verify, THE SYSTEM SHALL respond with the channel's failure form and write zero rows.
5. **Daily reconciliation against the channel bill.** *Done when:* WHEN the job runs over a fixture bill containing one settled transaction with no local order, THE SYSTEM SHALL exit non-zero and print that `out_trade_no`.
6. **Coordinate conversion at one boundary.** *Done when:* WHEN a stored WGS-84 landmark fixture is rendered for a domestic basemap, THE SYSTEM SHALL emit GCJ-02 coordinates within 20 m of the published landmark position.
7. **Pre-publish moderation gate.** *Done when:* WHEN the moderation adapter returns a block verdict, THE SYSTEM SHALL persist `moderation_status='rejected'` and the public listing query SHALL return zero rows for that item.
8. **GenAI disclosure surface.** *Done when:* WHEN the disclosure page renders, THE SYSTEM SHALL display the configured filed model name and filing number, and WHEN that configuration key is absent at boot THE SYSTEM SHALL fail startup with a named error rather than serve the page blank.
9. **Cross-border sink inventory.** *Done when:* WHEN the boundary test runs, THE SYSTEM SHALL assert that every configured third-party endpoint appears in the §7 sink table with a region or a redaction rule, and SHALL exit non-zero on the first unlisted host.

## Pitfalls

- **Putting a filing in the build order.** It is calendar time gated on a regulator. §20.1 manual gate, always. This is validator finding 17 and it is the mistake that stalls an autonomous build hardest.
- **Discovering the category needs a qualification you cannot obtain**, after the product is built. Mini Program categories (类目) and app-store listings gate on sector qualifications — medical, financial, educational. Check the category before the architecture, not before the launch.
- **Keying users on `openid`.** Duplicate accounts across surfaces, discovered when a user's orders are split across two identities.
- **Trusting the payment callback and never reconciling.** The first mismatch surfaces in an audit.
- **Modelling recurring revenue as a subscription row you own.** The user cancels the signing agreement inside the wallet and your row still says active.
- **Raw GPS on a domestic basemap.** Looks like a rendering bug, is a wrong-place bug.
- **Firebase, FCM or Google Fonts anywhere in the tree.** Each one is a blank screen or a dead notification channel for every mainland user, and each passes every test written outside the border.
- **Hardcoding the mirror.** Foreign CI stops building, and the fix lands as a revert rather than as the conditional it should have been.
- **Storing raw national ID numbers because a form collected them.** Sensitive PI, a 100× lower cross-border threshold, and a breach class of its own. Store the verification result.
- **Assuming the interview language decides this.** Ask where the users are. A Chinese-language brief for a US product needs none of this chapter, and imposing it costs weeks.

## See also

- `knowledge/capabilities/auth.md` — session-versus-token and the tenancy model this plugs a phone-first identity into
- `knowledge/capabilities/payments-rails.md` — the general webhook, idempotency and entitlement patterns these channels specialize
- `knowledge/capabilities/deployment.md` — environments, secrets and the region decision this capability constrains
- `knowledge/capabilities/ai-llm-integration.md` — the single gateway that makes swapping the model provider a config change
- `knowledge/localization.md` — writing the blueprint itself in Chinese, and the ASCII boundary it must not cross
