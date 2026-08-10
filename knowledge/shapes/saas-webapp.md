# Shape: SaaS Web Application

> A multi-user web app where people sign up, log in, and manage something that belongs to them — the default web shape, and the one most briefs collapse into.

Last verified: 2026-07-27

## Is this your project?

**Yes if:**
- Users create an account and come back to data that is theirs (notes, tasks, clients, projects, invoices).
- There is a logged-out marketing surface and a logged-in app surface on the same domain.
- Someone will eventually pay a recurring fee, even if the first release is free.
- The brief says "dashboard", "workspace", "my account", "team", "plan", or "trial".
- It is a productivity / notes / task / tracker app — zero architectural distinctness, so it lands here.
- **Booking, scheduling, or appointments** are the object users manage — slots, availability, calendars, reschedules, no-shows. Stay here and add `knowledge/capabilities/availability-engine.md`; the tenancy, billing, and CRUD spine below is unchanged, the calendar is a capability on top of it.

**No if:** — this shape is the default sink, so walk every exit before letting a brief land here. Each line is the one signal that beats "generic web app".

- Nobody logs in and the goal is conversion → `knowledge/shapes/marketing-site.md`.
- Users are employees on a company SSO, no signup and no billing → `knowledge/shapes/internal-tool.md`.
- The deliverable is endpoints another client consumes, no UI in scope → `knowledge/shapes/api-backend.md`.
- The core loop is a model reasoning and calling tools — remove it and nothing is left → `knowledge/shapes/agent-app.md`.
- The primary transaction is a cart over a catalog, with stock or fulfilment → `knowledge/shapes/ecommerce-storefront.md`.
- The value is other users' content, feeds, follows, and moderation → `knowledge/shapes/content-community-platform.md`.
- It has to be an icon on a home screen — camera, GPS, push, biometrics, or offline writes → `knowledge/shapes/mobile-app.md`.
- It installs on a desktop OS and owns local files, a tray item, or a global hotkey → `knowledge/shapes/desktop-app.md`.
- The value only exists on top of pages the user already visits → `knowledge/shapes/browser-extension.md`.
- The artifact is installed or imported by developers — a CLI, a package, an MCP server → `knowledge/shapes/cli-library-mcp.md`.
- The brief opens with a trigger ("when someone posts…", "every morning at 7") and the UI is at most a status page → `knowledge/shapes/automation-bot-integration.md`.
- The work is scheduled extraction, modelling, and freshness over volumes of data; the screens come after → `knowledge/shapes/data-pipeline-analytics.md`.
- Each action produces an image, video, audio clip, or mesh at a real per-generation provider cost → `knowledge/shapes/generative-media-app.md`.

## Default runtime track

**TypeScript / Node** — see `knowledge/runtime-tracks/ts-node.md`. One language across UI, server actions, and background jobs; the deepest ecosystem for auth, billing, and hosting glue, so you write product code instead of plumbing.

Alternatives: `knowledge/runtime-tracks/rails-laravel.md` when the team wants batteries included and server-rendered-first — scaffolding, migrations, jobs, mailers, and an admin in the box, ideal for CRUD-heavy B2B where nobody wants to assemble a stack. `knowledge/runtime-tracks/python.md` when the differentiator is data science or ML that would otherwise need a second service.

## The first architectural fork: tenancy

Decide in Phase 2, before any table exists. It changes every access policy, every query, and every URL.

| | Single-tenant (per-user) | Multi-tenant (org / workspace) |
|---|---|---|
| Owner of data | `user_id` on every row | `org_id` on every row; users join orgs |
| Access check | "is this row mine?" | "am I a member of this org, and what role?" |
| URLs | `/dashboard/projects/:id` | `/:orgSlug/projects/:id` |
| Billing subject | the user | the org, with seats |
| Choose when | solo/prosumer tool, B2C | anything sold to a company, or "invite a teammate" appears anywhere in the brief |

Retrofitting orgs onto a single-tenant schema is a full rewrite of the data layer. If there is *any* chance of teams, start multi-tenant with an auto-created personal org per signup — the cost is one extra table and one extra column.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|
| Auth | Signup, login, session, protected routes, password reset | `knowledge/capabilities/auth.md` |
| Database | Owns the entities; tenancy scoping lives here | `knowledge/capabilities/database.md` |
| Frontend architecture | Two shells (marketing, app), routing, fetch boundaries, server vs client state | `knowledge/capabilities/frontend-architecture.md` · `knowledge/capabilities/state-management.md` |
| Styling | Design tokens, component library, dark mode | `knowledge/capabilities/styling.md` |
| API design | Server actions vs routes; validation at the boundary | `knowledge/capabilities/api-design.md` |
| Payments | Plans, checkout, webhooks, entitlement gating | `knowledge/capabilities/payments-rails.md` |
| Deployment | Environments, secrets, migrations on deploy | `knowledge/capabilities/deployment.md` |
| Testing | Auth, tenancy isolation, and checkout are the regression hot spots | `knowledge/capabilities/testing.md` |
| Observability | Errors and failed webhooks must page someone | `knowledge/capabilities/observability.md` |

Also pull in: **`knowledge/capabilities/china-mainland.md` the moment the users are inside mainland China** — it rewrites hosting, auth, payments and the model provider at once, and it adds filing gates measured in weeks, so it has to run before the stack table rather than after; `knowledge/capabilities/accessibility.md` (keyboard and contrast on the app shell, not just the landing page); `knowledge/capabilities/enterprise-readiness.md` when the brief mentions SSO, SAML, audit logs, or SOC 2; `knowledge/capabilities/sync-and-collab.md` when two people edit one record at once; `knowledge/capabilities/ai-llm-integration.md` for an AI feature inside an otherwise ordinary SaaS, plus `knowledge/capabilities/credit-metering.md` if it is metered; **`knowledge/capabilities/availability-engine.md` the moment the brief mentions booking, appointments, slots, availability, reschedules, or a shared calendar** — that is the single most-missed capability in this shape, because a booking brief reads like ordinary CRUD right up until timezones, double-booking, and cancellation policy arrive.

## Data model

| Entity | Key fields | Notes |
|---|---|---|
| `user` | id, email, name, avatar, created_at | Mirror of the auth provider's user; never the source of truth for credentials |
| `organization` | id, name, slug, plan, created_at | Multi-tenant only. Slug is URL-visible and immutable after first use |
| `membership` | user_id, org_id, role, invited_by, accepted_at | Join table. Roles: owner, admin, member. Owner is non-removable |
| `invitation` | org_id, email, role, token, expires_at | Multi-tenant only. Tokens expire and are single-use |
| `subscription` | org_id (or user_id), provider_customer_id, provider_subscription_id, status, plan, current_period_end | Written by webhook, never by the UI |
| `<core entity>` | id, owner scope (`org_id` or `user_id`), created_by, timestamps | What the product is actually about; plus its one-to-many child (items, tasks, line items) |

Rules: every tenant-owned table carries its scope column and an index on it; soft-delete the core entity (`deleted_at`) so undo is possible; store money as integer minor units; add an `audit_log` (actor, action, target, metadata) on day one when the buyer is a company.

## Directory structure

```
# shown for the TypeScript track; the rails-laravel track uses its own framework layout
src/
  app/
    (marketing)/          # public: landing, pricing, legal — no session required
    (app)/                # protected shell: sidebar + topbar
      [org]/              # multi-tenant only — tenant segment wraps every app route
        <feature>/        # dashboard, list, detail, edit for the core entity
      settings/           # profile, members, billing
    api/webhooks/         # billing + auth provider webhooks (raw body, signature-verified)
  components/             # ui/ primitives · app/ widgets, tables, forms · marketing/ hero, pricing
  server/
    db/                   # schema + migrations
    queries/              # read paths — every function takes the tenant scope
    mutations/            # write paths — validate, authorize, then write
    auth/                 # session helpers, role guards
    billing/              # plan map, checkout, entitlement checks
  lib/                    # pure utilities, no I/O
```

Hard boundary: nothing in `components/` imports from `server/db/`. Data enters components through `server/queries` only.

## Build order

1. **Scaffold + tooling** — create the project per the runtime track, wire linter, formatter, and CI. *Done when:* the track's lint, typecheck, and build commands all pass on a clean checkout in CI.
2. **Design tokens + app shell** — palette, type scale, spacing, dark mode; empty marketing layout and empty app layout with sidebar/topbar. *Done when:* a viewport test asserts `document.documentElement.scrollWidth === clientWidth` on both layouts at 375px and 1440px, and a test fetching the raw server HTML with a dark-mode cookie set finds the dark theme class already on the root element in the first response — before any client script runs.
3. **Database + tenancy decision** — model `user`, `organization`, `membership`, and the core entity with its scope column; run the first migration. *Done when:* migrations apply to an empty database and roll back cleanly; a seed script inserts two orgs with one user each.
4. **Auth** — signup, login, logout, password reset, session on the server, route protection. *Done when:* **WHEN** an anonymous request hits any `(app)` route **THE SYSTEM SHALL** redirect to sign-in and return to the original URL after login; signup auto-creates the personal org and its owner membership.
5. **Tenancy guard + isolation tests** — one authorization helper every query and mutation calls. *Done when:* an automated test signs in as org A and receives 404 (not 403, not the record) for an org B record id, across list, detail, update, and delete.
6. **Core entity CRUD** — create, list, detail, edit, soft-delete, end to end, one feature only. *Done when:* a fresh account can complete create → edit → delete from the UI, and each write is visible after a hard refresh.
7. **Dashboard list view** — search, filter, sort, pagination, empty state, loading skeleton. *Done when:* one test asserts the empty state at 0 rows, a single row at 1, and exactly one page of rows plus a next-page control at 200+; a second asserts that reloading a filtered URL returns the same row ids in the same order.
8. **Members + invitations** *(multi-tenant only)* — invite by email, accept flow, role changes, remove member. *Done when:* redeeming an invite token creates exactly one membership row and the second redemption of the same token returns an error creating none; a token past `expires_at` is rejected; and a `member` receives 403 on an owner-only action.
9. **Billing** — plan map, checkout, customer portal, webhook handler with signature verification and idempotency. *Done when:* `stripe trigger checkout.session.completed` writes exactly one `subscription` row; replaying the same event changes nothing; an unsigned request returns 400.
10. **Entitlement gating** — plan limits enforced server-side, with an upgrade prompt in the UI. *Done when:* a free-plan account is blocked from exceeding its limit by the server even when the request is sent directly to the API, and the UI shows the upgrade path instead of an error toast.
11. **Settings** — profile, org name/slug, billing management, account deletion with data export. *Done when:* deletion removes or anonymizes every row keyed to the account and the user cannot log back in.
12. **Marketing surface** — landing page, pricing, metadata, OG image, sitemap, robots. *Done when:* every public route returns 200 with a unique title and description, and the sitemap lists exactly the public routes.
13. **Hardening** — rate limits on auth and mutations, error boundaries, structured logging, error reporting. *Done when:* the 11th login attempt in a minute from one IP returns 429, and a deliberately thrown server error appears in the error reporter with a request id.
14. **Deploy** — staging and production environments, secrets, migrations on release, custom domain. *Done when:* a merge to the main branch deploys and migrates automatically, and the signup → create → checkout path is completed once on production.

## Pitfalls

- **Retrofitting orgs.** Deciding tenancy in week three means rewriting every query, policy, and URL. Decide at step 3.
- **Authorization scattered across handlers.** One missed scope check leaks another tenant's data. Route every read and write through a single guard, and test it (step 5) rather than trusting review.
- **Trusting the client for entitlements.** Hiding the button is not gating. Enforce limits on the server; the UI only explains them.
- **Webhooks written naively.** Verify the signature on the raw body, make the handler idempotent by event id, return 200 fast and do work after. Duplicate deliveries are normal, not an edge case.
- **Abstracting before the second feature exists.** Build feature one concretely, ship it, then extract. Premature base classes are the most common source of dead code.
- **N+1 on the dashboard.** The list view is the most-loaded page in the product. Count queries while building it, not after a customer complains.

## Skills for the build phase

Install commands and fallbacks: `knowledge/skills-registry.md`. No leading slash means the skill auto-activates; never invoke those with a slash. If a skill is absent, fall back to this knowledge base, note it in one line, and continue.

| Skill | When |
|---|---|
| `ui-ux-pro-max` | Step 2 — palette, type scale, component style for the app shell |
| `frontend-design` | Steps 6-7 and 12 — dashboard views and the landing page |
| `emil-design-eng` | Step 7 and 10 — transitions, optimistic states, upgrade prompts |
| `playwright-cli` | Steps 5, 9 — E2E for tenancy isolation, auth, and checkout |
| `/claude-seo-ai:audit` · `/humanizalo` | After step 12 — public-route SEO, then de-AI the copy |

## See also

- `knowledge/runtime-tracks/ts-node.md` — default track: pins, scaffolding commands, gotchas. `knowledge/runtime-tracks/rails-laravel.md` — the server-rendered, batteries-included alternative
- `knowledge/capabilities/payments-rails.md` — plans, checkout, webhook contract behind steps 9-10
- `knowledge/capabilities/availability-engine.md` — slots, timezones, double-booking, reschedule and no-show policy, whenever the brief books anything
- `knowledge/stack-compatibility.md` — combinations to avoid before committing the stack
- `knowledge/shapes/internal-tool.md` — when there is no signup and no billing
- `knowledge/shapes/data-pipeline-analytics.md` — when the reporting the customer keeps asking about is the actual product
