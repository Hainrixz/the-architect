<p align="center">
  <h1 align="center">The Architect</h1>
  <p align="center">
    <strong>A Claude Code meta-agent that designs complete software blueprints.</strong>
  </p>
  <p align="center">
    Describe what you want to build. Get a complete blueprint. Let Claude Code build it for you.
  </p>
  <p align="center">
    <a href="https://tododeia.com">tododeia.com</a> &middot;
    <a href="#english">English</a> &middot;
    <a href="#español">Español</a> &middot;
    <a href="#简体中文">简体中文</a>
  </p>
</p>

<p align="center">
  <a href="https://github.com/Hainrixz/the-architect/releases/latest"><img src="https://img.shields.io/github/v/release/Hainrixz/the-architect?sort=semver&style=for-the-badge&label=release&color=blueviolet" alt="Latest release" /></a>
  <img src="https://img.shields.io/badge/Claude_Code-Plugin-blueviolet?style=for-the-badge" alt="Claude Code Plugin" />
  <img src="https://img.shields.io/badge/Blueprints-Markdown-blue?style=for-the-badge" alt="Markdown Blueprints" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="MIT License" />
  <img src="https://img.shields.io/badge/by-tododeia.com-black?style=for-the-badge" alt="by tododeia.com" />
</p>

---

# English

## What is The Architect?

Imagine you want to build a house. Before anyone picks up a hammer, you need a **blueprint** — a detailed plan that shows every room, every wall, every pipe, and every wire. Without it, the builders wouldn't know what to do.

**The Architect does this for software.**

```
You describe it     →  "a SaaS for restaurant reservations, team accounts, Stripe"
The Architect designs it  →  interviews you, picks the stack, writes the blueprint
Claude Code builds it     →  reads the blueprint, ships the project step by step
```

It does **not** write application code. It designs systems and produces blueprints — self-contained markdown files that a *different* Claude Code instance, with zero prior context, can build from without asking you a single question.

---

## Install

### Plugin (recommended)

Two commands, inside any Claude Code session:

```
/plugin marketplace add alan8918/the-architect
/plugin install the-architect@alan8918
```

Then type `/architect` — **in any directory**. Your blueprints are written to `./blueprints/` in whatever folder you're working in, never inside the plugin.

### Clone (still works, exactly as in v1)

```bash
git clone https://github.com/Hainrixz/the-architect.git
cd the-architect
claude
```

Claude Code reads `CLAUDE.md` and becomes The Architect. Same interview, same gates, same output — blueprints land in `./blueprints/` inside the clone. The slash commands and subagents are plugin-only; clone mode runs the same flow conversationally.

**Prerequisites:** [Claude Code](https://claude.com/claude-code) and a Claude subscription. Nothing else.

### Pick your path: quick or full

There are two entry points, and the difference is real. Choose before you start.

| | [`/architect-quick`](commands/architect-quick.md) | [`/architect`](commands/architect.md) |
|---|---|---|
| **Questions** | 3, in one message | 12–16, across 6–7 messages |
| **How long, end to end** | ~10 minutes | ~40–60 minutes |
| **Use it when** | You already know the stack, or the build is small and you mostly want the plan written down | You intend to hand the result to an autonomous builder and walk away |
| **What you give up** | Smart defaults for everything you weren't asked | Nothing — but it costs you an hour |
| **What you keep** | Both gates, EARS acceptance criteria, verify commands, the confirmation gate | Same |

Quick mode is genuinely faster than v1's default — three questions, one message, defaults stated out loud so you can veto them. Full mode is the one to use when nobody will be around to answer the builder's questions later.

---

## How it works

Four phases. You talk, it designs, it generates.

| Phase | What happens | What you do |
|---|---|---|
| **1. Discovery** | 2–3 questions. Classifies your project into one of **14 shapes**, and asks first whether this is new code or existing code. | Answer |
| **2. Deep dive** | Shape-specific questions. Picks the runtime track and the capabilities. The `stack-researcher` subagent verifies every version against the live registries. | Answer 3–5 |
| **3. Architecture** | One dense message: stack table, how it fits together, what v1 includes and what it explicitly excludes, rough build phases. **Both gates run here.** | Confirm or adjust |
| **4. Generate** | Picks bundle or single file from the step count and says which. Then: `blueprint-writer` composes, `blueprint-validator` audits until it returns PASS, files are written to `./blueprints/`. | Wait |

### How long Phase 4 takes: 20–30 minutes, silently

This is the part nobody warns you about, so here it is up front. Once you confirm the architecture, generation runs **roughly 20–30 minutes for a bundle, 10–15 for a single file**, and produces no output until it is finished. That time is real work — a live registry call for every version pin, a full composition pass, and at least one validator round trip — but from your side it looks like nothing happening.

The Architect is required to tell you the estimate before it starts. If it doesn't, that's a bug. Go make coffee; what comes back is a file path and the first command to run.

### The two gates (new in v2)

Phase 4 does not run until both pass. Neither is optional.

**Gate A — zero `[NEEDS CLARIFICATION]` markers.** Before presenting anything, The Architect scans its own draft and emits a marker for every decision still underspecified — scope boundaries, delete semantics, who can see whose data, who owns the API keys. Each marker is closed one of three ways: you answer it, you confirm a stated default, or it becomes an explicit Non-Goal. Entering generation with an open marker is forbidden.

> The failure it prevents: a blueprint that *reads* as complete because the gaps were quietly filled with plausible guesses, and a builder agent that implements the guess at 2am with nobody to ask.

**Gate B — adversarial pre-mortem.** Eight angles aimed at killing the plan before it's generated: false assumptions, market, competition, viability, unit economics, execution, the six-months-out obituary, and the blind spot nobody in the conversation is looking at. The 3–7 findings that survive their own rebuttal become Risk Register entries or Non-Goals. If one invalidates the architecture, it goes back to redesign instead of shipping as a "risk". Uses `/abogado-del-diablo` when installed; runs inline otherwise — the gate is mandatory, only the tooling is optional.

---

## What you get

A blueprint with **20 fixed sections**. Section 9, the build order, is what the other 19 exist to support.

**Every build step carries four fields: `Do`, `Done when`, `Verify`, `Checkpoint`.** This is the anti-drift fix. In v1, steps had no definition of done — so an autonomous builder had no stopping condition, over-built, and declared victory on work that never ran.

Here's one real step, abridged from the worked example in [`templates/blueprint-template.md`](templates/blueprint-template.md):

````markdown
#### Step 7 — Stripe checkout and subscription webhook

**Do**
Wire paid signup end to end. Create:
- `src/lib/stripe.ts` — the SDK client, reading `STRIPE_SECRET_KEY`
- `src/app/api/checkout/route.ts` — creates a Checkout Session for the signed-in user
- `src/app/api/webhooks/stripe/route.ts` — signature-verified receiver, raw-body parsing
- `src/lib/billing/sync-subscription.ts` — the single writer to `subscriptions`
- migration `007_subscriptions.sql` — `subscriptions` + `webhook_events` (dedupe ledger)

**Done when**
- [ ] WHEN a POST arrives at `/api/webhooks/stripe` with an invalid `Stripe-Signature` header THE SYSTEM SHALL respond `400` and write zero rows to `subscriptions`.
- [ ] WHEN `checkout.session.completed` is received for a known customer THE SYSTEM SHALL upsert exactly one `subscriptions` row with `status='active'` and a non-null `current_period_end`.
- [ ] WHEN the same Stripe event `id` is delivered twice THE SYSTEM SHALL return `200` both times and leave the `subscriptions` row count unchanged.
- [ ] WHEN `STRIPE_WEBHOOK_SECRET` is unset at boot THE SYSTEM SHALL fail startup with a named error, not serve traffic that silently accepts unsigned payloads.

**Verify**
```bash
pnpm test src/app/api/webhooks/stripe          # expect: 6 passed, 0 skipped
pnpm typecheck                                  # expect: exit 0

stripe listen --forward-to localhost:3000/api/webhooks/stripe &
stripe trigger checkout.session.completed
psql "$DATABASE_URL" -c \
  "select status, count(*) from subscriptions group by status;"
# expect: active | 1

stripe trigger checkout.session.completed       # same fixture, replayed
psql "$DATABASE_URL" -c "select count(*) from subscriptions;"
# expect: 1  (idempotent — not 2)
```

**Checkpoint**
```bash
git add -A && git commit -m "step 7: stripe checkout + subscription webhook"
git tag step-07-billing
# rollback target if step 8 goes wrong: git reset --hard step-07-billing
```
````

Acceptance criteria use EARS form — **WHEN** `<trigger>` **THE SYSTEM SHALL** `<observable response>`. "It looks right", "billing works", "is wired up" are banned; the validator fails a blueprint that contains them. Every criterion must be decidable by a script, today, without leaving the machine. Anything that genuinely needs a human or a store review queue moves to a post-build launch checklist — written down, but not a build gate.

### Output layout

**The Architect picks the mode and tells you which, in one line.** It is derived from the step count — **12 steps or more gets a bundle, 11 or fewer gets a single file** — because packaging is a consequence of the design, not a question worth interrupting you for. Say so at any point and your preference wins instead. Both land under `./blueprints/` in your working directory, and both carry identical acceptance criteria and verify commands.

**Bundle** — for parallel builders, multi-week builds, or resumable state:

```
./blueprints/<project-slug>/
├── blueprint.md          # the 20-section narrative artifact
├── tasks.json            # the machine-readable task DAG
├── epics/
│   ├── 01-<name>.md
│   └── 02-<name>.md
└── workspace/            # copied INTO the target project root by the builder
    ├── CLAUDE.md
    ├── AGENTS.md
    └── .claude/
        ├── settings.json
        ├── skills/<name>/SKILL.md
        └── rules/<name>.md
```

`workspace/` exists so the builder copies **one directory** into the project root — `cp -R workspace/. <project-root>/` and the agent configuration is in place.

**Single file** — one builder, a build measured in days, nothing to resume:

```
./blueprints/<project-slug>-blueprint.md
```

Everything inline. One file to send, paste, or commit anywhere.

---

## The 14 shapes

v1 had 6 archetypes. v2 has 14 shapes, and they're stack-agnostic — a shape describes *what a thing is*, never what it's written in.

| Shape | What it covers | Default track | |
|---|---|---|---|
| [SaaS Web Application](knowledge/shapes/saas-webapp.md) | Sign up, log in, manage something that's yours. The default web shape. | TypeScript / Node | |
| [Marketing / Content Site](knowledge/shapes/marketing-site.md) | Landing pages, portfolios, docs. Content-first, near-zero client JS. | TypeScript / Node | |
| [Mobile App](knowledge/shapes/mobile-app.md) | App Store / Play Store, release trains, platform review. | Mobile native | |
| [API / Backend Service](knowledge/shapes/api-backend.md) | Headless, consumed over the network by other software or agents. | TypeScript / Node | |
| [Internal Tool / Admin Dashboard](knowledge/shapes/internal-tool.md) | CRUD and charts for a known authenticated team. Never public. | TypeScript / Node | |
| [Content & Community Platform](knowledge/shapes/content-community-platform.md) | Content plus identity plus a social graph. Publications, memberships, courses. | TypeScript / Node | *renamed* |
| [Agent App](knowledge/shapes/agent-app.md) | The model *is* the product. Prompt → tool → trace → eval, not CRUD. | TypeScript / Node | **new** |
| [Generative Media App](knowledge/shapes/generative-media-app.md) | Credit-metered async generation — headshots, ad reels, voice, music, 3D. | TypeScript / Node | **new** |
| [E-commerce Storefront](knowledge/shapes/ecommerce-storefront.md) | Browse, cart, checkout, pay, fulfil, return. | TypeScript / Node | **new** |
| [CLI / Library / MCP Server](knowledge/shapes/cli-library-mcp.md) | The consumer is a developer or an agent. An API surface plus a distribution channel. | Go | **new** |
| [Browser Extension](knowledge/shapes/browser-extension.md) | Lives inside the browser, augments pages the user already visits, ships through store review. | TypeScript / Node | **new** |
| [Desktop App](knowledge/shapes/desktop-app.md) | Signed, self-updating, owns its local data, touches the filesystem and OS permissions. | TypeScript / Node | **new** |
| [Automation / Bot / Integration](knowledge/shapes/automation-bot-integration.md) | A trigger fires, work runs, a result lands elsewhere — and it survives failure unattended. | TypeScript / Node | **new** |
| [Data Pipeline & Analytics](knowledge/shapes/data-pipeline-analytics.md) | Move data out, reshape it, put answers in front of a named human on a schedule. | Python | **new** |

Ambiguous brief? It names the two candidates, says which it would pick and why, and asks the one question that decides it.

---

## The 3-axis knowledge split

**This is the intellectual core of v2.** v1 hardcoded its stack into 13 separate files — every archetype carried its own table of frameworks and versions. Refreshing one library meant editing thirteen files, so nobody did, and the knowledge base went stale in four months. v2 separates three orthogonal questions so a refresh edits **one file**, and a subagent verifies the numbers live at design time anyway.

| Axis | Directory | Answers | Version pins? |
|---|---|---|---|
| **Shape** | [`knowledge/shapes/`](knowledge/shapes/) (14) | *What is it?* | **Never** |
| **Runtime track** | [`knowledge/runtime-tracks/`](knowledge/runtime-tracks/) (5) | *What is it written in?* | **Yes — only here** |
| **Capability** | [`knowledge/capabilities/`](knowledge/capabilities/) (19) | *What does it do?* | **Never** |

A shape says "the project's ORM" and links to its track. Only the track names a package and a number. And even the track is treated as a **cache, not a source of truth**: the `stack-researcher` report produced in *this* session outranks it, always. A stale cached pin never overrides a live registry check.

### The three subagents

| Agent | Job |
|---|---|
| [`stack-researcher`](agents/stack-researcher.md) | Resolves every version against authoritative registries before it's written. Flags prereleases, unmaintained packages, and anything it could not verify. **The anti-staleness fix.** |
| [`blueprint-writer`](agents/blueprint-writer.md) | Composes and writes the deliverable in isolated context, so a long generation doesn't flood the interview thread. |
| [`blueprint-validator`](agents/blueprint-validator.md) | Adversarially audits the finished blueprint and returns PASS or FAIL with line references. Nothing is presented to you until it passes. |

If a version cannot be verified, the blueprint says *"verify before install"* rather than guessing. An honest gap beats a wrong pin.

---

## Commands

| Command | What it does |
|---|---|
| [`/architect`](commands/architect.md) | Full interview — four phases, both gates, validator-gated output |
| [`/architect-quick`](commands/architect-quick.md) | Fast-track: three questions, smart defaults, **same** confirmation gate |
| [`/architect-brownfield`](commands/architect-brownfield.md) | Design a change against an existing codebase |
| [`/architect-next`](commands/architect-next.md) | Resume a build — reads `tasks.json`, prints the next unblocked task with its criteria and verify command |
| [`/architect-refresh`](commands/architect-refresh.md) | Re-verify every pin in an existing blueprint against live registries; report what moved and what breaks |
| [`/architect-audit`](commands/architect-audit.md) | Re-run the validator over an existing blueprint or bundle |

`/architect-next` is what lets a long build survive across sessions: a fresh context with no memory gets one question answered — *what do I do next, and how do I know when it's done?*

---

## Brownfield: it works on code that already exists

Most coding-agent work is not greenfield. `/architect-brownfield` reads a repo and emits a blueprint for a **change** — a feature, a refactor, an integration, or a migration.

It starts with a Phase 0 that doesn't exist in the greenfield flow: **map the repo before asking anything.**

| What it reads | Where |
|---|---|
| Runtime track | Package manifest + lockfile, language version files, container base image |
| Framework and topology | Entry points, routing directory, server/client split, workspace layout |
| Conventions | Naming, module boundaries, error handling, the linter config that's actually enforced |
| Data layer | Migrations, schema files, ORM usage sites |
| Test setup | Runner, location, naming, coverage floor |
| CI and deploy | Workflow files, deploy config, env var surface |
| Existing agent instructions | `CLAUDE.md` / `AGENTS.md` — **these outrank the plugin's defaults** |

Then it prints a Repo Map and asks you to correct anything it read wrong. Your correction is cheaper than its assumption.

Standing rules: it never proposes rewriting working code you didn't ask it to touch, and your repo's conventions beat this plugin's defaults. Migrations additionally get a parity-and-cutover section with a shadow-run diff, abort criteria, and a decommission plan.

---

## Companion skills

All optional. If one isn't installed, The Architect falls back to its own knowledge base or built-in `WebSearch`/`WebFetch`, says so in one line, and keeps going. It never blocks generation on a missing skill.

**A leading `/` means it really is a slash command. No slash means it auto-activates — writing it with a slash is a silent no-op.**

### Used *by* The Architect, during design

| Skill | What it adds | Install |
|---|---|---|
| `/last30days` | What people actually said about a stack or niche **this month** | `/plugin marketplace add mvanhorn/last30days-skill` |
| `ui-ux-pro-max` | The concrete visual system — palette hexes, type scale, component style | `/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill`<br>`/plugin install ui-ux-pro-max@ui-ux-pro-max-skill` |
| `emil-design-eng` | Motion and interaction — easing, duration budgets, enter/exit behavior | `npx skills@latest add emilkowalski/skills` |
| `agent-browser` | Reference-site analysis, any URL → clean markdown | `npm install -g agent-browser` |
| `browser-harness` | Escalation: drives your **real logged-in Chrome** when the reference site needs auth | Paste the setup prompt from [the repo README](https://github.com/browser-use/browser-harness) |
| `find-skills` | Discovers installable build-phase skills to name in the blueprint | `npx skills add vercel-labs/skills --skill find-skills -g` |
| `pdf` | Reads client-supplied spec PDFs, RFPs, brand guides during discovery | `/plugin marketplace add anthropics/skills`<br>`/plugin install document-skills@anthropic-agent-skills` |

### Recommended *in* the blueprint, for the builder

| Skill | Recommended for | Install |
|---|---|---|
| `frontend-design` | Any project with a UI | `/plugin marketplace add anthropics/skills`<br>`/plugin install example-skills@anthropic-agent-skills` |
| `playwright-cli` | E2E testing | `npm install -g @playwright/cli@latest`<br>`playwright-cli install --skills` |
| `/claude-seo-ai:audit` `:geo` `:fix` `:score` | Public-facing surfaces — classic SEO **and** being citable by AI answer engines | `/plugin marketplace add Hainrixz/claude-seo-ai`<br>`/plugin install claude-seo-ai@claude-seo-ai` |
| `/humanizalo` | Marketing copy and written content (EN/ES) | `git clone https://github.com/Hainrixz/humanizalo.git ~/.claude/skills/humanizalo` |

Every recommended skill goes into blueprint Section 18 **with its install command** — naming a skill the builder can't install breaks the self-contained promise.

Full registry with licenses, star counts, and fallbacks: [`knowledge/skills-registry.md`](knowledge/skills-registry.md).

---

## Project structure

```
the-architect/
├── .claude-plugin/
│   ├── plugin.json                # plugin manifest
│   └── marketplace.json           # marketplace: alan8918
├── CLAUDE.md                      # clone-mode entrypoint
├── skills/architect/SKILL.md      # plugin-mode entrypoint — the state machine
├── commands/                      # 6 slash commands
├── agents/                        # 3 subagents
├── knowledge/
│   ├── shapes/                    # 14 — what it is        (no version pins)
│   ├── runtime-tracks/            #  5 — what it's written in  (ONLY place pins live)
│   ├── capabilities/              # 19 — what it does       (no version pins)
│   ├── skills-registry.md         # skill names, install commands, fallbacks
│   └── stack-compatibility.md     # known-bad combinations, cross-axis
├── questions/                     # the single source for the interview flow
│   ├── phase-1-discovery.md
│   ├── phase-2-branches.md
│   ├── phase-3-confirmation.md
│   └── phase-4-generate.md
└── templates/
    ├── blueprint-template.md      # the 20-section output skeleton
    ├── claude-md-template.md      # CLAUDE.md for the target project
    ├── tasks-schema.md            # tasks.json field contract
    └── epic-template.md           # epic file format
```

**Runtime tracks:** `ts-node.md` · `python.md` · `go.md` · `rails-laravel.md` · `mobile-native.md`

**Capabilities:** `auth` · `database` · `deployment` · `api-design` · `frontend-architecture` · `testing` · `styling` · `state-management` · `ai-llm-integration` · `agent-loop` · `credit-metering` · `payments-rails` · `realtime-voice` · `sync-and-collab` · `availability-engine` · `enterprise-readiness` · `accessibility` · `observability` · `china-mainland`

`china-mainland` is the one capability read *before* the stack table rather than after: shipping to users inside mainland China rewrites hosting, identity, payments and the model provider together, and adds regulator-gated filings measured in weeks. See the [简体中文](#简体中文) section.

`questions/` is the single source for the interview. `CLAUDE.md` reads it by relative path; `skills/architect/SKILL.md` reads the same files via `${CLAUDE_PLUGIN_ROOT}/questions/…`. Nothing is duplicated.

---

## Upgrading from v1

Nothing breaks. Your v1 blueprints are still markdown and still build. But the repo moved.

| v1 | v2 |
|---|---|
| `knowledge/archetypes/` — 6 files, each with its own hardcoded stack table | `knowledge/shapes/` — 14 files, stack-agnostic, zero version pins |
| `knowledge/building-blocks/` — 8 decision guides | `knowledge/capabilities/` — 18 |
| Versions scattered across 13 files | `knowledge/runtime-tracks/` — 5 files, the only place a pin may appear |
| `archetypes/content-platform.md` | `shapes/content-community-platform.md` |
| Blueprints written to `output/` inside the repo | `./blueprints/` in **your** working directory |
| 16 sections | 20 sections — CLAUDE.md is now §19.1, not §15 |
| Build steps with no definition of done | `Do` + `Done when` + `Verify` + `Checkpoint` on every step |
| Clone only | Plugin **or** clone |
| No subagents, no slash commands | 3 subagents, 6 commands |

**What to actually do:**

1. **Install the plugin** (see [Install](#install)). You no longer need to `cd` into a clone to design something.
2. **Audit an old blueprint:** `/architect-audit path/to/old-blueprint.md`. It will **FAIL** — v1 blueprints have no acceptance criteria. That's expected, not a bug; the report tells you exactly which steps have no stopping condition.
3. **Un-rot the numbers:** `/architect-refresh path/to/old-blueprint.md` re-verifies every pin against the live registries and reports what moved, what breaks, and what to change. Add `--apply` to edit in place.
4. **Regenerate if it's worth it.** For anything you haven't started building, a fresh `/architect` run is faster than patching a v1 blueprint into v2 shape.

Custom archetypes you added under `knowledge/archetypes/`? Port them to `knowledge/shapes/` and strip the version numbers out into a runtime track. That's the whole migration.

---

## Contributing

Contributions welcome — open an [issue or PR](https://github.com/Hainrixz/the-architect/issues). The highest-value places to help:

- **New shapes** — a project type the 14 don't cover, in `knowledge/shapes/`
- **Capabilities** — sharper decision matrices in `knowledge/capabilities/`
- **Runtime tracks** — a new ecosystem, or a refresh of an existing track's pins
- **Translations** — the interview runs in the user's language; more coverage helps

Two rules for any PR touching `knowledge/`: **no version number outside `runtime-tracks/`**, and **every build step gets an observable "Done when".**

---

## License

MIT. See [LICENSE](LICENSE).

---

## Built by [tododeia.com](https://tododeia.com) · [@soyenriquerocha](https://github.com/Hainrixz)

---
---

# Español

## ¿Qué es The Architect?

Imagina que quieres construir una casa. Antes de que alguien agarre un martillo, necesitas un **plano** — un plan detallado que muestre cada cuarto, cada pared, cada tubería y cada cable. Sin él, los constructores no sabrían qué hacer.

**The Architect hace esto para software.**

```
Tú lo describes        →  "un SaaS de reservaciones de restaurante, cuentas de equipo, Stripe"
The Architect lo diseña →  te entrevista, elige el stack, escribe el blueprint
Claude Code lo construye →  lee el blueprint y arma el proyecto paso a paso
```

**No** escribe código de aplicación. Diseña sistemas y produce blueprints — archivos markdown autocontenidos desde los que *otra* instancia de Claude Code, sin contexto previo, puede construir sin hacerte una sola pregunta.

---

## Instalación

### Plugin (recomendado)

Dos comandos, dentro de cualquier sesión de Claude Code:

```
/plugin marketplace add alan8918/the-architect
/plugin install the-architect@alan8918
```

Luego escribe `/architect` — **en cualquier directorio**. Tus blueprints se escriben en `./blueprints/` de la carpeta donde estés trabajando, nunca dentro del plugin.

### Clon (sigue funcionando, igual que en v1)

```bash
git clone https://github.com/Hainrixz/the-architect.git
cd the-architect
claude
```

Claude Code lee `CLAUDE.md` y se convierte en The Architect. Misma entrevista, mismos gates, mismo resultado — los blueprints caen en `./blueprints/` dentro del clon. Los slash commands y los subagentes son exclusivos del plugin; en modo clon el flujo corre conversacionalmente.

**Prerequisitos:** [Claude Code](https://claude.com/claude-code) y una suscripción a Claude. Nada más.

### Elige tu camino: rápido o completo

Hay dos puntos de entrada y la diferencia es real. Elige antes de empezar.

| | [`/architect-quick`](commands/architect-quick.md) | [`/architect`](commands/architect.md) |
|---|---|---|
| **Preguntas** | 3, en un solo mensaje | 12–16, en 6–7 mensajes |
| **Cuánto tarda, de punta a punta** | ~10 minutos | ~40–60 minutos |
| **Úsalo cuando** | Ya sabes qué stack quieres, o el build es chico y solo quieres el plan por escrito | Vas a entregarle el resultado a un constructor autónomo y te vas a ir |
| **Qué sacrificas** | Defaults inteligentes en todo lo que no te preguntó | Nada — pero te cuesta una hora |
| **Qué conservas** | Los dos gates, criterios EARS, comandos de verificación, el gate de confirmación | Igual |

El modo rápido es de verdad más rápido que el default de v1 — tres preguntas, un mensaje, y los defaults dichos en voz alta para que puedas vetarlos. El modo completo es el que quieres cuando nadie va a estar ahí para responderle las dudas al constructor después.

---

## Cómo funciona

Cuatro fases. Tú hablas, él diseña, él genera.

| Fase | Qué pasa | Qué haces tú |
|---|---|---|
| **1. Descubrimiento** | 2–3 preguntas. Clasifica tu proyecto en uno de los **14 shapes**, y pregunta primero si es código nuevo o código existente. | Respondes |
| **2. Profundización** | Preguntas específicas del shape. Elige el runtime track y las capabilities. El subagente `stack-researcher` verifica cada versión contra los registros en vivo. | Respondes 3–5 |
| **3. Arquitectura** | Un solo mensaje denso: tabla de stack, cómo encaja todo, qué incluye v1 y qué excluye explícitamente, fases de construcción. **Aquí corren los dos gates.** | Confirmas o ajustas |
| **4. Generar** | Elige bundle o archivo único según el número de pasos y te dice cuál. Luego: `blueprint-writer` compone, `blueprint-validator` audita hasta dar PASS, los archivos se escriben en `./blueprints/`. | Esperas |

### Cuánto tarda la Fase 4: 20–30 minutos, en silencio

Esta es la parte que nadie te advierte, así que va por delante. Una vez que confirmas la arquitectura, la generación corre **unos 20–30 minutos para un bundle, 10–15 para un archivo único**, y no produce nada hasta terminar. Ese tiempo es trabajo real — una llamada en vivo al registry por cada versión, una pasada completa de composición, y al menos una vuelta del validador — pero desde tu lado se ve como si no pasara nada.

The Architect está obligado a darte el estimado antes de empezar. Si no lo hace, es un bug. Ve por un café; lo que regresa es una ruta de archivo y el primer comando que hay que correr.

### Los dos gates (nuevos en v2)

La Fase 4 no corre hasta que ambos pasen. Ninguno es opcional.

**Gate A — cero marcadores `[NEEDS CLARIFICATION]`.** Antes de presentar nada, The Architect revisa su propio borrador y emite un marcador por cada decisión aún subespecificada — límites de alcance, semántica de borrado, quién puede ver los datos de quién, quién es dueño de las API keys. Cada marcador se cierra de tres maneras: lo respondes, confirmas un default declarado, o se convierte en un Non-Goal explícito. Entrar a generación con un marcador abierto está prohibido.

> La falla que evita: un blueprint que *se lee* completo porque los huecos se rellenaron en silencio con suposiciones plausibles, y un agente constructor que implementa la suposición a las 2am sin nadie a quién preguntarle.

**Gate B — pre-mortem adversarial.** Ocho ángulos apuntados a matar el plan antes de generarlo: premisas falsas, mercado, competencia, viabilidad, economía unitaria, ejecución, el obituario a seis meses, y el punto ciego que nadie en la conversación está mirando. Los 3–7 hallazgos que sobreviven a su propia refutación se vuelven entradas del Risk Register o Non-Goals. Si alguno invalida la arquitectura, se rediseña en vez de enviarlo como "riesgo". Usa `/abogado-del-diablo` si está instalada; si no, corre inline — el gate es obligatorio, solo la herramienta es opcional.

---

## Qué obtienes

Un blueprint con **20 secciones fijas**. La Sección 9, el orden de construcción, es a lo que sirven las otras 19.

**Cada paso de construcción lleva cuatro campos: `Do`, `Done when`, `Verify`, `Checkpoint`.** Este es el arreglo contra la deriva. En v1 los pasos no tenían definición de terminado — así que un constructor autónomo no tenía condición de parada, sobre-construía, y cantaba victoria sobre trabajo que nunca corrió.

Aquí un paso real, abreviado del ejemplo trabajado en [`templates/blueprint-template.md`](templates/blueprint-template.md):

````markdown
#### Step 7 — Stripe checkout and subscription webhook

**Do**
Wire paid signup end to end. Create:
- `src/lib/stripe.ts` — the SDK client, reading `STRIPE_SECRET_KEY`
- `src/app/api/checkout/route.ts` — creates a Checkout Session for the signed-in user
- `src/app/api/webhooks/stripe/route.ts` — signature-verified receiver, raw-body parsing
- `src/lib/billing/sync-subscription.ts` — the single writer to `subscriptions`
- migration `007_subscriptions.sql` — `subscriptions` + `webhook_events` (dedupe ledger)

**Done when**
- [ ] WHEN a POST arrives at `/api/webhooks/stripe` with an invalid `Stripe-Signature` header THE SYSTEM SHALL respond `400` and write zero rows to `subscriptions`.
- [ ] WHEN `checkout.session.completed` is received for a known customer THE SYSTEM SHALL upsert exactly one `subscriptions` row with `status='active'` and a non-null `current_period_end`.
- [ ] WHEN the same Stripe event `id` is delivered twice THE SYSTEM SHALL return `200` both times and leave the `subscriptions` row count unchanged.
- [ ] WHEN `STRIPE_WEBHOOK_SECRET` is unset at boot THE SYSTEM SHALL fail startup with a named error, not serve traffic that silently accepts unsigned payloads.

**Verify**
```bash
pnpm test src/app/api/webhooks/stripe          # expect: 6 passed, 0 skipped
pnpm typecheck                                  # expect: exit 0

stripe listen --forward-to localhost:3000/api/webhooks/stripe &
stripe trigger checkout.session.completed
psql "$DATABASE_URL" -c \
  "select status, count(*) from subscriptions group by status;"
# expect: active | 1

stripe trigger checkout.session.completed       # same fixture, replayed
psql "$DATABASE_URL" -c "select count(*) from subscriptions;"
# expect: 1  (idempotent — not 2)
```

**Checkpoint**
```bash
git add -A && git commit -m "step 7: stripe checkout + subscription webhook"
git tag step-07-billing
# rollback target if step 8 goes wrong: git reset --hard step-07-billing
```
````

Los criterios de aceptación usan forma EARS — **WHEN** `<disparador>` **THE SYSTEM SHALL** `<respuesta observable>`. "Se ve bien", "el cobro funciona", "quedó conectado" están prohibidos; el validador reprueba un blueprint que los contenga. Cada criterio tiene que poder decidirlo un script, hoy, sin salir de la máquina. Lo que de verdad necesita a un humano o una cola de revisión de tienda se va a un checklist de lanzamiento post-build — queda escrito, pero no es un gate de construcción.

### Formato de salida

**The Architect elige el modo y te dice cuál, en una línea.** Lo deriva del número de pasos — **12 pasos o más va en bundle, 11 o menos en archivo único** — porque el empaquetado es consecuencia del diseño, no una pregunta que valga la pena interrumpirte. Dilo en cualquier momento y tu preferencia gana. Ambos caen bajo `./blueprints/` en tu directorio de trabajo, y ambos llevan exactamente los mismos criterios de aceptación y comandos de verificación.

**Bundle** — para constructores en paralelo, builds de semanas, o estado reanudable:

```
./blueprints/<project-slug>/
├── blueprint.md          # el artefacto narrativo de 20 secciones
├── tasks.json            # el DAG de tareas legible por máquina
├── epics/
│   ├── 01-<name>.md
│   └── 02-<name>.md
└── workspace/            # el constructor lo copia DENTRO de la raíz del proyecto
    ├── CLAUDE.md
    ├── AGENTS.md
    └── .claude/
        ├── settings.json
        ├── skills/<name>/SKILL.md
        └── rules/<name>.md
```

`workspace/` existe para que el constructor copie **un solo directorio** a la raíz del proyecto — `cp -R workspace/. <project-root>/` y la configuración del agente ya está puesta.

**Archivo único** — un constructor, un build de días, nada que reanudar:

```
./blueprints/<project-slug>-blueprint.md
```

Todo inline. Un archivo para mandar, pegar o commitear donde sea.

---

## Los 14 shapes

v1 tenía 6 arquetipos. v2 tiene 14 shapes, y son agnósticos al stack — un shape describe *qué es una cosa*, nunca en qué está escrita.

| Shape | Qué cubre | Track por defecto | |
|---|---|---|---|
| [SaaS Web Application](knowledge/shapes/saas-webapp.md) | Te registras, entras, administras algo tuyo. El shape web por defecto. | TypeScript / Node | |
| [Marketing / Content Site](knowledge/shapes/marketing-site.md) | Landings, portafolios, docs. Contenido primero, casi cero JS en cliente. | TypeScript / Node | |
| [Mobile App](knowledge/shapes/mobile-app.md) | App Store / Play Store, trenes de release, revisión de plataforma. | Mobile native | |
| [API / Backend Service](knowledge/shapes/api-backend.md) | Sin UI propia, consumido por otro software o por agentes. | TypeScript / Node | |
| [Internal Tool / Admin Dashboard](knowledge/shapes/internal-tool.md) | CRUD y gráficas para un equipo autenticado y conocido. Nunca público. | TypeScript / Node | |
| [Content & Community Platform](knowledge/shapes/content-community-platform.md) | Contenido más identidad más grafo social. Publicaciones, membresías, cursos. | TypeScript / Node | *renombrado* |
| [Agent App](knowledge/shapes/agent-app.md) | El modelo *es* el producto. Prompt → tool → trace → eval, no CRUD. | TypeScript / Node | **nuevo** |
| [Generative Media App](knowledge/shapes/generative-media-app.md) | Generación asíncrona medida por créditos — headshots, reels, voz, música, 3D. | TypeScript / Node | **nuevo** |
| [E-commerce Storefront](knowledge/shapes/ecommerce-storefront.md) | Catálogo, carrito, checkout, pago, envío, devolución. | TypeScript / Node | **nuevo** |
| [CLI / Library / MCP Server](knowledge/shapes/cli-library-mcp.md) | El consumidor es un dev o un agente. Una superficie de API más un canal de distribución. | Go | **nuevo** |
| [Browser Extension](knowledge/shapes/browser-extension.md) | Vive dentro del navegador, aumenta páginas que el usuario ya visita, pasa por revisión de tienda. | TypeScript / Node | **nuevo** |
| [Desktop App](knowledge/shapes/desktop-app.md) | Firmada, se auto-actualiza, es dueña de sus datos locales, toca el filesystem y permisos del SO. | TypeScript / Node | **nuevo** |
| [Automation / Bot / Integration](knowledge/shapes/automation-bot-integration.md) | Se dispara un evento, corre el trabajo, el resultado aterriza en otro lado — y sobrevive fallos sin supervisión. | TypeScript / Node | **nuevo** |
| [Data Pipeline & Analytics](knowledge/shapes/data-pipeline-analytics.md) | Sacar datos, reformarlos, poner respuestas frente a un humano con nombre, en horario. | Python | **nuevo** |

¿Brief ambiguo? Nombra los dos candidatos, dice cuál elegiría y por qué, y hace la única pregunta que lo decide.

---

## La división de conocimiento en 3 ejes

**Este es el núcleo intelectual de v2.** v1 tenía el stack hardcodeado en 13 archivos distintos — cada arquetipo cargaba su propia tabla de frameworks y versiones. Refrescar una librería significaba editar trece archivos, así que nadie lo hacía, y la base de conocimiento se puso rancia en cuatro meses. v2 separa tres preguntas ortogonales para que un refresh edite **un solo archivo**, y de todos modos un subagente verifica los números en vivo al momento de diseñar.

| Eje | Directorio | Responde | ¿Versiones? |
|---|---|---|---|
| **Shape** | [`knowledge/shapes/`](knowledge/shapes/) (14) | *¿Qué es?* | **Nunca** |
| **Runtime track** | [`knowledge/runtime-tracks/`](knowledge/runtime-tracks/) (5) | *¿En qué está escrito?* | **Sí — solo aquí** |
| **Capability** | [`knowledge/capabilities/`](knowledge/capabilities/) (19) | *¿Qué hace?* | **Nunca** |

Un shape dice "el ORM del proyecto" y enlaza a su track. Solo el track nombra un paquete y un número. Y hasta el track se trata como **caché, no como fuente de verdad**: el reporte de `stack-researcher` producido en *esta* sesión le gana, siempre. Un pin cacheado y viejo jamás sobrescribe una verificación en vivo del registro.

### Los tres subagentes

| Agente | Trabajo |
|---|---|
| [`stack-researcher`](agents/stack-researcher.md) | Resuelve cada versión contra registros autoritativos antes de que se escriba. Marca prereleases, paquetes sin mantenimiento, y todo lo que no pudo verificar. **El arreglo contra el desfase.** |
| [`blueprint-writer`](agents/blueprint-writer.md) | Compone y escribe el entregable en contexto aislado, para que una generación larga no inunde el hilo de la entrevista. |
| [`blueprint-validator`](agents/blueprint-validator.md) | Audita adversarialmente el blueprint terminado y devuelve PASS o FAIL con referencias de línea. No se te presenta nada hasta que pase. |

Si una versión no se puede verificar, el blueprint escribe *"verify before install"* en vez de adivinar. Un hueco honesto es mejor que un pin equivocado.

---

## Comandos

| Comando | Qué hace |
|---|---|
| [`/architect`](commands/architect.md) | Entrevista completa — cuatro fases, ambos gates, salida validada |
| [`/architect-quick`](commands/architect-quick.md) | Vía rápida: tres preguntas, defaults inteligentes, **el mismo** gate de confirmación |
| [`/architect-brownfield`](commands/architect-brownfield.md) | Diseña un cambio sobre un codebase existente |
| [`/architect-next`](commands/architect-next.md) | Reanuda un build — lee `tasks.json` y muestra la siguiente tarea desbloqueada con sus criterios y su comando de verificación |
| [`/architect-refresh`](commands/architect-refresh.md) | Reverifica cada pin de un blueprint existente contra los registros en vivo; reporta qué se movió y qué se rompe |
| [`/architect-audit`](commands/architect-audit.md) | Vuelve a correr el validador sobre un blueprint o bundle existente |

`/architect-next` es lo que permite que un build largo sobreviva entre sesiones: un contexto nuevo sin memoria obtiene la respuesta a una pregunta — *¿qué sigue, y cómo sé cuándo está terminado?*

---

## Brownfield: ahora funciona sobre código que ya existe

La mayoría del trabajo de un agente de código no es greenfield. `/architect-brownfield` lee un repo y emite un blueprint de **cambio** — una feature, un refactor, una integración o una migración.

Arranca con una Fase 0 que no existe en el flujo greenfield: **mapear el repo antes de preguntar nada.**

| Qué lee | Dónde |
|---|---|
| Runtime track | Manifiesto de paquetes + lockfile, archivos de versión de lenguaje, imagen base del contenedor |
| Framework y topología | Entry points, directorio de rutas, split servidor/cliente, layout del workspace |
| Convenciones | Nombres, límites de módulos, manejo de errores, la config del linter que de verdad se aplica |
| Capa de datos | Migraciones, archivos de esquema, sitios de uso del ORM |
| Setup de tests | Runner, ubicación, nombres, piso de cobertura |
| CI y deploy | Archivos de workflow, config de despliegue, superficie de variables de entorno |
| Instrucciones de agente existentes | `CLAUDE.md` / `AGENTS.md` — **estas le ganan a los defaults del plugin** |

Después imprime un Repo Map y te pide que corrijas lo que haya leído mal. Tu corrección sale más barata que su suposición.

Reglas permanentes: nunca propone reescribir código que funciona y que no pediste tocar, y las convenciones de tu repo le ganan a los defaults del plugin. Las migraciones además llevan una sección de paridad y cutover con diff de shadow-run, criterios de aborto y plan de decomiso.

---

## Skills complementarias

Todas opcionales. Si alguna no está instalada, The Architect usa su propia base de conocimiento o el `WebSearch`/`WebFetch` integrado, lo dice en una línea, y sigue. Nunca bloquea la generación por una skill ausente.

**Un `/` al inicio significa que sí es un slash command. Sin `/` significa que se auto-activa — escribirla con slash es un no-op silencioso.**

### Las usa The Architect, durante el diseño

| Skill | Qué aporta | Instalación |
|---|---|---|
| `/last30days` | Lo que realmente se dijo de un stack o nicho **este mes** | `/plugin marketplace add mvanhorn/last30days-skill` |
| `ui-ux-pro-max` | El sistema visual concreto — hexes de paleta, escala tipográfica, estilo de componentes | `/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill`<br>`/plugin install ui-ux-pro-max@ui-ux-pro-max-skill` |
| `emil-design-eng` | Movimiento e interacción — easing, presupuestos de duración, entrada/salida | `npx skills@latest add emilkowalski/skills` |
| `agent-browser` | Análisis de sitios de referencia, cualquier URL → markdown limpio | `npm install -g agent-browser` |
| `browser-harness` | Escalada: maneja tu **Chrome real con sesión iniciada** cuando el sitio pide login | Pega el prompt de setup del [README del repo](https://github.com/browser-use/browser-harness) |
| `find-skills` | Descubre skills instalables de fase de construcción para nombrarlas en el blueprint | `npx skills add vercel-labs/skills --skill find-skills -g` |
| `pdf` | Lee PDFs de especificación, RFPs y guías de marca durante el descubrimiento | `/plugin marketplace add anthropics/skills`<br>`/plugin install document-skills@anthropic-agent-skills` |

### Recomendadas en el blueprint, para quien construye

| Skill | Recomendada para | Instalación |
|---|---|---|
| `frontend-design` | Cualquier proyecto con UI | `/plugin marketplace add anthropics/skills`<br>`/plugin install example-skills@anthropic-agent-skills` |
| `playwright-cli` | Testing E2E | `npm install -g @playwright/cli@latest`<br>`playwright-cli install --skills` |
| `/claude-seo-ai:audit` `:geo` `:fix` `:score` | Superficies públicas — SEO clásico **y** ser citable por motores de respuesta IA | `/plugin marketplace add Hainrixz/claude-seo-ai`<br>`/plugin install claude-seo-ai@claude-seo-ai` |
| `/humanizalo` | Copy de marketing y contenido escrito (EN/ES) | `git clone https://github.com/Hainrixz/humanizalo.git ~/.claude/skills/humanizalo` |

Cada skill recomendada entra en la Sección 18 del blueprint **con su comando de instalación** — nombrar una skill que quien construye no puede instalar rompe la promesa de autocontención.

Registro completo con licencias, estrellas y respaldos: [`knowledge/skills-registry.md`](knowledge/skills-registry.md).

---

## Estructura del proyecto

```
the-architect/
├── .claude-plugin/
│   ├── plugin.json                # manifiesto del plugin
│   └── marketplace.json           # marketplace: alan8918
├── CLAUDE.md                      # punto de entrada en modo clon
├── skills/architect/SKILL.md      # punto de entrada en modo plugin — la máquina de estados
├── commands/                      # 6 slash commands
├── agents/                        # 3 subagentes
├── knowledge/
│   ├── shapes/                    # 14 — qué es              (sin versiones)
│   ├── runtime-tracks/            #  5 — en qué está escrito (ÚNICO lugar con versiones)
│   ├── capabilities/              # 19 — qué hace            (sin versiones)
│   ├── skills-registry.md         # nombres de skills, instalación, respaldos
│   └── stack-compatibility.md     # combinaciones conocidas como malas, entre ejes
├── questions/                     # la fuente única del flujo de entrevista
│   ├── phase-1-discovery.md
│   ├── phase-2-branches.md
│   ├── phase-3-confirmation.md
│   └── phase-4-generate.md
└── templates/
    ├── blueprint-template.md      # el esqueleto de 20 secciones
    ├── claude-md-template.md      # CLAUDE.md para el proyecto destino
    ├── tasks-schema.md            # contrato de campos de tasks.json
    └── epic-template.md           # formato de archivo de épica
```

**Runtime tracks:** `ts-node.md` · `python.md` · `go.md` · `rails-laravel.md` · `mobile-native.md`

**Capabilities:** `auth` · `database` · `deployment` · `api-design` · `frontend-architecture` · `testing` · `styling` · `state-management` · `ai-llm-integration` · `agent-loop` · `credit-metering` · `payments-rails` · `realtime-voice` · `sync-and-collab` · `availability-engine` · `enterprise-readiness` · `accessibility` · `observability` · `china-mainland`

`china-mainland` es la única capability que se lee *antes* de la tabla de stack: enviar a usuarios en China continental reescribe hosting, identidad, pagos y proveedor de modelo a la vez, y agrega trámites regulatorios medidos en semanas.

`questions/` es la fuente única de la entrevista. `CLAUDE.md` la lee por ruta relativa; `skills/architect/SKILL.md` lee los mismos archivos vía `${CLAUDE_PLUGIN_ROOT}/questions/…`. Nada se duplica.

---

## Migrar desde v1

No se rompe nada. Tus blueprints de v1 siguen siendo markdown y siguen sirviendo para construir. Pero el repo se movió.

| v1 | v2 |
|---|---|
| `knowledge/archetypes/` — 6 archivos, cada uno con su tabla de stack hardcodeada | `knowledge/shapes/` — 14 archivos, agnósticos al stack, cero versiones |
| `knowledge/building-blocks/` — 8 guías de decisión | `knowledge/capabilities/` — 18 |
| Versiones repartidas en 13 archivos | `knowledge/runtime-tracks/` — 5 archivos, el único lugar donde puede aparecer un pin |
| `archetypes/content-platform.md` | `shapes/content-community-platform.md` |
| Blueprints escritos en `output/` dentro del repo | `./blueprints/` en **tu** directorio de trabajo |
| 16 secciones | 20 secciones — CLAUDE.md ahora es §19.1, no §15 |
| Pasos de construcción sin definición de terminado | `Do` + `Done when` + `Verify` + `Checkpoint` en cada paso |
| Solo clon | Plugin **o** clon |
| Sin subagentes, sin slash commands | 3 subagentes, 6 comandos |

**Qué hacer en concreto:**

1. **Instala el plugin** (ver [Instalación](#instalación)). Ya no necesitas hacer `cd` a un clon para diseñar algo.
2. **Audita un blueprint viejo:** `/architect-audit ruta/al/blueprint-viejo.md`. Va a dar **FAIL** — los blueprints v1 no tienen criterios de aceptación. Eso es lo esperado, no un bug; el reporte te dice exactamente qué pasos no tienen condición de parada.
3. **Desoxida los números:** `/architect-refresh ruta/al/blueprint-viejo.md` reverifica cada pin contra los registros en vivo y reporta qué se movió, qué se rompe y qué cambiar. Agrega `--apply` para editar en el lugar.
4. **Regenera si vale la pena.** Para cualquier cosa que no hayas empezado a construir, correr `/architect` de nuevo es más rápido que parchar un blueprint v1 a forma v2.

¿Tenías arquetipos propios en `knowledge/archetypes/`? Pásalos a `knowledge/shapes/` y saca los números de versión a un runtime track. Esa es toda la migración.

---

## Contribuir

Las contribuciones son bienvenidas — abre un [issue o PR](https://github.com/Hainrixz/the-architect/issues). Dónde más ayuda:

- **Nuevos shapes** — un tipo de proyecto que los 14 no cubren, en `knowledge/shapes/`
- **Capabilities** — matrices de decisión más afiladas en `knowledge/capabilities/`
- **Runtime tracks** — un ecosistema nuevo, o un refresh de los pins de un track existente
- **Traducciones** — la entrevista corre en el idioma del usuario; más cobertura ayuda

Dos reglas para cualquier PR que toque `knowledge/`: **ningún número de versión fuera de `runtime-tracks/`**, y **cada paso de construcción lleva un "Done when" observable.**

---

## Licencia

MIT. Ver [LICENSE](LICENSE).

---

## Construido por [tododeia.com](https://tododeia.com) · [@soyenriquerocha](https://github.com/Hainrixz)

---
---

# 简体中文

## The Architect 是什么？

盖房子之前得先有**图纸**——每个房间、每堵墙、每根管线都画清楚，工人才知道怎么干。

**The Architect 就是给软件画这张图纸的工具。**

```
你描述           →  "一个餐厅订座 SaaS，要团队账号，接 Stripe"
The Architect 设计 →  访谈你、定技术栈、写出蓝图
Claude Code 施工   →  读蓝图，一步一步把项目做出来
```

它**不写业务代码**。它做系统设计，产出蓝图——一份自包含的 markdown，另一个毫无上下文的 Claude Code 实例照着它就能构建，中途一句都不用问你。

**全程支持简体中文。** 你用中文提第一句话，之后的访谈、架构方案、蓝图正文、生成的 `CLAUDE.md` 全部是中文；而路径、标识符、命令、`tasks.json` 的键与状态值保持英文——这条边界写在 [`knowledge/localization.md`](knowledge/localization.md) 里，是硬约束，不是风格偏好。

---

## 安装

### 插件方式（推荐）

> **本仓库是 fork。** marketplace 名为 `alan8918`，与上游 `soyenriquerocha` 区分开，
> 两个可以同时存在而不冲突。上游是 [Hainrixz/the-architect](https://github.com/Hainrixz/the-architect)，
> 简体中文支持与中国大陆能力域是本分支新增的。

在任意 Claude Code 会话里执行两条命令：

```
/plugin marketplace add alan8918/the-architect
/plugin install the-architect@alan8918
```

然后输入 `/architect`，**在任何目录下都可以**。蓝图写到你当前工作目录的 `./blueprints/`，绝不会写进插件目录。

### 直接说中文也能唤起

不打命令也行。说这些话技能会自动激活：

> 帮我设计架构 · 设计一下这个应用 · 我想做一个 App · 技术选型建议 · 用什么技术栈 ·
> 出一份技术方案 · 写个 PRD · 生成蓝图 · 规划这个项目 · 分析我现有的代码 · 给这个仓库加个功能

---

## 怎么运作

四个阶段。你说，它设计，它生成。

| 阶段 | 发生什么 | 你要做什么 |
|---|---|---|
| **1. 摸清需求** | 2–3 个问题。把项目归入 **14 种项目形态**之一，并先问清是新项目还是存量代码。 | 回答 |
| **2. 深入细节** | 针对该形态的追问。定运行时技术栈和各能力域。`stack-researcher` 子代理逐一到线上仓库核验版本号。 | 回答 3–5 个 |
| **3. 架构定稿** | 一条密集的消息：技术栈表格、各部分如何咬合、v1 做什么和明确不做什么、大致构建阶段。**两道门禁在这里执行。** | 确认或调整 |
| **4. 生成** | 按步骤数量决定出蓝图包还是单文件并告知你。然后 `blueprint-writer` 撰写，`blueprint-validator` 反复审计直到 PASS，文件落到 `./blueprints/`。 | 等待 |

### 第 4 阶段要 20–30 分钟，而且中途没有任何输出

这点必须提前讲清楚。你确认架构之后，生成过程**蓝图包约 20–30 分钟，单文件约 10–15 分钟**，完成前不会有任何中间输出。这段时间是真在干活——每个版本号都要发一次线上仓库请求、完整撰写一遍、至少跑一轮校验器往返——但从你这边看就像卡住了。

The Architect 有义务在开始前把预估时间告诉你。**如果它没说，那是 bug。** 去泡杯茶，回来拿到的是文件路径和第一条命令。

### 两道门禁

第 4 阶段在两道门禁都通过前不会启动，两道都不可跳过。

**门禁 A —— `[NEEDS CLARIFICATION]` 标记必须清零。** 呈现方案之前，它会扫描自己的草稿，对每一个仍然含糊的决策打标记——范围边界、删除语义、谁能看到谁的数据、API key 归谁管。每个标记只有三种关闭方式：你回答、你确认它给的默认值、或者它变成明确的非目标。带着未关闭的标记进入生成阶段是被禁止的。

> 它防的是这种情况：蓝图**读起来**很完整，因为空白处被似是而非的猜测悄悄填上了，然后构建代理凌晨两点照着猜测实现，没人可问。

**门禁 B —— 对抗式事前尸检。** 从八个角度尝试在生成前先把方案否掉：错误假设、市场、竞争、可行性、单位经济模型、执行、半年后的"讣告"、以及在场所有人都没看到的盲区。经得起自我反驳的 3–7 条结论，会变成风险登记项或非目标。如果某一条直接推翻了架构，那就回去重新设计，而不是挂个"风险"标签发出去。

---

## 你会拿到什么

一份 **20 个固定章节**的蓝图。第 9 节"构建顺序"是核心，另外 19 节都是为了支撑它。

**每个构建步骤带四个字段：`Do`、`Done when`、`Verify`、`Checkpoint`。** 这是防跑偏的关键。早期版本的步骤没有"完成的定义"，自主构建代理因此没有停止条件——过度构建，然后对着从没跑起来过的功能宣布胜利。

中文蓝图里的验收标准长这样（EARS 关键词保持英文大写，其余是中文）：

```markdown
**Done when**
- [ ] WHEN 一个带无效 `Stripe-Signature` 头的 POST 到达 `/api/webhooks/stripe`，
      THE SYSTEM SHALL 返回 `400`，且向 `subscriptions` 写入 0 行。
- [ ] WHEN 同一个 Stripe 事件 `id` 被投递两次，THE SYSTEM SHALL 两次都返回 `200`，
      且 `subscriptions` 的行数不变。

**Verify**
pnpm test src/app/api/webhooks/stripe          # 期望：6 passed, 0 skipped
psql "$DATABASE_URL" -c "select status, count(*) from subscriptions group by status;"
# 期望：active | 1
```

**"支付功能正常工作"这种写法是缺陷，不是验收标准。** 校验器会把它判为 BLOCKER——中英文一视同仁，中文模糊词黑名单（正常工作／正确处理／实现完成／符合预期／没问题……）写在 [`knowledge/localization.md`](knowledge/localization.md) 里。

---

## 中文支持的边界：什么翻译，什么绝不翻译

这是中文用户唯一需要记住的一张表。跨过这条线，失败是**静默的**——校验器报 PASS，构建却在第 3 步崩掉。

| 跟随你的语言（中文） | 永远保持 ASCII 英文 |
|---|---|
| 访谈对话、提问、建议 | 代码、标识符、函数名、类型名 |
| 章节标题的中文名 | 文件名、目录名、所有路径 |
| 每个"为什么"的理由 | 环境变量名、CLI 命令、包名 |
| 验收标准里可观测的那部分 | EARS 关键词 `WHEN` / `THE SYSTEM SHALL` |
| 非目标、风险、权衡、待决问题 | `tasks.json` 的键、`id`、`epic`、`files[]`、`verify[]` |
| 任务群叙述、阶段说明 | `status` 值 `pending` / `in_progress` / `done` |
| 生成的 `CLAUDE.md` 的正文部分 | git tag 名、分支名、commit 标题 |
| 待澄清标记**里面**的问题 | `[NEEDS CLARIFICATION: …]` 标记本身 |

几条实操约定：

- **编号标题保留编号和英文名**：`## 9. 构建顺序 (Build Order)`。校验器按编号匹配，英文名供交叉引用和 `/architect-audit` 使用。
- **目录名（slug）一律 ASCII**。有英文短名就用英文短名，没有就用无声调拼音：`智能巡检系统` → `zhineng-xunjian`。CJK 出现在 `git tag` 里，只会在构建到一半时在别人机器上炸掉。
- **文件是 UTF-8 无 BOM**。Windows 上生成的 `.env` 带 BOM，会让第 10 节 Bootstrap 的第一条命令读到乱码的 key 名。
- **繁體中文**：你用繁體，它就用繁體，不做简繁转换，也不在一份文档里混用。

---

## 14 种项目形态

形态描述的是**这东西是什么**，与用什么语言写无关。

| 形态 | 覆盖什么 | 默认技术栈 |
|---|---|---|
| [SaaS Web 应用](knowledge/shapes/saas-webapp.md) | 注册、登录、管理属于自己的东西。Web 的默认形态。 | TypeScript / Node |
| [营销／内容站](knowledge/shapes/marketing-site.md) | 落地页、作品集、文档站。内容优先，客户端 JS 趋近于零。 | TypeScript / Node |
| [移动 App](knowledge/shapes/mobile-app.md) | 应用商店、发版列车、平台审核。 | 移动原生 |
| [API／后端服务](knowledge/shapes/api-backend.md) | 无界面，被别的软件或代理通过网络消费。 | TypeScript / Node |
| [内部工具／管理后台](knowledge/shapes/internal-tool.md) | 给已知的已认证团队用的增删改查和图表。永不公开。 | TypeScript / Node |
| [内容与社区平台](knowledge/shapes/content-community-platform.md) | 内容 + 身份 + 社交图谱。刊物、会员、课程。 | TypeScript / Node |
| [Agent 应用](knowledge/shapes/agent-app.md) | 模型本身**就是**产品。提示词 → 工具 → 轨迹 → 评测，不是增删改查。 | TypeScript / Node |
| [生成式媒体应用](knowledge/shapes/generative-media-app.md) | 按额度计费的异步生成——写真、广告片、语音、音乐、3D。 | TypeScript / Node |
| [电商店面](knowledge/shapes/ecommerce-storefront.md) | 浏览、购物车、结算、支付、履约、退货。 | TypeScript / Node |
| [CLI／库／MCP 服务器](knowledge/shapes/cli-library-mcp.md) | 使用者是开发者或代理。一套 API 界面加一条分发渠道。 | Go |
| [浏览器扩展](knowledge/shapes/browser-extension.md) | 活在浏览器里，增强用户本来就会访问的页面，走商店审核。 | TypeScript / Node |
| [桌面应用](knowledge/shapes/desktop-app.md) | 签名、自更新、自己管本地数据、碰文件系统和系统权限。 | TypeScript / Node |
| [自动化／机器人／集成](knowledge/shapes/automation-bot-integration.md) | 触发器触发、任务执行、结果落到别处——并且无人值守时也能扛住失败。 | TypeScript / Node |
| [数据管道与分析](knowledge/shapes/data-pipeline-analytics.md) | 把数据取出来、重塑、按时把答案摆到某个具体的人面前。 | Python |

需求含糊时，它会点出两个候选形态、说明自己会选哪个和为什么，然后只问一个能决定归属的问题。

---

## 面向中国大陆的项目

用户在境内，架构就不是"把英文方案翻译一遍"那么简单。
[`knowledge/capabilities/china-mainland.md`](knowledge/capabilities/china-mainland.md) 是专门为此新增的能力域，
它会在**画技术栈表之前**被读到——因为它一次性改写四行：托管、身份认证、支付、模型提供方。

它覆盖的东西：

| 领域 | 关键结论 |
|---|---|
| **境内 vs 境外部署** | 决定一切的第一个岔路。给出三条路（境内／港新／拆分）的对照，并明确说拆分架构是陷阱：同时承担备案负担和跨境延迟 |
| **合规门禁** | 域名实名、ICP 备案、经营性许可证、公安网安备案、等保2.0 定级备案、生成式AI 备案／登记、数据出境——每条标注**谁需要、卡住什么、通常要多久** |
| **数据出境阈值** | 不满 10 万人免申报；10 万–100 万人或不满 1 万人敏感信息走标准合同／认证；100 万人以上或 1 万人以上敏感信息走安全评估（自当年 1 月 1 日累计，非关基运营者） |
| **身份认证** | 手机号+验证码为默认，微信授权为快路径。**跨端主键是 `unionid` 不是 `openid`**——这是数据模型 bug，不是偏好问题 |
| **支付** | 微信支付＋支付宝必须都上；`out_trade_no` 是你自己的幂等键；异步通知不是账本；自动续费是签约流程不是"卡里有卡" |
| **网络现实** | Google 全家桶（含 Firebase／FCM）境内不可达；镜像源必须做成**环境变量条件配置**，写死会让境外 CI 挂 |
| **地图坐标系** | GCJ-02 vs WGS-84。原始 GPS 打在国内底图上偏几百米——看起来像渲染 bug，实际是定位错误 |
| **国产模型** | 直连境外模型不是可靠的生产路径。单一网关接口是让"换供应商"变成改配置而不是重写的前提 |

**最重要的一条规则：备案不是构建步骤。**
它由监管方决定何时通过，属于校验器 finding 17 明令禁止的"依赖外部方的验收标准"——
写成"第 3 步：Done when 备案通过"，自主构建代理要么永远卡住，要么自己宣布通过。
它进的是蓝图 §20.1 的人工门禁清单，带责任人和日期。

这个能力域同时给出 9 条可直接并入构建顺序的步骤，每条都带可脚本判定的 `Done when`——
比如"同一条支付通知投递两次，系统应当两次都返回成功且已支付订单数不变"。

---

## 知识库的三轴切分

**这是 v2 的设计核心。** 把三个正交的问题拆开，刷新一个库只需要改**一个文件**。

| 轴 | 目录 | 回答什么 | 有版本号吗 |
|---|---|---|---|
| **形态** | [`knowledge/shapes/`](knowledge/shapes/)（14） | *这是什么？* | **从不** |
| **运行时技术栈** | [`knowledge/runtime-tracks/`](knowledge/runtime-tracks/)（5） | *用什么写的？* | **是——只有这里有** |
| **能力域** | [`knowledge/capabilities/`](knowledge/capabilities/)（19） | *它要做什么？* | **从不** |

而且技术栈文件本身也只被当作**缓存，不是事实来源**：本次会话里 `stack-researcher` 跑出来的报告永远优先。过期的缓存版本号绝不会覆盖一次实时仓库核验。

> **知识库文件用英文书写**，这是刻意的：它们是喂给模型的上下文，不是给人读的文档。运行时检测到你说中文，输出就是中文。给知识库提中文 PR 会被要求改回英文——但 `README.md`、`CONTRIBUTING.md` 和 `knowledge/localization.md` 例外。

### 三个子代理

| 代理 | 职责 |
|---|---|
| [`stack-researcher`](agents/stack-researcher.md) | 写下之前，对权威仓库核验每个版本号。标出预发布版、无人维护的包、以及核验不到的东西。**这是防过期的机制。** |
| [`blueprint-writer`](agents/blueprint-writer.md) | 在隔离上下文里撰写并写出交付物，避免一次长生成把访谈线程淹掉。 |
| [`blueprint-validator`](agents/blueprint-validator.md) | 对成稿做对抗式审计，返回 PASS 或 FAIL 并给出行号。不通过就不呈现给你。 |

版本核验不到时，蓝图会写*"安装前请自行核验"*，而不是猜一个。诚实的空白胜过错误的锁定。

---

## 命令

| 命令 | 做什么 |
|---|---|
| [`/architect`](commands/architect.md) | 完整访谈——四阶段、两道门禁、校验器把关 |
| [`/architect-quick`](commands/architect-quick.md) | 快车道：三个问题、智能默认，确认门禁**照旧** |
| [`/architect-brownfield`](commands/architect-brownfield.md) | 针对存量代码库设计变更 |
| [`/architect-next`](commands/architect-next.md) | 恢复构建——读 `tasks.json`，输出下一个依赖已满足的任务及其验收标准和校验命令 |
| [`/architect-refresh`](commands/architect-refresh.md) | 对照线上仓库重新核验已有蓝图的每个版本锁定，报告什么变了、什么会坏 |
| [`/architect-audit`](commands/architect-audit.md) | 对已有蓝图或蓝图包重跑校验器 |

`/architect-next` 是长周期构建能跨会话存活的关键：一个毫无记忆的新上下文，只需要一个问题被回答——*下一步做什么，怎么知道做完了？*

---

## 存量代码：它也能改已经存在的项目

`/architect-brownfield` 先读仓库再提建议。第 0 阶段产出一张仓库地图——技术栈、约定、目录结构、测试方式、构建与部署路径——之后所有建议都必须与这张图一致。

**仓库现有的约定优先于本插件的默认值。** 它也**绝不会提议重写你没让它碰的、正在正常工作的代码**。涉及迁移（框架、数据库、服务商、语言）时，蓝图必须带 §9.1：对等性验证和切换计划，含每个阶段自己的完成条件和自己的回滚路径。

---

## 项目结构

```
the-architect/
├── .claude-plugin/                # 插件与市场清单
├── CLAUDE.md                      # clone 模式入口
├── skills/architect/SKILL.md      # 插件模式入口——状态机
├── commands/                      # 6 个斜杠命令
├── agents/                        # 3 个子代理
├── knowledge/
│   ├── shapes/                    # 14 —— 这是什么      （无版本号）
│   ├── runtime-tracks/            #  5 —— 用什么写的    （唯一有版本号的地方）
│   ├── capabilities/              # 19 —— 它要做什么    （无版本号）
│   ├── localization.md            # 语言边界契约——非英文会话必读
│   ├── skills-registry.md         # 技能名、安装命令、降级方案
│   └── stack-compatibility.md     # 已知不兼容组合，跨轴
├── questions/                     # 访谈流程的唯一来源
└── templates/                     # 蓝图／CLAUDE.md／tasks.json／任务群模板
```

---

## 贡献

欢迎提 [issue 或 PR](https://github.com/Hainrixz/the-architect/issues)。最需要帮忙的地方：

- **新形态** —— 现有 14 种没覆盖到的项目类型，放 `knowledge/shapes/`
- **能力域** —— 把 `knowledge/capabilities/` 的决策矩阵磨得更锋利
- **运行时技术栈** —— 新生态，或给已有技术栈刷新版本号
- **本地化** —— `knowledge/localization.md` 的中文模糊词黑名单和术语表，欢迎补充实战中踩到的坑

任何动 `knowledge/` 的 PR 都有两条硬规则：**`runtime-tracks/` 之外不许出现版本号**，以及**每个构建步骤都带一个可观测的 "Done when"**。知识库正文一律英文。

---

## 许可

MIT，见 [LICENSE](LICENSE)。

---

## 由 [tododeia.com](https://tododeia.com) · [@soyenriquerocha](https://github.com/Hainrixz) 打造
