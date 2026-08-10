# Phase 2: Shape-Specific Deep Dive

> One branch per shape. Ask the questions that change the blueprint, load the capability files the
> answers point at, then go to Phase 3.

Last verified: 2026-07-27

---

## How to use this file

1. Phase 1 gave you a shape. Read `knowledge/shapes/<shape>.md` **before** opening its branch here —
   the shape file already tells you the default runtime track and core capabilities, so you only need
   to ask about what the shape file leaves open.
2. Jump straight to that branch. Do not read the other thirteen.
3. Ask **max 3 questions per message.** Two is usually better. This is a conversation, not a form.
4. **Skip anything Phase 1 already answered.** Re-asking is the fastest way to lose the user.
5. Answers select capability files. Read the ones that fire, skip the rest.
6. A branch has 3-5 questions. You rarely need all of them. Stop when you can pass the exit check
   at the bottom of this file.

### The relevance test

Before you ask anything, answer this: **what changes in the blueprint if they say the other thing?**

If nothing changes, cut the question and pick a default. "What's your favorite framework?" changes
nothing when the shape file already names a default track. "Does a team own the data or does an
individual?" changes every table in the schema. Ask the second one.

---

## Universal follow-ups

At most **two** of these, and only when the shape branch leaves them genuinely open.

| Ask | Only when | Changes |
|---|---|---|
| "What is explicitly *not* in v1?" | Scope sounds unbounded | Cuts the build order — the most valuable answer in the whole interview |
| "Is there existing code we're extending, or is this greenfield?" | Unclear from Phase 1 | Brownfield means an audit pass before any build step, and inherited stack constraints |
| "Does this touch personal data, payments, or health/financial records?" | Any real user data | Loads `knowledge/capabilities/enterprise-readiness.md` and changes the data model |
| "Any accessibility obligation — public sector, EU, enterprise procurement?" | Any UI, any buyer with a legal team | Loads `knowledge/capabilities/accessibility.md` and adds acceptance criteria, not a polish pass |
| "Who deploys and operates this after it ships?" | Solo founder vs a platform team | Determines how much of `knowledge/capabilities/deployment.md` and `observability.md` is warranted |
| **"Are the users inside mainland China, or outside?"** | **Ask on every project whose audience Phase 1 did not already place** | **Loads `knowledge/capabilities/china-mainland.md`, and it is the one capability that must be read before the stack table rather than after — it changes hosting, auth, payments and the model provider at once** |

**The mainland-China follow-up is a gate, not a preference.** If the answer is "inside", ask one more
question immediately — *"境内部署还是境外部署？"* — because ICP filing gates the host answering for the
domain at all, and everything downstream forks on it. Then read
`knowledge/capabilities/china-mainland.md` in full before Phase 3. Its filings are calendar time owned
by a regulator, so they land in the blueprint's §20.1 manual gates and never in the §9 build order.

---

## Runtime track resolution

Every shape file names a default track. **Do not re-litigate it** unless one of these fires:

| Signal from the user | Action |
|---|---|
| Names a language or framework they already run in production | Honor it. Read the matching `knowledge/runtime-tracks/*.md` and note the deviation in Phase 3 |
| "Our team only knows X" | Team skill beats your preference every time. Switch tracks |
| Heavy ML / data-science work in-process | `knowledge/runtime-tracks/python.md` |
| Single static binary, hard latency floor, or high-concurrency network service | `knowledge/runtime-tracks/go.md` |
| Existing monolith they want to extend | `knowledge/runtime-tracks/rails-laravel.md` |
| Needs a platform API no cross-platform layer exposes | `knowledge/runtime-tracks/mobile-native.md` |
| Nothing above | Keep the shape's default — `knowledge/runtime-tracks/ts-node.md` for most web and app shapes. Say so in one line in Phase 3 and move on |

Cross-check the result against `knowledge/stack-compatibility.md` before Phase 3. Some combinations
are known-bad and it is much cheaper to catch that now.

---

## Skills during Phase 2

A leading `/` means it is really a slash command. **No slash means it auto-activates** — writing it
with a slash is a silent no-op.

| Skill | Use it in Phase 2 for | Notes |
|---|---|---|
| `/last30days` | Current standing of a technology the user named or you are unsure about | Use it instead of guessing at a library's health |
| `find-skills` | **Once per interview.** Discover skills worth recommending for the BUILD phase | Auto-activates. Results go in the blueprint, not into your design work |
| `agent-browser` | The user shared a reference site — pull it to markdown and analyze structure | First choice for reference analysis |
| `browser-harness` | Escalation only: the reference is behind a login | Drives the user's real Chrome. Ask before using |
| `pdf` | The user attached a spec, RFP, or brand guide as a PDF | Extraction only, not design |

**Not in Phase 2:** `ui-ux-pro-max` and `emil-design-eng` belong in Phase 3, once the shape and
capabilities are settled. `frontend-design`, `playwright-cli`, and `/claude-seo-ai:*` are build-phase
tools — recommend them inside the blueprint, do not run them now.

**Never hard-depend on a skill.** If one is not installed, fall back to the knowledge base or
built-in `WebSearch` / `WebFetch`, say so in one line, and keep going.

---

# Branches

Every branch ends with a **capabilities to load** table. Inside those tables a bare filename —
`database.md` — is relative to `knowledge/capabilities/`. Load only the rows whose condition fired;
loading all eighteen capability files produces a bloated, generic blueprint.

---

## Branch: SaaS / Web App
`knowledge/shapes/saas-webapp.md`

**Q1 · Ownership.** "Does a team or workspace own the data, or does each individual user own their own?"
→ *Changes:* every table's owner column, the row-level access rules, whether invites and roles exist
at all, and whether billing is per-seat. This is the single most expensive thing to retrofit.

**Q2 · Money.** "How does this get paid for — flat subscription, per-seat, usage-based, or free for now?"
→ *Changes:* per-seat needs seat counting and proration; usage-based needs a metering table and an
aggregation job; flat needs neither. "Free for now" still needs the user↔plan relation stubbed.

**Q3 · Liveness.** "Does anything have to update without a refresh — and is that notifications, a live
dashboard, or two people editing the same thing at once?"
→ *Changes:* notifications are a cheap subscription; multiplayer editing is a conflict-resolution
project. These are not the same feature and users conflate them constantly.

**Q4 · Buyer.** "Who signs the check — an individual with a credit card, or a company with a
procurement process?"
→ *Changes:* enterprise buyers mean SSO, audit logs, data residency questions, and a security review.
That is weeks of work that must be in the build order from day one, not bolted on.

**Q5 · Edges.** "What has to talk to this from outside — email, calendars, Slack, an existing system?"
→ *Changes:* outbound integrations add a job queue; inbound ones add webhook endpoints and an
idempotency story.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/auth.md`, `database.md`, `frontend-architecture.md`, `deployment.md`, `testing.md` |
| Any UI at all | `knowledge/capabilities/styling.md`, `accessibility.md` |
| Any payment (Q2) | `knowledge/capabilities/payments-rails.md` |
| Usage-based pricing (Q2) | `knowledge/capabilities/credit-metering.md` |
| Multiplayer editing (Q3) | `knowledge/capabilities/sync-and-collab.md` |
| Non-trivial client state (Q3) | `knowledge/capabilities/state-management.md` |
| Enterprise buyer (Q4) | `knowledge/capabilities/enterprise-readiness.md`, `observability.md` |
| Scheduling/booking is core | `knowledge/capabilities/availability-engine.md` |

---

## Branch: Marketing / Landing Page
`knowledge/shapes/marketing-site.md`

**Q1 · The one action.** "What is the single thing a visitor should do — book a demo, start a trial,
buy, or join a list?"
→ *Changes:* the entire page structure and what counts as success. A site with three equal CTAs
converts on none of them.

**Q2 · Who edits, how often.** "Who changes the copy after launch — you in a code editor, or a
non-technical person, and how often?"
→ *Changes:* a non-technical editor means a CMS, a preview environment, and a publish flow. A
developer editing markdown in the repo means none of that exists.

**Q3 · Reach.** "How do people find this — paid ads, search, or you send them the link?"
→ *Changes:* if it is search, structured data, sitemaps, and answer-engine visibility become build
steps with acceptance criteria. If you send them the link, all of that is waste.

**Q4 · Server or not.** "Does anything need to happen server-side — gated content, personalization,
form handling, A/B tests?"
→ *Changes:* the difference between a fully static deploy and a rendered app. Ask before assuming;
most marketing sites need exactly one server function, for the form.

**Q5 · Locales.** "One language, or several — and are they translations of the same pages or different
content per market?"
→ *Changes:* routing, hreflang, and whether the CMS needs a locale dimension. Different content per
market is a much bigger build than translated strings.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/frontend-architecture.md`, `styling.md`, `deployment.md`, `accessibility.md` |
| CMS needed (Q2) | `knowledge/capabilities/database.md` |
| Search matters (Q3) | Note `/claude-seo-ai:audit` and `/claude-seo-ai:geo` as build-phase steps in the blueprint |
| Server-side anything (Q4) | `knowledge/capabilities/api-design.md` |
| Gated content (Q4) | `knowledge/capabilities/auth.md` |
| Sells directly | `knowledge/capabilities/payments-rails.md` |

---

## Branch: Mobile App
`knowledge/shapes/mobile-app.md`

**Q1 · Platforms and the hard requirement.** "iOS, Android, or both — and is there anything it must do
that lives deep in the OS, like background location, Bluetooth, HealthKit, or a widget?"
→ *Changes:* a cross-platform layer covers most apps. One deep platform API can force
`knowledge/runtime-tracks/mobile-native.md` or a hand-written native module, which changes the whole
estimate.

**Q2 · Offline.** "On a plane with no signal — can the user read what they already have, or do they
also need to create and edit, with those changes merging later?"
→ *Changes:* read-only caching is a day. Offline writes mean a local database, a sync queue, and a
conflict-resolution policy you have to decide now.

**Q3 · Backend.** "Is there an API already, or do we design the backend too?"
→ *Changes:* an existing API means adapting to its contract, including its auth. A new one means the
build order carries a whole server track alongside the app.

**Q4 · Money.** "If it is paid — is it in-app purchase, or do people pay on a website and sign in?"
→ *Changes:* in-app purchase means store billing, receipt validation, and the platform's cut.
External payment means keeping purchase flows out of the binary to stay within store policy.

**Q5 · Release reality.** "Who owns the developer accounts, and have you shipped to the stores before?"
→ *Changes:* provisioning, signing, and review timelines are real build steps. A first-time submission
routinely costs a week that nobody planned for.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/auth.md`, `state-management.md`, `deployment.md`, `testing.md`, `accessibility.md` |
| Offline writes (Q2) | `knowledge/capabilities/sync-and-collab.md`, `database.md` |
| New backend (Q3) | `knowledge/capabilities/api-design.md`, `database.md` |
| Any payment (Q4) | `knowledge/capabilities/payments-rails.md` |
| Crash/usage visibility | `knowledge/capabilities/observability.md` |
| Voice or live audio | `knowledge/capabilities/realtime-voice.md` |

---

## Branch: API / Backend Service
`knowledge/shapes/api-backend.md`

**Q1 · Consumers.** "Who calls this — only your own frontend, named partners, or the public internet?"
→ *Changes:* your own frontend means you can change the contract freely. Public consumers mean
versioning, published docs, keys, quotas, and a deprecation policy — a different product.

**Q2 · Contract.** "Is the shape of the data fixed and known, or do callers need to ask for arbitrary
slices of it?"
→ *Changes:* fixed shapes point at resource or RPC endpoints; arbitrary slices are the only honest
reason to take on a graph query layer. Ask about the need, not the acronym.

**Q3 · Identity.** "How does a caller prove who they are — a key they paste into a config, a user
token from your app, or an OAuth grant on behalf of someone else?"
→ *Changes:* keys need issuance, rotation, and scoping. Delegated OAuth needs a consent screen and
token storage. These are not interchangeable.

**Q4 · Async work.** "Is any request too slow to answer inline — file processing, external calls,
scheduled work, fan-out?"
→ *Changes:* introduces a queue, a worker process, job state the caller can poll, and a whole second
deployment target.

**Q5 · Blast radius.** "If one caller sends a hundred times normal traffic, who else notices?"
→ *Changes:* determines rate limiting, per-tenant isolation, idempotency keys on writes, and whether
the database needs read replicas. Ask this even for small APIs — the answer shapes the schema.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/api-design.md`, `database.md`, `auth.md`, `testing.md`, `observability.md`, `deployment.md` |
| Public or partner consumers (Q1) | `knowledge/capabilities/enterprise-readiness.md` |
| Async work (Q4) | Job/queue section of `knowledge/capabilities/deployment.md` |
| Usage-billed access | `knowledge/capabilities/credit-metering.md`, `payments-rails.md` |
| LLM behind the endpoint | `knowledge/capabilities/ai-llm-integration.md` |

---

## Branch: Internal Tool / Dashboard
`knowledge/shapes/internal-tool.md`

**Q1 · Data ownership.** "Where does the data live now — and does this tool write back to a production
database it does not own?"
→ *Changes:* owning the schema means normal migrations. Writing into someone else's production
database means read replicas, guarded writes, and a rollback plan for every mutation.

**Q2 · Identity, already solved.** "How does your team log into other internal things — Google
Workspace, Okta, Entra, or nothing?"
→ *Changes:* an existing identity provider means single sign-on and zero password handling. Building
a login form for eleven colleagues is wasted work and a liability.

**Q3 · The verb.** "What do people actually do here — look at numbers, edit records, or approve things?"
→ *Changes:* dashboards are read paths and caching. Editing needs validation and optimistic UI.
Approvals need state machines, an audit trail, and notifications. Wildly different builds.

**Q4 · Danger.** "Is there any action here that would be expensive to undo — refunds, deletions,
sending something to customers?"
→ *Changes:* forces confirmation flows, permission tiers, an immutable audit log, and soft deletes.

**Q5 · Escape hatches.** "Does anyone need to pull this out — CSV export, a scheduled emailed report?"
→ *Changes:* adds background jobs and file generation. Often replaces half the requested UI.

Before designing: if the honest answer is a spreadsheet plus an off-the-shelf admin builder, **say so.**
Recommending against a build is a legitimate outcome of Phase 2.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/auth.md`, `database.md`, `frontend-architecture.md`, `deployment.md` |
| Existing IdP (Q2) | SSO section of `knowledge/capabilities/auth.md`, plus `enterprise-readiness.md` |
| Approvals (Q3) | `knowledge/capabilities/state-management.md` |
| Destructive actions (Q4) | `knowledge/capabilities/enterprise-readiness.md`, `observability.md` |
| Any table or form | `knowledge/capabilities/styling.md`, `accessibility.md` |

---

## Branch: Content / Community Platform
`knowledge/shapes/content-community-platform.md`

**Q1 · Who writes.** "Does a small editorial team publish, or do users post?"
→ *Changes:* editorial content needs a workflow and a preview. User content needs spam defense,
moderation queues, reporting, rate limits, and a legal position. Do not blur these.

**Q2 · Discovery.** "How does someone find the third-best thing on the platform — search, categories,
a feed, or recommendations?"
→ *Changes:* a feed needs a ranking function and fan-out strategy. Search needs an index and a
relevance story. Categories need a taxonomy someone maintains.

**Q3 · The social graph.** "Do users follow each other, comment, or message privately?"
→ *Changes:* follows create a graph and a notification fan-out. Private messaging is a separate
product with its own abuse surface. Comments alone are far cheaper than all three.

**Q4 · Money.** "Free, ads, subscription, paid content, or creator payouts?"
→ *Changes:* paying creators means multi-party payouts, tax identity collection, and a ledger —
categorically harder than charging readers.

**Q5 · Moderation and law.** "If someone posts something illegal at 3am, what happens?"
→ *Changes:* forces a report flow, a takedown path, retention rules, and a decision about minors.
If they have no answer, that is itself the finding, and it belongs in the blueprint.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/auth.md`, `database.md`, `frontend-architecture.md`, `styling.md`, `accessibility.md`, `deployment.md` |
| User-generated content (Q1) | `knowledge/capabilities/enterprise-readiness.md`, `observability.md` |
| Search or feed (Q2) | Search/indexing section of `knowledge/capabilities/database.md` |
| Notifications or live comments (Q3) | `knowledge/capabilities/sync-and-collab.md` |
| Any money (Q4) | `knowledge/capabilities/payments-rails.md` |
| AI recommendations or summaries | `knowledge/capabilities/ai-llm-integration.md` |

---

## Branch: Agent App
`knowledge/shapes/agent-app.md`

The questions here decide the architecture completely. Get Q1 and Q3 right and the rest follows.

**Q1 · Shape of the run.** "Does the user sit and watch it work, or do they kick it off and come back
later — and could a run take longer than a few minutes?"
→ *Changes:* everything. A synchronous chat is a streaming request handler. A long run is a durable
workflow: persisted state, resumable steps, retries, a run record the user can reopen after closing
the tab. Building the first and discovering you needed the second is a rewrite.

**Q2 · Capability boundary.** "What can it actually *do* — just read and answer, call your internal
APIs, or take actions in outside systems like sending email, filing tickets, or moving money?"
→ *Changes:* read-only agents are a retrieval problem. Acting agents need a tool registry, per-tool
permissions, argument validation, and an irreversibility rating on every tool. Ask explicitly: **which
of these actions cannot be undone?**

**Q3 · The human.** "Before it does something consequential, does a person approve — and who is that
person?"
→ *Changes:* an approval gate means a run can pause for hours, so state must be durable, a queue
and a review UI must exist, and every paused run needs a timeout policy. This is the difference
between a demo and something a business will actually turn on.

**Q4 · What it knows.** "What does it need to read that is not in the model — and roughly how much of
it, how often does it change, and can every user see all of it?"
→ *Changes:* a few hundred stable documents need no vector database. Millions of chunks changing
hourly need an ingestion pipeline and a reindex strategy. **Per-user document permissions are the
hard one** — retrieval must filter by the asker's access, or the agent becomes a data leak.

**Q5 · Cost and proof.** "Who pays for the tokens, is there a ceiling per user, and how will you know
next month's version is better than this month's?"
→ *Changes:* a ceiling means metering and enforcement before the call, not a monthly invoice
surprise. "How will you know it is better" forces an eval set into the build order — without one, an
agent silently regresses and nobody notices.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/agent-loop.md`, `ai-llm-integration.md`, `database.md`, `observability.md`, `deployment.md`, `testing.md` |
| Long-running or approvals (Q1, Q3) | Durable job section of `knowledge/capabilities/deployment.md`, plus `state-management.md` |
| Acts on outside systems (Q2) | `knowledge/capabilities/api-design.md`, `auth.md` |
| Per-user document access (Q4) | `knowledge/capabilities/auth.md`, `enterprise-readiness.md` |
| Token ceiling or resale (Q5) | `knowledge/capabilities/credit-metering.md`, `payments-rails.md` |
| Streaming chat UI | `knowledge/capabilities/frontend-architecture.md`, `state-management.md` |
| Spoken interaction | `knowledge/capabilities/realtime-voice.md` |
| Ships as an MCP server too | `knowledge/shapes/cli-library-mcp.md` |

Invoke the bundled `claude-api` skill before writing any model ID, price, or API parameter into the
blueprint. Never hand-maintain a model table.

---

## Branch: Generative Media App
`knowledge/shapes/generative-media-app.md`

Do not ask whether generation is asynchronous. It always is. Ask what the user experiences while
they wait, and who pays for the failed attempts.

**Q1 · The job.** "Which output — image, video, audio or voice, 3D — and does the user bring their own
input, like a photo to edit or a product shot to place?"
→ *Changes:* pure text-to-output needs no upload path. User inputs mean upload handling, file
validation, storage costs, and a consent question about faces and likenesses.

**Q2 · Providers.** "One provider, or do you want a fallback when a model is deprecated or down?"
→ *Changes:* multi-provider means a normalized job interface, per-provider adapters, and a routing
rule. Generation models are deprecated aggressively; a single hardcoded provider is a scheduled
outage. Recommend the abstraction even for v1.

**Q3 · Waiting and finishing.** "After they hit generate — do they watch a progress bar, or leave and
get told later? And if they close the tab mid-render, what should happen?"
→ *Changes:* a job record with polling is the floor. Live progress adds a subscription channel.
"Tell me later" adds email or push. "Survives a closed tab" rules out doing the work in the request.

**Q4 · The meter.** "Credits, a subscription with limits, or pay per render — and if a generation
fails, do they get their credits back?"
→ *Changes:* credits need a ledger with holds, not a decrementing integer — you reserve before the
call and settle or refund after. Whether credits expire, roll over, or are refunded on failure must
be decided now; it is a schema decision and a support-ticket generator.

**Q5 · Liability and storage.** "Who is responsible if someone generates something they should not —
and where do the outputs live, for how long, and who can see them?"
→ *Changes:* forces input and output moderation, a review queue or an automated classifier, a
takedown path, signed versus public URLs, and a retention window. Also settle commercial rights,
because paying users will ask on day one.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/ai-llm-integration.md`, `credit-metering.md`, `database.md`, `deployment.md`, `observability.md`, `auth.md`, `state-management.md` |
| Money changes hands (Q4) | `knowledge/capabilities/payments-rails.md` |
| Live progress (Q3) | `knowledge/capabilities/sync-and-collab.md` |
| Voice or spoken output (Q1) | `knowledge/capabilities/realtime-voice.md` |
| Gallery, feed, or sharing | `knowledge/shapes/content-community-platform.md` |
| An LLM plans multi-step edits | `knowledge/capabilities/agent-loop.md` |
| Any UI | `knowledge/capabilities/frontend-architecture.md`, `styling.md`, `accessibility.md` |

---

## Branch: E-commerce Storefront
`knowledge/shapes/ecommerce-storefront.md`

**Q1 · What is sold.** "Physical goods, digital downloads or licenses, a subscription, or someone's
time?"
→ *Changes:* physical means inventory, shipping rates, addresses, and returns. Digital means license
issuance and delivery. Time means a booking calendar — a genuinely different build, and possibly a
different shape.

**Q2 · How many sellers.** "Is this your catalog only, or do other people sell through it?"
→ *Changes:* a marketplace needs seller onboarding with identity verification, split payments,
payout schedules, and per-seller reporting. It is several times the work of a single-seller store.
Confirm it before assuming either way.

**Q3 · Merchant of record.** "Who legally makes the sale and owes the tax — your company, or a
provider that takes on that role?"
→ *Changes:* the highest-leverage answer in this branch. A merchant-of-record provider absorbs
global sales tax, VAT, and invoicing. Being your own merchant of record means tax calculation,
registration thresholds, and remittance become your build steps. Recommend the provider unless they
have a tax function already.

**Q4 · Catalog shape.** "Do products have variants or options, do prices change by country or
customer, and roughly how many items — and who edits them?"
→ *Changes:* variants explode the schema and the URL structure. Regional pricing adds a currency and
tax-inclusive display decision. A non-technical merchandiser means an admin UI is a build step, not
a database seed.

**Q5 · Fulfillment truth.** "Where does stock live, and what should happen when two people buy the
last one at the same time?"
→ *Changes:* reveals whether inventory is authoritative here or synced from a warehouse system, and
forces an explicit oversell policy — reserve at checkout, or accept and refund. It also drags in
returns and partial refunds, which people always forget.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/payments-rails.md`, `database.md`, `frontend-architecture.md`, `deployment.md`, `observability.md`, `accessibility.md`, `styling.md` |
| Accounts or order history | `knowledge/capabilities/auth.md` |
| Marketplace (Q2) | `knowledge/capabilities/enterprise-readiness.md` |
| Subscriptions (Q1) | Recurring section of `knowledge/capabilities/payments-rails.md` |
| Selling time (Q1) | `knowledge/capabilities/availability-engine.md` |
| Search and organic traffic matter | `knowledge/capabilities/api-design.md`, plus `/claude-seo-ai:audit` as a build-phase step |
| Live stock or cart sync | `knowledge/capabilities/sync-and-collab.md` |

---

## Branch: CLI / Library / MCP Server
`knowledge/shapes/cli-library-mcp.md`

**Q1 · Which one, and who consumes it.** "Is this run by a human at a terminal, imported by another
developer, or called by an AI agent through MCP?"
→ *Changes:* three different contracts. Humans need flags, help text, sensible defaults, and readable
errors. A library needs a stable typed surface and no side effects on import. An MCP server needs
tool names and descriptions a model can choose correctly from — the description *is* the interface.

**Q2 · Public surface.** "What exactly are you promising not to break — which commands, which exported
functions, which tools?"
→ *Changes:* defines the entry point set and forces everything else to be internal. Without this
answer, every implementation detail becomes a compatibility obligation by accident.

**Q3 · Distribution.** "How does someone get it — a package registry, a package manager, a single
downloadable binary, or a plugin marketplace?"
→ *Changes:* a single self-contained binary points hard at `knowledge/runtime-tracks/go.md`. A
registry package points at the ecosystem your consumers already use. This decides the runtime track
more than anything else in this branch.

**Q4 · Compatibility policy.** "When you need to remove something, how much warning do people get —
and how many old versions do you support?"
→ *Changes:* determines whether the build order includes a deprecation mechanism, feature detection,
and a compatibility test matrix in CI, or none of that.

**Q5 · Environment.** "Where does it get credentials and config, and does it need to behave
differently when there is no human watching — CI, a pipe, a script?"
→ *Changes:* non-interactive mode means machine-readable output, meaningful exit codes, no prompts,
and no colors. Secrets handling decides config file versus environment versus OS keychain.

**MCP specifics.** If the answer to Q1 includes MCP, also settle: local stdio or remote HTTP, and if
remote, how callers authenticate. On statefulness: the latest **ratified** MCP spec revision is
stateful. A "stateless" revision exists only as an unratified draft, and its own compatibility notes
say modern-only servers fail against currently deployed hosts. **Build dual-era and default to
performing `initialize`.** Do not write that MCP went stateless.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/api-design.md`, `testing.md`, `deployment.md` |
| Persists anything | `knowledge/capabilities/database.md` |
| Remote MCP or hosted service | `knowledge/capabilities/auth.md`, `observability.md` |
| Wraps a model | `knowledge/capabilities/ai-llm-integration.md` |
| Metered or paid access | `knowledge/capabilities/credit-metering.md` |

---

## Branch: Browser Extension
`knowledge/shapes/browser-extension.md`

**Q1 · Targets.** "Chrome only, or also Firefox and Safari?"
→ *Changes:* Chrome-family is one codebase. Safari requires wrapping the extension in a native app
shell, an Apple developer account, and a separate review. Firefox differs in background-script
lifetime. Each added browser is a real chunk of the build order, not a config flag.

**Q2 · Reach into the page.** "Does it need to read or change pages on *any* site, or a specific list
of sites — or can it act only when the user clicks the icon?"
→ *Changes:* the permission model, which is also the review model. Broad host access triggers a
scary install prompt, slower store review, and a justification you must write. Click-scoped access
avoids all three. Push hard toward the narrowest scope that still works.

**Q3 · Backend.** "Does it need an account, sync across devices, server-side processing, or a paid
tier?"
→ *Changes:* introduces a server, and extension authentication is its own problem — a content script
cannot rely on your site's cookies, so plan a token exchange and where the token is stored.

**Q4 · Data and code.** "What data leaves the user's machine, and is any logic fetched at runtime
rather than shipped in the package?"
→ *Changes:* manifest v3 forbids executing remotely hosted code, so anything dynamic must be data
interpreted by shipped code. Data collection also drives the store privacy disclosure, which is a
publishing blocker if wrong.

**Q5 · Distribution.** "Public store listing, unlisted link, or enterprise-managed install — and who
owns the developer account?"
→ *Changes:* store review latency and rejection risk become schedule items. Enterprise deployment
means policy-based install and no store review at all.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/frontend-architecture.md`, `state-management.md`, `styling.md`, `testing.md`, `deployment.md` |
| Backend or accounts (Q3) | `knowledge/capabilities/auth.md`, `api-design.md`, `database.md` |
| Paid tier (Q3) | `knowledge/capabilities/payments-rails.md` |
| Error visibility in the wild | `knowledge/capabilities/observability.md` |
| Calls a model | `knowledge/capabilities/ai-llm-integration.md` |
| Enterprise distribution (Q5) | `knowledge/capabilities/enterprise-readiness.md` |

---

## Branch: Desktop App
`knowledge/shapes/desktop-app.md`

**Q1 · Why not a web app.** "What does this need from the machine — the filesystem, local hardware, a
background process, offline work, system-level shortcuts?"
→ *Changes:* if the honest answer is "we want an icon in the dock", say so and recommend a web app.
If it is filesystem or hardware access, that requirement drives the entire architecture and the
choice of shell.

**Q2 · Targets and audience.** "macOS, Windows, Linux — and are you shipping to the public or to a
managed corporate fleet?"
→ *Changes:* each OS is its own packaging, signing, and installer path. A managed fleet means silent
install packages and no consumer store at all.

**Q3 · Signing.** "Do you already have an Apple developer account and a Windows signing certificate?"
→ *Changes:* unsigned desktop software is effectively unshippable — users hit a security wall.
Obtaining a certificate takes real calendar time and must appear early in the build order, not at the
end. Ask before designing the release pipeline.

**Q4 · Updates.** "How do users get version two — silently in the background, prompted, or by
re-downloading?"
→ *Changes:* auto-update means an update feed you host, signed release artifacts, staged rollout, and
a rollback plan. It is a subsystem, not a checkbox.

**Q5 · Local data.** "What is stored on the machine, is it sensitive, and does it need to survive the
machine being lost?"
→ *Changes:* local-only means a local store and a backup story. Syncing to a server means accounts,
conflict resolution, and encryption decisions.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/deployment.md`, `frontend-architecture.md`, `state-management.md`, `testing.md`, `accessibility.md` |
| Local persistence (Q5) | `knowledge/capabilities/database.md` |
| Syncs to a server (Q5) | `knowledge/capabilities/sync-and-collab.md`, `auth.md`, `api-design.md` |
| Crash reporting and telemetry | `knowledge/capabilities/observability.md` |
| Licensing or subscription | `knowledge/capabilities/payments-rails.md` |
| Corporate fleet (Q2) | `knowledge/capabilities/enterprise-readiness.md` |
| Embeds a model locally or remotely | `knowledge/capabilities/ai-llm-integration.md` |

---

## Branch: Automation / Bot / Integration
`knowledge/shapes/automation-bot-integration.md`

**Q1 · Trigger.** "What starts it — a schedule, an incoming event or webhook, a person typing a
command, or polling something that has no webhooks?"
→ *Changes:* a schedule needs a cron host and overlap protection. Webhooks need a public endpoint,
signature verification, and a fast acknowledgement. Polling needs cursor state and a backoff policy.
Chat commands need to respond within the platform's acknowledgement window and defer the real work.

**Q2 · Host platform.** "Which platform is it living inside — and is this for one workspace or will
other organizations install it?"
→ *Changes:* single-workspace means one static credential in an environment variable. Multi-tenant
install means an OAuth flow, per-tenant token storage, refresh handling, and a listing review. Also
surfaces platform-specific limits: rate caps, payload sizes, and acknowledgement deadlines.

**Q3 · Idempotency.** "If the same event arrives twice — and it will, because every platform retries —
what must not happen twice?"
→ *Changes:* forces a dedupe key, a processed-events table, and idempotent writes. Skipping this is
the single most common way these projects fail in production, silently and expensively.

**Q4 · Failure.** "When step three of five fails, should it retry, roll back, or stop and tell a
human — and who is that human?"
→ *Changes:* determines retry policy with backoff, a dead-letter destination, whether partial work is
compensated, and where alerts land. Also decides whether runs need to be replayable.

**Q5 · Judgment.** "Does anything in the flow require a decision that is not a fixed rule?"
→ *Changes:* if yes, an LLM enters the loop and this inherits the agent shape's concerns — tool
permissions, cost, and evaluation. If no, keep it deterministic; it will be cheaper and far more
reliable.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/api-design.md`, `database.md`, `deployment.md`, `observability.md`, `testing.md` |
| Multi-tenant install (Q2) | `knowledge/capabilities/auth.md`, `enterprise-readiness.md` |
| LLM in the loop (Q5) | `knowledge/capabilities/agent-loop.md`, `ai-llm-integration.md` |
| Per-tenant usage limits | `knowledge/capabilities/credit-metering.md` |
| Paid installs | `knowledge/capabilities/payments-rails.md` |
| Has a configuration UI | `knowledge/capabilities/frontend-architecture.md` |

---

## Branch: Data Pipeline / Analytics
`knowledge/shapes/data-pipeline-analytics.md`

**Q1 · Freshness, in numbers.** "How stale can the data be before someone is upset — five minutes, an
hour, this morning's?"
→ *Changes:* everything. People say "real time" and mean "by 9am". A daily batch is a scheduled job.
Sub-minute freshness is a streaming system with an operations burden ten times larger. Make them
answer in units of time, not adjectives.

**Q2 · Sources.** "Which systems does the data come from, how do you get it out — API, database
replica, change capture, file drops — and roughly what volume per day?"
→ *Changes:* determines extraction method, whether backfill is even possible, and whether the volume
justifies a warehouse or fits comfortably in a normal database. Most projects are far smaller than
they assume.

**Q3 · Who reads it.** "Is the output for your internal team, or does it appear inside your product
for customers?"
→ *Changes:* the biggest fork in this branch. An internal dashboard tolerates slow queries and
trusted access. Customer-facing analytics needs per-tenant isolation, query latency budgets,
authorization on every read, and a cost model — effectively a product, not a report.

**Q4 · Correctness.** "What happens when data arrives late, arrives twice, or the source changes a
column name without telling you?"
→ *Changes:* forces idempotent, re-runnable transformations, a restatement strategy, schema-drift
detection, and data-quality tests as build steps with real acceptance criteria. Skipping this is how
dashboards become quietly wrong.

**Q5 · Governance.** "Is there personal or regulated data in here, how long may you keep it, and is
anyone forbidden from seeing certain columns?"
→ *Changes:* adds masking, column-level access rules, a retention and deletion job, and lineage
tracking that a compliance reviewer can follow.

**Capabilities to load**

| If | Load |
|---|---|
| Always | `knowledge/capabilities/database.md`, `deployment.md`, `observability.md`, `testing.md` |
| Customer-facing output (Q3) | `knowledge/capabilities/auth.md`, `api-design.md`, `frontend-architecture.md` |
| Personal or regulated data (Q5) | `knowledge/capabilities/enterprise-readiness.md` |
| Usage-based billing on the data product | `knowledge/capabilities/credit-metering.md`, `payments-rails.md` |
| Charts and dashboards in a UI | `knowledge/capabilities/styling.md`, `accessibility.md` |
| Embeddings, semantic search, or model features | `knowledge/capabilities/ai-llm-integration.md` |

---

## Hybrid projects

Real projects overlap. Pick the **primary** shape — the one that carries the risk and most of the
build order — run its branch fully, then borrow at most two questions from the secondary branch.

| Combination | Primary | Borrow |
|---|---|---|
| SaaS with a marketing site | `knowledge/shapes/saas-webapp.md` | Marketing Q1 and Q3, and treat the site as its own deploy target |
| AI feature inside an existing product | The existing shape | Agent App Q1, Q2, and Q5 |
| Mobile app plus its backend | `knowledge/shapes/mobile-app.md` | API Backend Q3 and Q4 |
| Store that also publishes content | `knowledge/shapes/ecommerce-storefront.md` | Content Q2 |
| Internal tool that runs automations | `knowledge/shapes/internal-tool.md` | Automation Q3 and Q4 |
| Product that also ships an MCP server | The product shape | CLI/MCP Q1 and its MCP specifics |

Name the secondary shape explicitly in Phase 3 so the build order covers both.

---

## Exit check

Do not move to Phase 3 until you can state, without hedging:

1. **The shape**, and the secondary shape if it is a hybrid.
2. **The runtime track**, and one sentence on why — default or deviation.
3. **The capability list**, drawn only from `knowledge/capabilities/`, with each one traceable to
   something the user actually said.
4. **The core entities** and who owns each row.
5. **The money model**, if any money exists.
6. **The riskiest part of the build**, and what makes it risky.
7. **What is out of scope for v1.**

If any of these is still fuzzy, ask one more question. One. Then go to Phase 3.

---

## See also

- `questions/phase-1-discovery.md` — shape classification that feeds this file
- `questions/phase-3-confirmation.md` — presenting the architecture for sign-off
- `knowledge/stack-compatibility.md` — check the shape + track + capability combination before Phase 3
- `knowledge/skills-registry.md` — authoritative skill names and invocation forms
