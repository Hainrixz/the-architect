# Phase 1: Discovery

Goal of this phase: decide **which of the 14 shapes** the project is, and whether it is a new build
or a change to something that already exists. Nothing else. Stack, hosting, and libraries are Phase 2
and Phase 3 — do not drift into them here.

Last verified: 2026-07-27

---

## How to run this phase

1. **Detect the user's language** from their first message. Conduct the entire interview in it.
   Not English? Read `knowledge/localization.md` now — it is the boundary between what follows the
   user and what stays ASCII English (identifiers, paths, commands, `status` values, EARS keywords).
   For a CJK language also ask, once, whether the project has an English short name: the slug,
   every directory and every git tag are ASCII, and guessing a romanization late is expensive.
2. **Max 3 questions per message.** Hard rule. Never dump the question list.
3. **Gate 0 first** (new vs existing). It changes every downstream phase, so ask it before anything else.
4. Ask Q1–Q3. By the end of Q3 you should have a shape or a shortlist of two.
5. If you have a shortlist, go to **Disambiguation** and ask the ONE resolving question for that pair.
   Ask one question — do not re-run the whole interview.
6. State the shape, state why, read the shape file, move to Phase 2.

Never guess silently. If two shapes are live, say so out loud and ask the tiebreaker.

---

## Gate 0 — New build or existing codebase?

**Ask first, always:**
> "Before anything else — is this a new project from zero, or a change to something that already exists?"

| Answer | Route |
|---|---|
| New, nothing built yet | Continue with Q1 below |
| "I have a repo / it's half-built / I want to add X to my app" | **Stop this flow.** Go to the brownfield flow — `commands/architect-brownfield.md` |
| "I have a design/Figma/spec but no code" | Greenfield. Continue, and treat the artifact as an input in Phase 3 |
| "It exists but I want a full rewrite" | Greenfield shape classification, but note the migration in the blueprint |

Brownfield is not a new-build interview. Reading an existing codebase's conventions beats designing
around it, and the build order becomes a change plan instead of a scaffold. Do not try to force it
through this file.

---

## Core Questions

### Q1 — The vision
> "What are you building? Describe it in your own words — what does it do, and what breaks if it doesn't exist?"

Open-ended on purpose. Listen for the nouns (who uses it, what artifact comes out) more than the
adjectives. Match against the **Signals** table below.

### Q2 — The audience
> "Who uses it? Your own team, paying customers, other developers, or a machine calling it?"

This alone separates `internal-tool` from `saas-webapp`, and `api-backend` from `cli-library-mcp`.
Also get rough scale — 5 users and 50,000 users are different products.

### Q3 — The stage
> "Is this a prototype to prove the idea, or does it need to survive real users on day one?"

Prototype → smart defaults, minimum surface, skip hardening. Production → testing, observability,
error handling, CI in the build order.

### Q4 — Tech constraints *(ask only if not already answered)*
> "Any hard constraints? Existing stack you must match, a language your team knows, a cloud you're locked into?"

Constraints beat preferences. "We're a Python shop" and "must run on our own hardware" change the
runtime track before you get to recommend one.

**One constraint outranks the rest, and it is not a technical one: where the users are.** If Q2's
answer places them inside mainland China — or the brief says 国内 / 大陆 / 境内, names a mainland
cloud, or mentions 备案, 微信小程序, 微信支付 or 支付宝 — say so in the RUNNING BRIEF now and read
`knowledge/capabilities/china-mainland.md` in Phase 2 **before drafting the stack table**. It rewrites
hosting, identity, payments and the model provider together, and it adds regulator-gated filings
measured in weeks. Discovering it in Phase 3 means redesigning, not adjusting. Do not infer it from
the interview language: a Chinese-speaking founder building for a US market needs none of it. Ask.

### Q5 — Timeline *(ask only if scope is unclear)*
> "What's the horizon — a weekend, a couple of weeks, or a quarter?"

Weekend = the smallest thing that ships. Quarter = full architecture with room to grow.

---

## The 14 shapes

| Shape | One line | File |
|---|---|---|
| SaaS / web app | Users sign up, log in, and do work inside it | `knowledge/shapes/saas-webapp.md` |
| Marketing site | Content you publish to convince someone | `knowledge/shapes/marketing-site.md` |
| Mobile app | Ships through an app store, lives on a phone | `knowledge/shapes/mobile-app.md` |
| API / backend | HTTP surface other programs call | `knowledge/shapes/api-backend.md` |
| Internal tool | Ops/admin surface for people who work for you | `knowledge/shapes/internal-tool.md` |
| Content / community platform | Many authors publish; readers interact | `knowledge/shapes/content-community-platform.md` |
| Agent app | An LLM loop with tools is the product | `knowledge/shapes/agent-app.md` |
| Generative media app | You pay per run and a file comes out | `knowledge/shapes/generative-media-app.md` |
| E-commerce storefront | Catalog, cart, checkout, fulfillment | `knowledge/shapes/ecommerce-storefront.md` |
| CLI / library / MCP server | Developers or agents consume it, not end users | `knowledge/shapes/cli-library-mcp.md` |
| Browser extension | Runs inside someone else's web page | `knowledge/shapes/browser-extension.md` |
| Desktop app | Installed binary with OS and filesystem access | `knowledge/shapes/desktop-app.md` |
| Automation / bot / integration | Wakes on an event, no UI, no consumers | `knowledge/shapes/automation-bot-integration.md` |
| Data pipeline / analytics | Moves and models data; correctness is the product | `knowledge/shapes/data-pipeline-analytics.md` |

---

## Signals

Phrases users actually say, mapped to a first guess. A signal is a hypothesis, not a verdict —
confirm with Q2 before committing.

| They say something like | First guess |
|---|---|
| "users sign up", "subscription", "per-seat", "workspaces", "trial" | `saas-webapp` |
| "landing page", "waitlist", "launch page", "convert visitors", "we need a site" | `marketing-site` |
| "iOS", "Android", "App Store", "push notifications", "offline on the phone" | `mobile-app` |
| "endpoints", "our frontend team will consume it", "microservice", "webhook receiver", "public API" | `api-backend` |
| "admin panel", "our ops team", "internal", "replace a spreadsheet", "back office" | `internal-tool` |
| "blog", "docs site with users", "forum", "creators post", "comments", "profiles", "feed" | `content-community-platform` |
| "chatbot", "agent", "it uses tools", "answers questions about our docs", "copilot for X" | `agent-app` |
| "generates images", "makes videos", "voiceover", "credits", "renders", "it takes a minute" | `generative-media-app` |
| "cart", "checkout", "SKUs", "inventory", "shipping", "we sell physical products" | `ecommerce-storefront` |
| "npm package", "CLI", "SDK", "MCP server", "developers install it", "`npx` it" | `cli-library-mcp` |
| "Chrome extension", "adds a button to LinkedIn", "highlights text on any page", "manifest" | `browser-extension` |
| "menu bar app", "runs locally", "reads my files", "installer", "works offline on my Mac" | `desktop-app` |
| "when someone fills the form, then", "Slack bot", "syncs Stripe to Notion", "runs every night" | `automation-bot-integration` |
| "ETL", "warehouse", "dbt", "we need one number we trust", "reporting on 12 sources" | `data-pipeline-analytics` |

### Anti-signals — features that are NOT shapes

The most common misclassification is promoting a **capability** to a shape.

| They mention | It does NOT mean | It means |
|---|---|---|
| Payments / Stripe | `ecommerce-storefront` | a capability — `knowledge/capabilities/payments-rails.md` |
| "AI", "GPT", "it summarizes" | `agent-app` | a capability — `knowledge/capabilities/ai-llm-integration.md` |
| A dashboard screen | `data-pipeline-analytics` | a screen in whatever shape it lives in |
| Login | `saas-webapp` | a capability — `knowledge/capabilities/auth.md` |
| Realtime / websockets | a shape of its own | `knowledge/capabilities/sync-and-collab.md` |

---

## Disambiguation

The hard part. Each pair below has **one** question that resolves it. Ask that question — not five.

### 1. mobile-app vs api-backend vs saas-webapp
*"A mobile app with a backend"* — the single most common ambiguous brief.

> **Ask:** "If I handed you a finished API tomorrow with every endpoint you need, how much of the project would still be left?"

| Answer | Primary | Secondary |
|---|---|---|
| "Almost all of it — the app IS the work" | `mobile-app` | `api-backend` (thin, owned by the same repo) |
| "Barely anything — the hard part is the API" | `api-backend` | none, or a thin client |
| "There's also a full web version people use at a desk" | `saas-webapp` | `mobile-app` |

Follow-up when it's still tied: **"Does anything other than your own app call this backend?"**
No → the backend is an implementation detail of the mobile app; do not treat it as a second shape.
Yes → it is a genuine second shape and needs its own contract and versioning.

### 2. agent-app vs saas-webapp
> **Ask:** "If you replaced every model call with a hardcoded response, is there still a product?"

Yes → `saas-webapp` with the AI capability bolted on. The LLM is a feature.
No → `agent-app`. The loop, the tools, the context management, and the eval harness are the product.

Second tell: does the system decide *what to do next* on its own (tool selection, multi-step), or does
it call a model once per user click? Multi-step autonomy → `agent-app`.

### 3. agent-app vs generative-media-app
> **Ask:** "When the user hits go, do they get back an answer, or a file?"

| They get | Shape |
|---|---|
| Text, a decision, an action taken in another system | `agent-app` |
| An image, video, audio track, deck, or PDF they download | `generative-media-app` |

Both use models. The difference is that media apps have an artifact with storage, a preview, a
gallery, and a download — and a job that outlives the request.

### 4. generative-media-app vs saas-webapp
> **Ask:** "Does one click cost you real money, and does it take more than ten seconds?"

Both yes → `generative-media-app`. That answer forces two things a normal SaaS does not have:
a **credit ledger** (`knowledge/capabilities/credit-metering.md`) and an **async job queue** with
status polling. Those two reshape the data model and the entire build order.

One or neither → `saas-webapp` that happens to generate something.

### 5. cli-library-mcp vs api-backend
> **Ask:** "Who is on the other end — a human at a terminal, another program importing your code, an agent, or a service over the network?"

| Consumer | Shape | Why it differs |
|---|---|---|
| Human at a terminal | `cli-library-mcp` | Distribution is a package registry; there is no uptime |
| Another program importing it | `cli-library-mcp` | The public surface is exported symbols and semver |
| An agent via MCP | `cli-library-mcp` | Tool definitions and a transport, not routes |
| A remote service over HTTP | `api-backend` | You own deploys, auth, rate limits, and an SLA |

If the answer is "you install it and it runs on the user's machine", it is never `api-backend` —
no matter how much logic it contains.

### 6. internal-tool vs saas-webapp
> **Ask:** "Do the people who use this work for you?"

Yes → `internal-tool`. Consequences: SSO over signup flows, no marketing surface, no billing, no
onboarding, permissive UI, and honest audit logging. Speed of building beats polish.

No, strangers pay for access → `saas-webapp`. You now owe signup, billing, multi-tenancy isolation,
and a support surface.

Trap: "an admin panel for our SaaS" is **not** `internal-tool` as the primary shape. It is a
`saas-webapp` whose build order includes an admin area. Ask: **"Is the tool the product, or does it
support the product?"**

### 7. marketing-site vs content-community-platform
> **Ask:** "Does anyone log in and create content?"

No — you and your team publish, visitors only read → `marketing-site`. Content is authored at build
time, the site can be fully static, and the whole job is performance, SEO, and conversion.

Yes — users write posts, comment, have profiles, moderation matters → `content-community-platform`.
You now need auth, a runtime database, a moderation queue, and a feed. The gap between these two in
build effort is roughly 5x, so getting this wrong is expensive.

A blog whose posts you write yourself is a `marketing-site`, even if there are a hundred posts.

### 8. browser-extension vs desktop-app
Separate shapes. They share almost nothing: different manifests, different permission models,
different stores, different update mechanics, different build order start to finish.

> **Ask:** "Does it need to change what someone sees on a web page they're already visiting, or does it need the filesystem and to keep running when the browser is closed?"

| Need | Shape |
|---|---|
| Read/modify pages on sites you don't own, act on the current tab | `browser-extension` |
| Local files, native menus, background daemon, works with no browser open | `desktop-app` |
| Both, honestly | Pick `desktop-app` primary and ship a thin companion extension |

Never answer this with "it can do both" and merge the build orders. They do not merge.

### 9. data-pipeline-analytics vs internal-tool
> **Ask:** "If the dashboard were replaced by a CSV export, would the project still be worth doing?"

Yes → `data-pipeline-analytics`. The deliverable is the pipeline: ingestion, modeling, freshness,
lineage, and tests on the data itself. Charts are the last 10%.

No — the data is already clean and queryable, and the whole job is presenting and acting on it →
`internal-tool`. The deliverable is the UI.

Second tell: is anyone asking "why do these two numbers disagree?" That question means pipeline.

### 10. automation-bot-integration vs api-backend
> **Ask:** "Does something call it, or does it wake up by itself?"

Wakes itself — cron, a webhook from another vendor, a message on a queue, a Slack mention — with no
UI and no consumers you own → `automation-bot-integration`. It optimizes for idempotency, retries,
dead-letter handling, and secrets, not for routes and response shapes.

Something you own calls it and expects a response → `api-backend`.

Tell: if there is no client to write documentation *for*, it is not an API.

### Tiebreaker cheat sheet

| Pair | The one question | → A | → B |
|---|---|---|---|
| mobile-app / api-backend | "Finished API handed to you — what's left?" | Most of it: mobile | Nothing: api |
| agent-app / saas-webapp | "Hardcode the model output — still a product?" | No: agent | Yes: saas |
| agent-app / generative-media-app | "Answer back, or a file back?" | Answer: agent | File: media |
| generative-media-app / saas-webapp | "Costs money per click AND slow?" | Both yes: media | Else: saas |
| cli-library-mcp / api-backend | "Who's on the other end?" | Human/program/agent: cli | Network service: api |
| internal-tool / saas-webapp | "Do users work for you?" | Yes: internal | No: saas |
| marketing-site / content-community | "Does anyone log in and create?" | No: marketing | Yes: community |
| browser-extension / desktop-app | "Other people's pages, or the filesystem?" | Pages: extension | Files: desktop |
| data-pipeline / internal-tool | "Still worth it as a CSV?" | Yes: pipeline | No: internal |
| automation-bot / api-backend | "Called, or self-waking?" | Self: automation | Called: api |
| ecommerce-storefront / saas-webapp | "Do you ship or fulfill anything per order?" | Yes: ecommerce | No: saas |

---

## Composition

Most real projects are **one shape plus capabilities**. Some are genuinely **two shapes**. Forcing a
single answer when the truth is composite produces a blueprint that silently omits half the work.

**The rule:**
1. Name the **primary** shape — the one whose build order the project follows.
2. Name the **secondary** shape only if it has its own deploy target, its own users, or its own
   release cycle. Otherwise it is a capability, not a shape.
3. Say both out loud to the user. Address both in the build order, primary first.

| Common composition | Primary | Secondary | Why it's two |
|---|---|---|---|
| SaaS with a public marketing site | `saas-webapp` | `marketing-site` | Different deploy, different perf budget, different content workflow |
| Mobile app with its own API | `mobile-app` | `api-backend` | Only if something else consumes the API |
| Agent product with a billing dashboard | `agent-app` | `saas-webapp` | The account surface is a real app |
| Media generator with a credits store | `generative-media-app` | — | Credits are a capability, not a shape |
| Storefront with a headless CMS blog | `ecommerce-storefront` | `content-community-platform` | Only if outside authors publish |
| Internal dashboard over a warehouse | `data-pipeline-analytics` | `internal-tool` | Pipeline and UI ship separately |
| SDK plus a hosted API | `api-backend` | `cli-library-mcp` | The package has its own release cadence |

**Cap it at two.** Three shapes means the scope is wrong — split it into two blueprints and say so.
Offer to design the primary now and the second one after, rather than producing one bloated document.

---

## Fast-track mode

If the user says "just build it", "skip the questions", or invokes the quick command
(`commands/architect-quick.md`), ask exactly three:

1. "What is it, in one sentence?"
2. "Who uses it?"
3. "Any stack you're locked into?"

Then classify from the Signals table, pick smart defaults for everything else, state the shape and
the two biggest assumptions you made, and go. Do not run Disambiguation — if it is ambiguous, pick
the more capable shape and say which alternative you rejected.

---

## After classification

1. **State it plainly:** "This is a `<shape>`. Here's why: `<one or two signals from their answers>`."
   If there is a secondary shape, name it in the same breath.
2. If a tiebreaker was needed, say which one and what their answer decided. It builds trust and lets
   them correct you cheaply.
3. **Read `knowledge/shapes/<shape>.md`** — and the secondary shape file if there is one.
4. Proceed to `questions/phase-2-branches.md`, using the section for the primary shape.

Do not read runtime tracks or capability files yet. Phase 2 decides which ones apply.

---

## See also

- `questions/phase-2-branches.md` — the shape-specific deep dive that follows this phase
- `questions/phase-3-confirmation.md` — presenting the architecture for sign-off
- `commands/architect-brownfield.md` — the flow for existing codebases (Gate 0)
- `knowledge/shapes/saas-webapp.md` — the most common landing point; read it to calibrate the others
