# Stack compatibility

> The cross-axis check. A shape, a runtime track, and a capability each look fine on their own —
> this file catches the combinations that only break once you put them together.

Last verified: 2026-07-27

**No version numbers live in this file.** Every combination below is stated as a *role* — "a
full-stack React framework", "a typed SQL-shaped ORM", "a managed Postgres with built-in auth". What
those roles resolve to today, and at which pinned version, lives in `knowledge/runtime-tracks/`.
Services are named directly (Supabase, Stripe, Cloudflare, Vercel) because services are not pinned.
A rule written against a version number expires within a release; a rule written against a role does
not.

## How to use it

1. Draft the stack from the shape's default track plus the capabilities the brief needs.
2. Read the known-bad table. A hit is a design error, not a warning — change the stack, or write down
   in the blueprint why the conflict does not apply to this project.
3. Record the result in the blueprint stack section: *"Checked against
   `knowledge/stack-compatibility.md` — no known-bad combinations"*, or name the conflict and the
   resolution.

"No known-bad combinations" is a claim about *this table*, not about the stack. Write it as
"none of the rows below apply", never as "no conflicts exist" — the rows here are the ones someone
already paid for. Anything the default track mandates against itself (a linter and a styling engine
shipped by the same track, a package manager and a scaffolder shipped by the same track) belongs in
the runtime track's Gotchas, and the blueprint must carry those forward even when this table is
clean.

Verify anything load-bearing against the vendor's own docs before repeating it. Several of the most
quoted incompatibilities in this space were fixed by the vendor years ago and survive only as
folklore — see [Rules that expired](#rules-that-expired).

## Proven combinations

| Combination | Composition | Track | Why it holds |
|---|---|---|---|
| **Default SaaS** | full-stack React framework · utility-CSS framework + copy-in component library · managed Postgres with auth, storage and row-level policies (Supabase) · hosted identity provider (Clerk) · Stripe · Vercel | `runtime-tracks/ts-node.md` | Deepest ecosystem, one language end to end, and every piece has a first-party adapter for the others. The default until something in the brief argues otherwise. |
| **Lean SaaS** | same frontend · platform auth *instead of* a separate identity provider · merchant-of-record billing · Vercel | `runtime-tracks/ts-node.md` | Two fewer services and two fewer webhook syncs. The MoR owns sales tax, which is worth more than it looks to a solo founder. |
| **API-first** | portable HTTP microframework · typed SQL-shaped ORM · serverless Postgres (Neon) · container PaaS (Railway, Render, Fly) | `runtime-tracks/ts-node.md` | No UI framework tax. The microframework runs unchanged on Node, Bun, and Workers, so the host stays a reversible decision. |
| **Content / marketing** | islands static-site framework · headless CMS (Sanity, or git-committed MDX) · CDN-first host (Cloudflare Pages, Netlify) | `runtime-tracks/ts-node.md` | Ships zero JS by default, so Core Web Vitals are a starting condition rather than a project. |
| **Mobile + backend** | managed React Native toolchain · the toolchain's cloud build and OTA service (EAS) · hosted BaaS or your own API | `runtime-tracks/mobile-native.md` (app) + `runtime-tracks/ts-node.md` (backend) | Signing, store submission, and over-the-air updates are the hard parts, and the managed toolchain owns all three. |
| **Internal tool** | full-stack React framework · copy-in component library · full-featured ORM · Postgres · Vercel or a container PaaS | `runtime-tracks/ts-node.md` | The copy-in library already has the table, form, and dialog primitives an admin UI is 90% made of. The heavier ORM wins here because DX beats query control on a low-traffic tool. |
| **Data / analytics** | Python orchestration · columnar warehouse · a dashboard on the default web track | `runtime-tracks/python.md` (+ `ts-node.md` for the UI) | Transformation belongs where the numeric ecosystem is. The dashboard is a separate deliverable with a separate track — do not force one runtime across both. |
| **Server-rendered CRUD** | batteries-included server-rendered framework · its own ORM · its own deploy tool · a long-lived VM or container | `runtime-tracks/rails-laravel.md` | Admin, auth, jobs, and mailers ship in the box. Fastest path when the product is forms over a database and the team already knows the framework. |

## Known-bad combinations

Each row states the conflict in terms that stay true across releases.

| Conflict | What actually breaks | Do instead |
|---|---|---|
| Linter/formatter that parses CSS itself **+** CSS-first styling engine with custom at-rules | The linter has its own CSS parser and rejects the styling engine's at-rules (`@theme`, `@utility`, `@variant`, `@apply`) as unknown syntax. This is a **parse** error, not a lint rule, so turning rules off does not silence it and `--write` cannot fix it. It fires on the stylesheet the *scaffolder itself generated*, so the very first `lint` run of step 1 exits non-zero before a line of product code exists. It hides easily: the scaffolder often pins an older linter major that still parses the file, and the break only appears when you upgrade to the track's pinned version. Both halves are mandated by the same default track, so a stack built from the track alone always contains it. | Enable the linter's custom-at-rule parser option in the linter config *before* the first `lint` run, and treat it as part of scaffolding rather than a fix. On the TypeScript track the key is `css.parser.tailwindDirectives: true` in `biome.json` — exact key, versions, and the surrounding scaffold order are in `knowledge/runtime-tracks/ts-node.md` Gotchas. Do not resolve it by ignoring the stylesheet or disabling CSS linting: that surrenders the whole styling layer to the linter's blind spot. |
| Utility-CSS framework **+** runtime CSS-in-JS | Two cascade and specificity models fighting over the same elements. Worse on a server-first React framework: a runtime CSS-in-JS library needs a client boundary, so every styled component silently opts out of server rendering. | Pick one. Utility CSS on the server-first track. If a design system already exists in CSS-in-JS, move it to a zero-runtime/compiled variant before porting. |
| Two identity providers (hosted provider **+** framework auth library) | Two session cookies, two user tables, and no answer to "who is the source of truth" when they disagree. Every downstream authorization check has to pick a side. | Exactly one owns the session. Keep the second only as a federated login *into* the first. See `knowledge/capabilities/auth.md`. |
| Long-lived process design **+** request-scoped host | A worker loop, an in-process scheduler, or an in-memory queue on a serverless host runs on your laptop and evaporates in production — no invocation, no execution. | Split the tiers: serverless web, container worker, one shared database. See `knowledge/capabilities/deployment.md`. |
| In-memory realtime state **+** horizontally scaled host | Rooms, presence, and counters in module scope are per-instance. Reconnects land on a different instance and a deploy splits the fleet across two versions. Not a serverless problem — any multi-instance deploy has it. | External pub/sub adapter plus an external state store (Redis). Multi-node Socket.IO additionally needs either sticky sessions or a WebSocket-only transport. |
| Raw-TCP database driver **+** a runtime without TCP sockets | Edge and Workers-style runtimes expose fetch, not sockets. The connection fails at runtime, never at build, so it ships green. | Use the platform's HTTP/WebSocket driver or the ORM's driver adapter for that runtime — and check the bundle-size ceiling while you are there. |
| Per-request serverless connections **+** un-pooled Postgres | Connection exhaustion under the first traffic spike. The symptom looks like a slow database and is not. | Connect through a pooler; reserve a direct URL for migrations and workers. Transaction-mode poolers drop prepared statements, advisory locks, and `LISTEN`/`NOTIFY` — configure the driver for it. |
| Document database **+** relational data | Foreign keys, joins, and multi-document invariants become application code that races. The bugs arrive as orphaned rows months later. | Postgres. Reach for a `jsonb` column when *part* of the payload is genuinely shapeless. See `knowledge/capabilities/database.md`. |
| SQL ORM **+** an SDK-only document platform (Firestore and friends) | There is no SQL wire protocol to talk to. Nothing about the pairing works, at any version. | Use the platform SDK, or move to a SQL database if you want an ORM. |
| Schema-first graph API **+** simple CRUD | A gateway, a schema, a resolver layer, and an N+1 batching story bolted onto six endpoints. All cost, no payoff. | REST or a typed RPC layer. Add a graph API when many clients genuinely need different shapes of the same object. |
| A second HTTP framework inside a full-stack framework | Two routers, two middleware chains, and one deploy target that only ever invokes one of them. Auth applied in the wrong chain is silently skipped. | Use the framework's own route handlers. If you need a standalone server, make it a separate deployable with its own track. |
| Global store library with reducer boilerplate **+** a small app | Ceremony per feature and a store that duplicates what the server cache already holds. | Server state in a query cache, local state in component state, one small global store only if something genuinely spans routes. See `knowledge/capabilities/state-management.md`. |
| Custom native modules **+** a prebuilt generic client app | The store-distributed client can only load native code compiled into it. Adding a native dependency and reloading in the generic client fails with a confusing JS-level error. | Move to a development build the moment the first native module lands. Every native dependency also constrains the SDK upgrade path — check support before installing. See `knowledge/shapes/mobile-app.md`. |
| Extension platform **+** a dev pipeline that emits `eval` or remote code | Extension CSP forbids remotely hosted code and eval-based module wrappers, so HMR dev builds are rejected while the production build passes. The failure appears only when loading the unpacked extension. | Build to static files and reload the extension; keep HMR for a standalone dev harness page. See `knowledge/shapes/browser-extension.md`. |
| Admin/UI framework **+** a different major of the component runtime it is built on | The peer constraint either blocks the install or resolves to a half-working pair where some components render and some silently do not. | Upgrade both together, driven by the admin framework's supported matrix. Pins live in `knowledge/runtime-tracks/rails-laravel.md`. |
| Two migration systems on one database | The platform's dashboard schema editor plus your ORM's migrations. State drifts, and the next migration fails in production only. | One owns the schema. The dashboard is read-only from day one. |
| Any Google-hosted runtime dependency **+** mainland China users | Fonts, reCAPTCHA, Maps, Analytics and Firebase — including FCM push — do not resolve from inside the border. Nothing fails at build, nothing fails in foreign CI, and nothing fails for the developer who wrote it: the page renders blank or the notification channel is simply dead for every mainland user. | Self-host fonts, use a domestic map and captcha vendor, and route Android push through per-vendor channels. See `knowledge/capabilities/china-mainland.md`. |
| A hardcoded package-registry mirror **+** CI running outside mainland China | A mirror pinned in the lockfile, `.npmrc`, `pip.conf` or a Dockerfile makes the project build in Shanghai and fail everywhere else — and the usual fix lands as a revert, which then breaks the mainland developer again. Both failure directions are real, so neither default is correct. | Select the mirror from one environment variable and run both paths in CI. See `knowledge/capabilities/china-mainland.md`. |
| Self-hosted search engine **+** a static or edge-only host | The engine needs a persistent process and a disk. There is nowhere for it to run next to the site. | A hosted search service, or a container host alongside the site. See `knowledge/shapes/content-community-platform.md`. |

## Rules that expired

Retired here so nobody re-adds them from memory. Each was checked against the vendor's own docs on
2026-07-27.

| Retired rule | What is actually true | Source |
|---|---|---|
| "The full-featured ORM cannot run on Cloudflare Workers — use the SQL-shaped one" | It runs on Workers through driver adapters — Prisma Postgres, Neon, PlanetScale, D1, Turso, and plain Postgres with the Node-compatibility flag. The real constraints are *which driver* the runtime allows and the Workers bundle-size ceiling on the free plan, not the ORM. | `prisma.io/docs` — deploy to Cloudflare |
| "WebSockets cannot work on a serverless host — it is serverless" | Vercel Functions serve WebSocket connections on Fluid compute; both a bare WebSocket server and Socket.IO work. The durable constraint is **state, not transport**: a connection pins to one instance, and reconnects are not guaranteed the same instance — so rooms, presence, and counters go in an external store. That constraint is in the known-bad table above, correctly scoped. | `vercel.com/docs/functions/websockets` |
| "Previous-major config patterns inside the current major" (the utility-CSS framework) | A version-transition trap, not a cross-axis conflict — it stops being true one major later, and it is invisible to a role-based rule. It belongs in the runtime track's Gotchas, where `knowledge/runtime-tracks/ts-node.md` already documents it (JS config file vs. CSS-first config). | this repo |

## Auth-to-database pairings

| Auth model | Database partner | Why |
|---|---|---|
| Hosted identity provider (Clerk, Auth0, WorkOS) | Any — the provider is independent | Identity lives with the provider; you keep a local user row keyed by the provider's id and sync it by webhook. Never join against the provider's API in a request path. |
| Platform auth bundled with the database (Supabase, Firebase) | That platform's own database | The session is already in the row-level policy context. Tightest integration available, and the reason to accept the platform's gravity — leaving means reimplementing authorization. |
| Self-hosted auth library inside the app | Any database your ORM supports | You own the schema and the session table. The library's adapter must match the ORM you already picked — check that pairing before committing to either. |
| Enterprise SSO / SCIM broker | Any | Directory sync writes into your own users and orgs tables. Design those tables to accept an external source of truth from day one. See `knowledge/capabilities/enterprise-readiness.md`. |

## Hosting-to-framework pairings

| Framework role | Host | Why |
|---|---|---|
| Full-stack React framework | Its vendor's platform (Vercel) first; a Node container when vendor coupling is unacceptable | Every framework feature ships on the vendor platform first. Self-hosting works but you re-own routing, caching, and image handling. |
| Islands / static-site framework | CDN-first host (Cloudflare Pages, Netlify) | Static output by default. Add an SSR adapter only when a page genuinely cannot be built ahead of time. |
| Portable HTTP microframework | Container PaaS (Railway, Render, Fly), or an edge runtime **if** the code imports no Node built-ins | Portability is the whole reason to pick this role. One Node-only import in shared handler code forfeits it. |
| Batteries-included server-rendered framework | A long-lived VM or container, deployed by the framework's own tool | The framework assumes a persistent process, a local filesystem, and in-process background jobs. |
| React Native app | The toolchain's cloud build and OTA service (EAS) | Credentials, signing, store submission, and staged rollout are the hard parts, and they live there. |
| Python pipeline or worker | Container PaaS or a managed orchestrator | Long runtimes and heavy dependencies. A request-scoped host is the wrong shape entirely. |

## ORM-to-database pairings

| ORM role | Fits | Notes |
|---|---|---|
| Typed SQL-shaped ORM | Postgres, MySQL, SQLite/libSQL, D1 | Thin runtime, edge-friendly, schema file is the source of truth and migrations are diffed from it. The default on the TypeScript track. |
| Full-featured ORM | Postgres, MySQL, SQLite — and edge runtimes via driver adapters | Generated client, mature migrations, a GUI. Heavier client, so check the bundle ceiling before putting it in an edge function. |
| Query builder | Any SQL database | Type-safe with almost no abstraction. Bring your own migration runner — that is the trade. |
| Document ODM | That document database only | Not portable. Choosing the ODM is choosing the database. |
| Platform SDK | That platform only | Fine for prototypes and client-side reads. Access rules live outside your codebase and are easy to leave untested. |

## Checks a script can run

Each guard is decidable locally, with no network, no reviewer, and no deploy. Wire them into the
build as a lint step; each exits non-zero on a hit.

| Guard | Check |
|---|---|
| Two identity providers | The dependency manifest lists more than one auth SDK or auth library. |
| Two styling paradigms | The manifest lists both a utility-CSS framework and a runtime CSS-in-JS library. |
| CSS-first at-rules the linter cannot parse | The manifest lists both a CSS-parsing linter and a CSS-first styling engine, and the linter config does not enable its custom-at-rule parser option. Cheaper equivalent: run the lint command against the generated global stylesheet on a clean scaffold and require exit 0 — a blueprint whose step 1 was never executed once cannot claim this. |
| Raw-TCP driver on an edge target | Resolve the import graph of the edge entrypoint; it must not reach a socket-based database driver. Static resolution only — no deploy needed. |
| Two migration systems | The repo contains both an ORM migrations directory and checked-in platform schema exports. |
| In-memory realtime state | The realtime handler module declares no module-scope mutable collection. |
| Un-pooled serverless database URL | The runtime connection string resolves to the direct host rather than the pooler host; the direct URL appears only in the migration and worker configuration. |
| Unpinned toolchain | The manifest pins a package manager and the repo carries a runtime version file. |

## See also

- `knowledge/runtime-tracks/ts-node.md` — what every role above resolves to, with pins and gotchas
- `knowledge/runtime-tracks/mobile-native.md` — native module and SDK constraints behind the mobile row
- `knowledge/runtime-tracks/rails-laravel.md` — supported majors for the admin/component-runtime pairing
- `knowledge/capabilities/database.md` — pooling, migrations, and the ORM decision in full
- `knowledge/capabilities/deployment.md` — host choice, background jobs, and the serverless boundary
- `knowledge/capabilities/auth.md` — session ownership and the provider-mirror pattern
- `knowledge/capabilities/styling.md` — the styling paradigm decision this file only rules on
- `knowledge/shapes/saas-webapp.md` — the shape the default combination targets
