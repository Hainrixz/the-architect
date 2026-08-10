# Shape: E-commerce Storefront

> A catalog of things people buy — browse, cart, checkout, pay, fulfil, return — built by a merchant, a brand team, or a marketplace operator.

Last verified: 2026-07-27

## Is this your project?

**Yes if:** the user says "sell", "store", "SKUs", "checkout", "shipping"; revenue is per-transaction on a catalog, not per-seat on a plan; there are variants (size, colour, licence tier) and something decrements when one sells; someone downstream picks, packs, ships, or delivers a file — and handles returns; or several sellers list on one site and the money has to split.

**No if:**
- It's recurring access to software billed per seat or plan → `knowledge/shapes/saas-webapp.md`
- It's one product, one buy button, and the real work is conversion copy and ranking → `knowledge/shapes/marketing-site.md`
- It's paid content behind a membership paywall with a feed → `knowledge/shapes/content-community-platform.md`
- You're building the commerce API someone else's storefront consumes → `knowledge/shapes/api-backend.md`
- The store *is* a native app, with push, wallet, and loyalty → `knowledge/shapes/mobile-app.md`
- Nobody buys anything here — staff operate orders, refunds, and stock that some other system already took money for → `knowledge/shapes/internal-tool.md`. A merchant back-office *attached* to your own storefront is the `(admin)/` group below, not a separate shape; a back-office over a platform you did not build is that shape.
- What is sold is generated on demand per order — an image, a video, a voice clip — and the queue, credits, and provider cost are the real system → `knowledge/shapes/generative-media-app.md`

## Default runtime track

**TypeScript/Node** — see `knowledge/runtime-tracks/ts-node.md`. Storefronts are read-heavy, SEO-critical, and edge-cacheable; a full-stack React framework gives server rendering, per-route caching, and webhook handlers in one deploy.
Alternatives: `knowledge/runtime-tracks/rails-laravel.md` when the merchant back-office (pricing rules, purchase orders, warehouse users) is the bulk of the work — batteries-included admin beats hand-rolling it. `knowledge/runtime-tracks/python.md` when the catalog comes out of an existing data or ML pipeline. `knowledge/runtime-tracks/mobile-native.md` only when the app is the store.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|
| Payments | Checkout, refunds, disputes, split payouts, tax | `knowledge/capabilities/payments-rails.md` |
| Database | Catalog, inventory, orders — transactional, not eventual | `knowledge/capabilities/database.md` |
| Auth | Guest checkout first, accounts second, admin roles third | `knowledge/capabilities/auth.md` |
| Frontend architecture | Cached PDP/PLP plus dynamic cart is the whole rendering problem | `knowledge/capabilities/frontend-architecture.md` |
| API design | Webhook idempotency, product feeds, agent-facing reads | `knowledge/capabilities/api-design.md` |
| Deployment | CDN for catalog, region-pinned database for orders | `knowledge/capabilities/deployment.md` |
| Observability | A silent checkout failure is revenue you never see | `knowledge/capabilities/observability.md` |
| Testing | Money paths need deterministic tests, not manual clicks | `knowledge/capabilities/testing.md` |
| Accessibility | Checkout is where inaccessible UI becomes a legal claim; also what agents parse | `knowledge/capabilities/accessibility.md` |
| Enterprise readiness | Marketplace only — KYC, seller audit trail, payout records | `knowledge/capabilities/enterprise-readiness.md` |
| Mainland China operation | Selling to mainland buyers replaces the card rail with WeChat Pay and Alipay, and adds ICP filing to the critical path | `knowledge/capabilities/china-mainland.md` |

## Branches that change everything

**Should you build this at all?** Be willing to say no.

| Situation | Recommendation | Why |
|---|---|---|
| Physical goods, small team, standard flow | Hosted commerce platform's own storefront and theme | Cart, tax, fraud, address validation, promo stacking, returns, and a warehouse-ready admin already exist |
| Digital goods sold worldwide, small team | Merchant of record (Lemon Squeezy, Paddle, Polar) | They become seller of record and own global VAT/GST — the biggest hidden cost of digital goods |
| Bespoke configurator, quote-to-cart, or commerce *is* the product | Build it | Nothing off the shelf models your pricing |
| Marketplace with many sellers and split money | Build the platform, rent the rails (Stripe Connect or equivalent) | Onboarding, KYC, and payouts are not worth writing |

**Walk this ladder during the interview, not during the build.** It is a Phase 3 decision: the chosen path and the two rejected options each go into the blueprint's decision log with a `Would reverse if` trigger, before the build order is written. It is deliberately *not* build step 1 — a step whose acceptance criterion is "the blueprint says so" is already satisfied the moment the blueprint exists, and gates nothing.

**Headless commerce is a trap for most small teams.** You keep paying the platform *and* inherit rebuilding cart, checkout, tax, fraud rules, discount stacking, and the admin. Choose it only when storefront design or performance is a genuine competitive weapon and someone owns it full-time.

**Physical vs digital** — never one flag on `Product`; fulfilment is a strategy per line item.

| | Physical | Digital |
|---|---|---|
| Inventory & tax | Real stock, reservations, oversell risk; origin/destination sales tax, nexus thresholds, duties | Unlimited or a licence-key pool; VAT/GST at buyer location from the first sale, no threshold |
| Fulfilment & returns | Warehouse or 3PL, labels, tracking, partial shipments; RMA, inbound shipping, restock, condition grading | Signed expiring download URL or licence issuance; refund window plus entitlement revocation, no restock |

**Marketplace (multiple sellers).** Only take this on when sellers genuinely receive money; it roughly doubles the build. See `knowledge/capabilities/payments-rails.md`.

- **Money flow.** Buyer pays the platform, platform takes commission, remainder reaches the seller via the provider's connected-account model. Never route seller money through your operating account — that makes you a money transmitter.
- **Onboarding and KYC.** Identity verification runs in the provider's hosted flow, not yours. A seller stays `pending` — cannot publish, cannot be paid — until the provider reports verification complete and payouts enabled.
- **Payouts and disputes.** Provider-managed on a schedule, with a rolling reserve for physical goods since chargebacks land after delivery. Every payout reconciles to order lines; ship a seller-facing statement on day one. Decide in writing whether platform or seller eats a chargeback and encode it as a column.
- **Multi-seller carts.** One cart becomes several orders, each with its own fulfilment and payout, grouped by a single payment intent. Retrofitting this later is a rewrite.

**Agent-readable commerce.** AI shopping agents are a real channel — treat them as a client, not an SEO afterthought.

- Emit structured product data (Schema.org `Product`/`Offer`) with price, currency, availability, GTIN or MPN, and shipping/return policy on every product page.
- Price and availability must be present in server-rendered HTML, on a stable canonical URL that is never session- or experiment-scoped. If an agent must run your JS to learn the price, you are invisible.
- Publish a machine-readable product feed and an `llms.txt` pointing at it, your policies, and support contact.
- Agentic *checkout* protocols are still shifting: build the read path now, treat agent-initiated purchase as opt-in, verify the current specification before implementing, and run agent orders through the same fraud, rate-limit, and reservation rules as humans.

## Data model

| Entity | Notes |
|---|---|
| `Product` | Marketing container: title, description, media, collections. Not sellable. |
| `Variant` | **The sellable unit.** SKU, option values, weight, tax class, fulfilment strategy. |
| `Price` | Per variant, per currency, optional per market. Integer minor units plus currency code — never floats. |
| `InventoryLevel` | Per variant per location: `on_hand`, `reserved`, `available`. Reserve at checkout start, commit on payment, release on expiry. |
| `Cart` / `CartLine` | Anonymous-capable, cookie-bound, expiring. Totals recomputed server-side on every read. `Customer`/`Address` optional — never required to buy. |
| `Order` / `OrderLine` | **Immutable snapshot** of title, price, tax, and discount at purchase time. Never join a historical order to a live product. |
| `Payment` / `WebhookEvent` | Provider intent ID, status, captured and refunded amounts. `WebhookEvent` holds the provider event ID under a unique constraint — the idempotency ledger for every money mutation. |
| `Shipment` / `Entitlement` | Physical: carrier, tracking, lines shipped, partial-shipment capable. Digital: what may be downloaded, how often, until when. |
| `ReturnRequest` / `Discount` | Return lines, reason, condition, refund amount, restock decision. Discounts: code or automatic, stacking rules, usage caps, per-customer limits. |
| `Seller` / `Payout` | Marketplace only. Seller holds provider account ID and verification status. |

## Directory structure

```
app/                          # shown for the TypeScript track
  (storefront)/
    products/[handle]/        # PDP — cached, structured data emitted here
    collections/[slug]/       # PLP — cached, faceted
    cart/ checkout/           # dynamic, no-store, never cached
    orders/[token]/           # guest-visible order status
  (admin)/                    # catalog, orders, inventory, sellers
  api/webhooks/payments/      # signature-verified, idempotent
  api/feed/                   # machine-readable product feed
lib/catalog/ lib/cart/        # queries + cache tags; server-authoritative mutations
lib/pricing/ lib/inventory/   # money, tax, discounts; reserve / commit / release
lib/fulfillment/ lib/payouts/ # physical + digital strategies; marketplace only
tests/money/ tests/e2e/       # pricing math; browse → cart → pay → order
```

## Build order

1. **Scaffold and prove the money rails reach a sandbox** — project init per the runtime track, linter, formatter, CI; payment-provider test-mode credentials loaded from the environment and exercised by a script. (The build-vs-buy ladder was settled in Phase 3 — see above.) · *Done when:* the track's lint, typecheck, and build commands exit 0 on a clean checkout in CI; a script authenticates against the provider's test API and creates then voids a zero-value intent, exiting 0; and booting with the provider key unset exits non-zero naming the missing variable.
2. **Money and catalog primitives** — money type in integer minor units; `Product`/`Variant`/`Price` schema; seed ~20 variants across two option axes. · *Done when:* unit tests prove `add`/`multiply`/`allocate` never lose a minor unit, and seeding produces the expected variant count.
3. **PLP and PDP, server-rendered** — collection listing, variant selector, tag-based caching. · *Done when:* view-source on a product page shows price and availability with JS disabled, and a price change invalidates that page within one revalidation cycle.
4. **Server-authoritative cart** — add/update/remove lines, all totals computed server-side. · *Done when:* a tampered client-side price is ignored — the API returns the true total and a test asserts it.
5. **Inventory reservation** — reserve on checkout start with a TTL, release on expiry. · *Done when:* two concurrent checkouts for the last unit yield exactly one success and one out-of-stock, proven by a concurrency test.
6. **Checkout and payment** — provider-hosted or provider-element checkout; the order is created from the webhook, never from the browser redirect. · *Done when:* a test-mode purchase creates exactly one `Order`, and replaying the same webhook event ID creates zero additional rows.
7. **Tax** — wire a tax service or a merchant of record; store per-line tax and the rate applied. · *Done when:* orders to two jurisdictions produce different, correct tax lines and each records its rate.
8. **Fulfilment, order status, and transactional email** — shipment records for physical, signed expiring entitlements for digital, guest-visible order page, confirmation and shipping notices. · *Done when:* a digital order's download URL returns 200 with the file inside its TTL and 403 after it; a physical order ships partially and shows two tracking numbers; the confirmation email carries the correct total and a working status link.
9. **Refunds, returns, and discounts** — RMA request, partial refund, restock decision, discount stacking policy and usage caps. · *Done when:* a partial refund increases `amount_refunded`, restocks the line, and is rejected above the captured amount; tests cover the four documented stacking cases.
10. **Agent-readable layer** — structured data, product feed, `llms.txt`. · *Done when:* the feed and a PDP validate in a structured-data testing tool with zero errors, and a JS-disabled fetch returns price plus availability.
11. **Marketplace: sellers and payouts** *(skip if single-seller)* — hosted onboarding, verification gating, split payments, seller statements. · *Done when:* an unverified seller cannot publish a variant, and a completed test order shows platform commission plus seller balance summing to the order total.
12. **Observability and reconciliation** — alerts on checkout error rate, webhook lag, payment/order mismatch. · *Done when:* a nightly job reports zero orders lacking a matching payment, and a forced webhook failure fires an alert.

## Pitfalls

- **Trusting client-supplied prices or totals.** Recompute every amount server-side from the variant record at mutation time. Treat client money values as decoration.
- **Floats for money.** Integer minor units plus currency code everywhere, including JSON. Rounding drift in tax and split payouts is the most common data corruption in this shape.
- **Orders joined to live products.** Renaming a product must not rewrite last year's invoices. Snapshot title, price, tax, and discount onto the order line.
- **Non-idempotent webhooks.** Providers retry and deliver out of order. Store the provider event ID under a unique constraint and process it in the same transaction as the order write.
- **Overselling and deferred tax.** Decrementing stock at payment success rather than reserving at checkout start guarantees oversell under concurrency. Digital goods owe VAT/GST from the first cross-border sale — decide tax handling before writing checkout, not after launch.
- **Caching the cart.** Cart, checkout, and account pages are per-user and must be `no-store`. One bad cache header leaks another shopper's cart.
- **Decisions deferred into rewrites.** Single-seller order models cannot be split later — model `Seller` and per-seller `Order` in step 2 if sellers appear in the brief at all. Substring search collapses past a few thousand SKUs, so plan the index at design time. Forced signup is the largest abandonment lever here — ship guest checkout first.

## Skills for the build phase

See `knowledge/skills-registry.md` for the full list. For this shape: `ui-ux-pro-max` for palette, type scale, and the PDP/PLP component system before any UI is written; `frontend-design` for build-phase storefront UI; `emil-design-eng` for cart and checkout micro-interactions, where perceived speed converts; `/claude-seo-ai:audit` and `:geo` for product-page indexability and AI-answer-engine visibility, running `:geo` after the agent-readable layer lands; `playwright-cli` for E2E over browse → cart → pay → order in provider test mode; `agent-browser` to analyse a competitor storefront the user references; `/humanizalo` for product copy and transactional email. If any is unavailable, fall back to this knowledge base plus `WebSearch`/`WebFetch`, note it in one line, and continue.

## See also

- `knowledge/runtime-tracks/ts-node.md` — pinned versions and scaffolding for the default track
- `knowledge/capabilities/payments-rails.md` — checkout, refunds, connected accounts, payouts
- `knowledge/capabilities/database.md` — transactional modelling for inventory and orders
- `knowledge/capabilities/api-design.md` — webhook idempotency, feeds, agent-facing endpoints
- `knowledge/stack-compatibility.md` — combinations to avoid before committing
- `knowledge/shapes/saas-webapp.md` — when revenue is a recurring plan, not a catalog
- `knowledge/shapes/internal-tool.md` — when the back-office is the whole deliverable and nobody checks out
