# Capability: Payments Rails

> Taking money reliably — choosing a billing model, deciding who owes the tax, and surviving webhooks that arrive twice, out of order, or not at all.

Last verified: 2026-07-27

## When a project needs this

- Anyone pays for anything: a plan, a one-time purchase, a credit pack, a booking deposit, a marketplace order.
- The brief says "pricing page", "free trial", "upgrade", "seats", "subscription", "checkout", or "payouts".
- Access to a feature depends on what someone bought — entitlement gating is a payments problem, not a UI problem.
- Money must reach a third party (a seller, a creator, a contractor) rather than staying with you.
- The payers are inside mainland China → read `knowledge/capabilities/china-mainland.md` instead of picking a card processor. WeChat Pay and Alipay are both required, recurring billing is a signing flow rather than a card on file, and daily reconciliation is a job you own.

Skip only when the first release is genuinely free with no upgrade path. Add the `subscription` table anyway if a paid tier appears anywhere in the brief; retrofitting entitlements is worse than carrying an unused table.

## Decision matrix

| Model | Best for | Pros | Cons |
|---|---|---|---|
| **Subscription** (per seat or flat) | SaaS, memberships, tools used weekly | Predictable revenue, provider owns the renewal machinery | Dunning, proration, trials, and cancellation are all yours to model |
| One-time purchase | Licences, courses, digital downloads, physical goods | Simplest possible state — paid or not | No recurring revenue; entitlement must still be revocable on refund |
| Usage-based (post-paid) | APIs and infra sold to companies | Aligns price with value, no top-up friction | You carry runaway-usage risk and need real invoicing plus collections |
| Prepaid credits | Anything with real marginal cost per action | Cash up front, spend is capped by construction | Needs a ledger — see `knowledge/capabilities/credit-metering.md` |
| Marketplace split | Several sellers on one platform | Platform takes a cut without touching the funds | Roughly doubles the build: onboarding, KYC, payouts, disputes |

Hybrids are normal: a plan that includes a monthly credit refill plus packs for overflow is the standard generative-product shape.

## Merchant of record vs handling your own tax

This is the decision people get wrong, and it is expensive to reverse.

A **merchant of record (MoR)** becomes the legal seller. They collect and remit VAT, GST, and sales tax worldwide, handle invoicing and refunds, and absorb fraud and chargeback administration. You get one payout and no tax registrations. You pay a higher percentage and give up control of the checkout, the customer relationship, and the data.

Running your **own payments** with a tax engine bolted on means a lower rate, full control, and every registration threshold, filing, and audit being yours.

| | Merchant of record | Own payments + tax service |
|---|---|---|
| Vendors | Paddle, Polar, Lemon Squeezy *(see caveat)* | Stripe, Adyen, Braintree — plus Stripe Tax, Anrok, Avalara, or TaxJar |
| Rate | Higher, all-in | Lower, plus separate tax-service cost |
| Tax registrations | Zero | Yours, per jurisdiction, once thresholds trip |
| Checkout control | Their hosted flow | Fully yours |
| Choose when | Digital goods, sold globally, small team, no finance function | Physical goods, US-heavy revenue, marketplace, or when checkout is a competitive surface |

Digital goods owe VAT/GST at the buyer's location **from the first sale, with no threshold**. A two-person team selling a download worldwide should not be filing in dozens of jurisdictions — take the MoR.

> **Lemon Squeezy caveat.** Stripe acquired Lemon Squeezy in 2024, and its standing as an independent MoR has been unsettled since. Do not default to it. Verify whether it is still onboarding new merchants and what the migration path looks like — run `/last30days` or check the vendor's own site — before putting it in a blueprint. Paddle and Polar are the safe MoR recommendations today.

## Recommendation

**Stripe as the default rail; a merchant of record when the product is digital, global, and the team is small.**

Stripe wins on documentation, test-mode tooling, connected accounts for marketplaces, and the sheer number of integrations that already assume it — which matters because payments code is the code you least want to write yourself. Take Adyen only when a payments team asks for it by name, or when regional acquiring is a hard requirement.

Two additional defaults:

- **Mobile apps:** digital goods and in-app subscriptions must go through the store's in-app purchase system. Physical goods and real-world services must not. Use a subscription abstraction layer over both stores rather than writing receipt validation twice — details in `knowledge/shapes/mobile-app.md`.
- **Do not build your own billing UI.** Use the provider's hosted checkout and hosted customer portal until a concrete requirement forces you off them. Every hour spent on a bespoke card form is an hour spent enlarging your PCI scope.

## The webhook reliability problem

Webhooks are the only truthful source of payment state, and they are hostile. Four failure modes, four defences.

1. **Forgery.** Verify the signature against the **raw request body** before any parsing. A framework body parser that rewrites the bytes silently breaks verification; exclude the webhook route from it explicitly. Reject on failure with `400` and log it.
2. **Duplicate delivery.** Insert the provider event id into an events table under a **unique constraint**, in the same transaction as the state change. A duplicate then fails the insert and does nothing else. Providers retry for days — duplicates are normal traffic, not an edge case.
3. **Out-of-order delivery.** `subscription.updated` can land before `subscription.created`. Never apply a payload's state blindly. Treat the webhook as "object X changed", then **fetch object X from the provider API** and write the current truth. This eliminates ordering bugs entirely and makes a stale replay harmless.
4. **Non-delivery.** Endpoints go down and deliveries get abandoned. Run a nightly reconciliation that lists provider objects changed since the last run and compares them with local state. Alert on any mismatch.

Two more rules that belong here:

- **Return `2xx` fast, then work.** Do the durable write (event row + state) inline; push email, analytics, and provisioning to a queue. A slow handler gets retried, which turns one event into five.
- **Idempotency keys on outbound calls too.** Charges, refunds, and transfers all take a client-supplied key. A retried `POST` without one is a double charge.

## The subscription state machine

Model it explicitly. Scattered `if (plan === 'pro')` checks are how entitlement bugs ship.

```
incomplete ──▶ trialing ──▶ active ──▶ past_due ──▶ canceled
                              │            │
                              ├──▶ paused ─┘
                              └──▶ canceled (at period end)
```

- **Entitlement is derived, never stored as a boolean.** One function: `entitlements(subscription) → { plan, limits, activeUntil }`. Every gate calls it. `active`, `trialing`, and `past_due` inside the grace window all grant access; everything else does not.
- **`past_due` keeps access** until dunning ends. Cutting a paying customer off during a failed-card retry is a self-inflicted churn event.
- **Cancel at period end by default.** Immediate cancellation only on explicit request, and it triggers a proration decision.
- **Upgrades apply immediately with proration; downgrades apply at period end.** Let the provider compute the proration and show its preview before the user confirms — never compute it yourself.
- **Dunning:** the provider's retry schedule plus your own email sequence. On final failure, downgrade to the free tier and **keep the data**. Deleting on non-payment guarantees the customer never comes back.
- **Refunds do not revoke entitlement.** Revoke explicitly, in the refund handler. For digital goods, revoke on dispute opened, not on dispute lost.

## Marketplace payouts

Only take this on when sellers genuinely receive money.

- **Never route seller funds through your operating account.** That makes you a money transmitter with the licensing burden that implies. Use the provider's connected-account model so funds move provider-side.
- **KYC runs in the provider's hosted flow.** A seller stays `pending` — cannot publish, cannot be paid — until the provider reports verification complete and payouts enabled.
- **Charge type matters.** Destination charges when your platform is the merchant the buyer sees; separate charges plus transfers when one cart splits across several sellers. Retrofitting the second onto the first is a rewrite.
- **Reserves and negative balances.** Chargebacks land after delivery. Hold a rolling reserve for physical goods, and decide in writing — as a column, not a policy doc — whether platform or seller absorbs a chargeback.
- **Every payout reconciles to order lines**, and sellers get a statement on day one. Payout disputes without a statement are unwinnable.

## Data model additions

| Table | Key fields | Notes |
|---|---|---|
| `customers` | `owner_id` (user or org), `provider_customer_id` | One per billing subject. Created lazily at first checkout |
| `subscriptions` | `owner_id`, `provider_subscription_id`, `status`, `plan`, `current_period_end`, `cancel_at_period_end` | **Written by webhook only.** No UI code path writes this table |
| `plans` | `code`, `provider_price_id` per interval and currency, `limits` | Plan limits live in code or config, keyed by `code` — never hardcoded in gates |
| `payment_events` | `provider`, `external_event_id` (unique), `type`, `payload`, `processed_at` | The idempotency ledger for every money mutation |
| `invoices` / `refunds` | `provider_ref`, `amount_cents`, `currency`, `status`, `reason` | Integer minor units plus a currency code. Never floats |
| `entitlements` | `owner_id`, `feature`, `limit`, `source` | Only when entitlements outlive a subscription (lifetime deals, comps) |
| `connected_accounts` / `payouts` | `seller_id`, `provider_account_id`, `verification_status`, `payouts_enabled` | Marketplace only |

## Build steps this adds

1. **Plan map and entitlement function** — plan codes, limits, provider price ids per interval; one `entitlements()` function. · *Done when:* unit tests cover every plan code, and a grep-based test fails the build on any plan-name string literal outside the plan map.
2. **Customer creation and hosted checkout** — create-or-fetch the provider customer, redirect to hosted checkout. · *Done when:* a test-mode purchase reaches the success URL, and after a **second** checkout attempt for the same account a script queries the provider's customer-list **API** filtered by that account's external id and asserts exactly one customer exists — proving create-or-fetch, not create-again. Reading it in the vendor's dashboard UI is a launch-checklist item, not a build gate.
3. **Webhook endpoint with signature verification** — raw body, excluded from the body parser. · *Done when:* a correctly signed request returns `200`; an unsigned or tampered request returns `400` and writes nothing.
4. **Idempotent event ledger** — unique constraint on the provider event id, written in the same transaction as the state change. · *Done when:* replaying one event ten times produces exactly one `subscriptions` row and one `payment_events` row.
5. **Refetch-on-webhook** — the handler fetches the object from the provider API rather than trusting the payload. · *Done when:* delivering `updated` before `created` still converges to the correct final state, proven by a test that reverses the order.
6. **Subscription state machine** — all transitions in one module, with grace handling for `past_due`. · *Done when:* a table-driven test asserts access for every status, and a `past_due` account still loads the app while the trial-expired account does not.
7. **Server-side entitlement gating** — limits enforced in the API, not the UI. · *Done when:* a free-plan account is refused past its limit even when the request is sent directly to the API, and the UI renders an upgrade path rather than an error.
8. **Customer portal, upgrades, downgrades** — hosted portal plus a proration preview before confirmation. · *Done when:* an upgrade shows the provider's prorated amount before confirming, and a downgrade takes effect at `current_period_end`, not immediately.
9. **Refunds, disputes, and revocation** — refund handler revokes entitlement; dispute opened revokes digital access. · *Done when:* a partial refund updates the refunded amount, is rejected above the captured amount, and a full refund removes access on the next request.
10. **Tax** — MoR, or a tax service wired with per-line rate storage. · *Done when:* purchases from two jurisdictions produce different, correct tax lines, each recording the rate applied.
11. **Reconciliation and alerting** — nightly compare of provider objects against local state. · *Done when:* the job reports zero mismatches on seeded data, and deliberately deleting a local `subscriptions` row makes it report exactly one.
12. **Marketplace payouts** *(skip if single-seller)* — hosted onboarding, verification gating, split charges, seller statements. · *Done when:* an unverified seller cannot publish, and a completed test order shows platform commission plus seller balance summing exactly to the order total.

## Pitfalls

- **Creating the order or subscription from the browser redirect.** The user closes the tab and the purchase vanishes. The webhook creates the record; the redirect only shows a receipt.
- **Parsing the body before verifying the signature.** The most common integration bug, and it fails open.
- **Trusting webhook payload state.** Refetch. Providers do not guarantee order.
- **Floats for money.** Integer minor units plus currency code everywhere, including JSON. Rounding drift is the most common data corruption in payment code.
- **Entitlement as a boolean column.** It desynchronizes from the subscription the first time a webhook is missed. Derive it.
- **No test-mode E2E.** Money paths need deterministic automated tests, not manual clicks in a dashboard. Provider CLI event triggers exist — use them in CI.
- **Deleting data on non-payment.** Downgrade and retain. Reactivation is the cheapest revenue you will ever get.
- **Ignoring the tax decision until launch.** Choosing a MoR after you have a customer base means migrating every subscription. Decide before checkout is written.
- **Hand-rolling card collection.** Hosted checkout keeps card data out of your systems and your PCI scope small.

## See also

- `knowledge/capabilities/credit-metering.md` — when the purchase grants credits rather than access
- `knowledge/capabilities/database.md` — transactional writes, unique constraints, and money columns
- `knowledge/capabilities/api-design.md` — webhook route conventions and idempotency headers
- `knowledge/capabilities/observability.md` — alerting on webhook lag, failed charges, and reconciliation drift
- `knowledge/shapes/saas-webapp.md` — plans, seats, and entitlement gating in context
- `knowledge/shapes/ecommerce-storefront.md` — carts, refunds, and marketplace splits
- `knowledge/shapes/mobile-app.md` — store in-app purchase rules for digital goods
