# Capability: Deployment

> Getting the thing off your laptop and keeping it there — host choice, pipeline, secrets, preview
> environments, rollback, domains, and the background work a request/response host cannot run.

Last verified: 2026-07-27

## When a project needs this

Every project. What changes is the *shape* of the answer. Listen for:

- "It has to be live at our domain" — custom domain, TLS, DNS cutover.
- "Clients review before it goes out" — preview deploys per pull request.
- "Some of this takes a few minutes" — the work does not belong in an HTTP handler.
- "It emails everyone every Monday" — scheduled work, retries, idempotency.
- "We're on AWS already" / "it can't leave our network" — the host is decided; design around it.
- "国内" / "备案" / a mainland cloud — read `knowledge/capabilities/china-mainland.md` before choosing a region. ICP filing gates the host answering for the domain at all, and it is measured in weeks.
- "Two people push at once and prod breaks" — you need a pipeline, not a deploy button.

## Decision matrix

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **Vercel** | Full-stack React framework apps, marketing sites | Zero-config for the framework it maintains, best-in-class preview deploys, edge routing | Function timeouts and bandwidth pricing bite at scale; strong gravity toward its own primitives |
| **Netlify / Cloudflare Pages** | Static and mostly-static sites | Fast global CDN, generous free tiers, simple redirects/headers files | Server-side story is thinner; Pages pushes you into Workers for anything dynamic |
| **Cloudflare Workers** | Edge APIs, high-volume low-CPU endpoints | Near-zero cold start, cheap at volume, D1/R2/KV/Queues in one bill | Not full Node — some libraries and native modules will not run; per-request CPU limits |
| **Railway / Render** | Long-lived containers, APIs, workers, cron | Docker in, URL out; managed Postgres and Redis alongside; no timeout ceiling | Single-region by default; less edge presence |
| **Fly.io** | Multi-region apps, persistent volumes, WebSockets | Runs close to users, real machines you can exec into, cheap scale-to-zero | More ops surface than a PaaS; you own more of the failure modes |
| **AWS / GCP (containers or functions)** | Enterprise, existing cloud commitment, compliance | Every primitive exists; residency and networking controls; committed-spend discounts | Highest setup and knowledge cost; easy to build something only one person understands |
| **VPS + Docker Compose** (Coolify, Dokku, plain systemd) | Cost floor, data sovereignty, hobby-to-small production | Predictable bill, no vendor limits, full control | You are the ops team: patching, backups, TLS renewal, uptime |
| **Managed BaaS host** (Supabase, Firebase, Convex) | Apps whose backend *is* the database | Auth, storage, and jobs bundled with the data layer | Backend logic constrained to the platform's execution model |

## Recommendation

**Default: deploy the frontend to the platform that maintains your framework, and the long-lived
backend to a container PaaS.** For a full-stack React framework app that is Vercel plus a managed
Postgres. For an API or worker fleet it is Railway or Render with a Dockerfile. This costs nothing
in flexibility early and buys you preview deploys, TLS, and a rollback button on day one.

Deviate when:

| Situation | Host instead |
|---|---|
| Sustained high request volume, thin per-request CPU | Cloudflare Workers — the cost curve is the whole argument |
| Persistent connections, in-memory state, multi-region latency targets | Fly.io |
| Existing cloud contract, VPC or residency requirements, procurement review | The cloud they already bought |
| Cost floor matters more than ops time; a single small server is genuinely enough | VPS + Docker, with backups automated before launch |
| GPU inference, video encode, headless browsers, anything over a few minutes | A dedicated worker host — never a serverless function |

Two rules that survive every host choice: **the build must run identically in CI and locally**, and
**one command redeploys any given revision**. If either is false, fix that before optimizing hosts.

## When a serverless platform is the wrong host

Serverless is excellent for request/response work with bounded runtime. It fails predictably outside
that box. Move the workload to a long-lived process when any of these are true:

| Signal | Why serverless fights you |
|---|---|
| Work exceeds the platform's function timeout | Hard ceiling; splitting the job into chained invocations is a distributed system you did not want |
| Realtime rooms, presence, or fan-out with in-memory state | Several serverless platforms now serve WebSockets, so transport is no longer the blocker — **state and instance pinning are**. A connection pins to one instance and a reconnect may land elsewhere, so rooms, presence and counters need an external store or a dedicated realtime service |
| In-memory cache, connection pool, or model kept warm | Every cold start rebuilds it; pooling against Postgres from many short-lived instances exhausts connections |
| Large files: video, image batches, big CSV imports | Ephemeral disk and payload limits; streaming through a function is expensive and fragile |
| GPU or heavy CPU | Not offered, or offered at a price that makes a rented machine look free |
| Steady, predictable, always-on load | Per-invocation pricing loses to a rented container above a surprisingly low threshold |

Hybrid is normal and correct: serverless for the web tier, a container for workers, one shared
database. Do not force a single execution model on the whole system.

## Durable execution and background jobs

The rule: **an HTTP handler validates, writes a record, enqueues, and returns.** Anything that can
take seconds, call a flaky third party, or need a retry belongs behind a queue.

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **Platform-native queue + cron** (Cloudflare Queues, cloud-provider queue services) | Staying inside one vendor | No extra bill or dashboard; already in your IAM story | Weak local development; orchestration logic ends up hand-rolled |
| **Inngest** | Event-driven steps in an existing serverless app | Steps are durable and individually retried; runs on top of your normal functions; strong local dev | Vendor holds your orchestration state; step semantics take a day to internalize |
| **Trigger.dev** | Long-running jobs you want to write as plain code | No timeout ceiling, real-time run inspection, self-hostable | Younger ecosystem; jobs execute on its infrastructure unless you self-host |
| **Temporal** | Multi-day, multi-service workflows where correctness is contractual | Deterministic replay, first-class versioning, the strongest durability guarantees | Heaviest concept load and the most operational surface of the four |
| **Queue library + your own workers** (BullMQ, Celery, Sidekiq, a Postgres-backed queue) | Teams already running containers | No new vendor, full control, cheapest at volume | You build retries, backoff, dead-lettering, and the dashboard yourself |

Recommendation: **Inngest if the app is already serverless, a queue library with your own workers if
you already run containers.** Reach for Temporal only when a workflow spans days and services and a
lost run is a real incident — its guarantees are worth its weight there and nowhere else.

Non-negotiables regardless of choice: every job is **idempotent** (same input twice, one effect),
retries use **exponential backoff with jitter**, exhausted retries land in a **dead-letter store**
that raises an alert, and every enqueue carries the **request id** that caused it so a trace connects
the two. See `knowledge/capabilities/observability.md`.

## CI/CD tiers

Pick the smallest tier that fits. Adding a stage later is easy; unwinding ceremony nobody needed is not.

**Tier 0 — Minimal (MVPs, solo builds, marketing sites)**
```
push to main   → build → deploy to production
open PR        → build + preview deploy
```
Preview deploys *are* your staging environment. A separate staging server at this stage is a second
thing to keep in sync for no benefit.

**Tier 1 — Standard (anything with paying users)**
```
open PR        → lint · typecheck · unit + integration tests · preview deploy
merge to main  → full suite · migrations · deploy to production · smoke test against the live URL
nightly        → E2E suite against production
```
Branch protection on `main`: green checks required, no direct pushes.

**Tier 2 — Enterprise (compliance, audited change control, multiple teams)**
```
open PR        → Tier 1 checks + dependency audit + secret scan + SAST + preview deploy
merge to main  → deploy to staging automatically
tag a release  → approval gate → canary or blue/green to production → automated rollback on error-rate breach
```
Add signed commits, an artifact registry with immutable digests, and a deploy log that records who
shipped what and when. Most teams asking for Tier 2 actually need Tier 1 plus an approval gate.

Keep total PR check time under about ten minutes. Past that, people stop waiting and start merging
around you. Cache dependencies, run jobs in parallel, and push slow suites to nightly.

## Environments and secrets

| Environment | Purpose | Data |
|---|---|---|
| Local | Development | Seeded fixtures; never a production dump |
| Preview (per PR) | Review and QA | Ephemeral branch database or a shared preview database — never production |
| Staging (Tier 1+ only) | Pre-production rehearsal | Anonymized copy of production shape |
| Production | Real users | Real data, restricted access, backups verified by restore |

Rules:

- **Parse and validate every environment variable at boot. Crash on missing.** A schema-validated env
  module is one file and prevents the most common production outage there is.
- **Never commit `.env`.** Commit `.env.example` with every key present and every value a placeholder.
- Client-exposed variables use the framework's public prefix and are treated as **published**. Nothing
  secret goes near that prefix — publishable keys only.
- Secrets live in the platform's secret store or a dedicated manager. If a secret has ever been in a
  git object, in a log line, or in a chat message, it is burned — rotate it.
- Every secret has a documented rotation procedure. Write it before you need it at 2am.
- Preview environments get **their own** third-party keys in test mode. A preview deploy must not be
  able to charge a real card or email a real customer.

## Preview deploys, rollback, and domains

**Preview deploys.** Every PR gets a URL. Wire the preview to test-mode keys and an isolated
database — branch databases where the provider offers them, otherwise a shared preview instance reset
on a schedule. Post the URL as a PR comment; review happens on the URL, not on the diff.

**Rollback.** The question is not "can we roll back" but "how long does it take and who can do it."

- Application code: keep the last known-good revision one command or one click away. Practice it once
  before launch so the first real rollback is not the rehearsal.
- Database: **migrations are forward-only in production.** Never plan to roll a migration back under
  load. Use expand → deploy → backfill → contract across separate deploys so old and new code both
  work against the intermediate schema. See `knowledge/capabilities/database.md`.
- Define the trigger in advance: error rate above X for Y minutes rolls back automatically or pages a
  human. A rollback policy invented during an incident is a slower rollback.

**Custom domains.** Apex plus `www`, one canonical and the other redirecting permanently. TLS is
automatic on every platform in the matrix — verify auto-renewal is on rather than assuming it.
Set DNS TTL low a day before a cutover and restore it after. Add SPF, DKIM, and DMARC when the app
sends mail, or your transactional email lands in spam and you will blame the wrong layer for a week.

## Data model additions

| Table | Holds |
|---|---|
| `jobs` | Job type, payload, status, attempt count, `run_after`, last error — even with a hosted job runner, a local row is what your UI queries |
| `job_dead_letters` | Payload and error of runs that exhausted retries, plus a replay marker |
| `idempotency_keys` | Key, scope, request fingerprint, stored response, expiry — so a retried enqueue does not double-execute |
| `deploys` (Tier 2) | Revision SHA, actor, environment, started/finished, outcome — the audit answer to "what changed" |

## Build steps this adds

Splice these into the host shape's build order.

1. **Env schema and boot validation** — one module parses every variable and exits non-zero on a
   missing one; `.env.example` lists all keys. · *Done when:* removing a required variable makes the
   app fail to start with a message naming that variable, and no `.env` file is tracked by git.
2. **Deploy the skeleton on day one** — before features, get a hello-world through the real pipeline
   to the real host. · *Done when:* the production URL serves the app over TLS and a health endpoint
   returns 200 with the build SHA.
3. **CI pipeline at Tier 0 or 1** — lint, typecheck, and the test suite run on every PR.
   · *Done when:* a PR that breaks a test cannot be merged, and a green PR produces a preview URL.
4. **Preview environment isolation** — test-mode third-party keys and a non-production database on
   previews. · *Done when:* a checkout on the preview URL creates a test-mode charge and zero rows in
   the production database.
5. **Migrations in the pipeline** — migrations run as a gated step before the new revision takes
   traffic. · *Done when:* a deliberately failing migration aborts the deploy with the previous
   revision still serving.
6. **Background job runner** — queue, worker process, backoff, dead-letter. · *Done when:* a job
   forced to fail its full retry budget lands in the dead-letter store and emits an alert-worthy log
   line carrying the originating request id.
7. **Idempotency for enqueued work** — the same key enqueued twice executes once. · *Done when:* a
   test enqueues the same key twice and asserts exactly one side effect.
8. **Rollback rehearsal** — deploy a revision, roll back, confirm recovery. · *Done when:* the
   previous revision is restored by one documented command in under five minutes, with the procedure
   written in the repo README.
9. **Domain and TLS cutover** — apex and `www` resolving, canonical redirect, certificate auto-renewing.
   · *Done when:* both hostnames serve over HTTPS and the non-canonical one returns a 301 to the
   canonical one.
10. **Deploy-time smoke test** — a handful of assertions run against the live URL after every
    production deploy. · *Done when:* a deploy whose smoke test fails marks the pipeline red and
    (Tier 2) triggers automatic rollback.

## Pitfalls

- **Deploying for the first time in week six.** The first deploy always surfaces something — a
  missing variable, a native dependency, a build that only worked on your machine. Pay that cost in
  hour one, not the night before launch.
- **Long work inside an HTTP handler.** It works locally, times out in production, and the user
  retries — now you have two half-finished runs. Enqueue instead.
- **Non-idempotent jobs.** Every queue retries, including ones you did not know retried. A job that
  is not safe to run twice will eventually charge someone twice.
- **Preview environments pointed at production.** One reviewer clicking around a preview should not
  be able to email your customer list. Isolate keys and data before you hand out the first URL.
- **Migrations shipped in the same deploy as the code that needs them.** Expand, deploy, backfill,
  contract — four deploys, never one.
- **Secrets in the build log.** Echoing environment during CI debugging publishes them to anyone with
  repo read access. Mask them, and rotate anything you printed.
- **Health checks that only prove the process is alive.** Check the database and queue connections
  too, or the load balancer will happily route to a broken instance.
- **Silent serverless cost.** A per-invocation bill with no alert becomes a five-figure surprise
  after one crawler discovers a slow endpoint. Set a budget alert the day you deploy — see
  `knowledge/capabilities/observability.md`.
- **A pipeline nobody trusts.** Flaky tests get skipped, then the whole gate gets bypassed. Quarantine
  a flaky test the same day it flakes; do not let it erode the pipeline.

## See also

- `knowledge/capabilities/observability.md` — logs, alerts, and the cost model that catches a runaway bill
- `knowledge/capabilities/testing.md` — what runs in the pipeline and what proves a deploy healthy
- `knowledge/capabilities/database.md` — migration strategy, connection pooling, branch databases
- `knowledge/runtime-tracks/ts-node.md` — build and start commands, containerization notes for this track
- `knowledge/runtime-tracks/python.md` — worker processes, packaging, and process managers for this track
- `knowledge/shapes/api-backend.md` — health checks, migration gating, zero-downtime restarts in context
- `knowledge/stack-compatibility.md` — host and runtime combinations that fight each other
