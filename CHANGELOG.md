# Changelog

All notable changes to The Architect are documented in this file.

The format is based on [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/), and this
project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) as scoped in
[VERSIONING.md](VERSIONING.md) — read that first if you are wondering why a directory rename is a
MAJOR bump in a repo with no code.

## [Unreleased]

### Added

- **Simplified Chinese (zh-Hans) support.** The interview already ran in the user's language; what
  was missing was everything around it.
  - `knowledge/localization.md` — the new binding contract for non-English sessions. Defines the
    translation boundary (prose and headings follow the user; identifiers, paths, commands, git
    refs, `tasks.json` keys and `status` values stay ASCII English), the canonical bilingual heading
    form `## 9. 构建顺序 (Build Order)`, ASCII slug derivation from a Chinese project name, the
    Simplified-Chinese vague-word blacklist, CJK typography and encoding rules, a fixed EN→ZH
    terminology table, and the failure modes each rule prevents.
  - **Activation.** Simplified-Chinese trigger phrases added to the `architect` skill description,
    `plugin.json`, `marketplace.json`, and all six command descriptions — "帮我设计架构",
    "技术选型建议", "生成蓝图", "分析我现有的代码", "继续构建" and friends now route the same way
    their English and Spanish equivalents do.
  - **Validator hardening.** Finding 2 (unobservable acceptance criteria) gained its Chinese
    counterparts — 正常工作 / 正确处理 / 实现完成 / 符合预期 and the hedges 应该 / 尽量 / 基本上.
    Finding 11 now states explicitly that sections match **by number, never by heading text**, so a
    translated heading is not a missing section. New finding 38 and new **Sweep 28** cover the
    localization boundary: non-ASCII in a path, identifier, git ref or `tasks.json` field;
    translated EARS keywords, `status` values, `NOT APPLICABLE` or `[NEEDS CLARIFICATION]` tokens;
    and a UTF-8 BOM on an emitted `workspace/` file.
  - **README** gained a full 简体中文 section, including the translate/never-translate table.

- **`knowledge/capabilities/china-mainland.md` — the 19th capability.** Shipping to users inside
  mainland China is not a translation problem, it is an architecture problem, and the knowledge base
  had no file that said so. Covers: the inside-versus-outside-the-border fork and why the split
  architecture is a trap; the compliance gates as a table of *who needs it, what it blocks, how long
  it takes* (domain real-name, ICP filing, commercial ICP licence, public-security filing, MLPS 2.0
  grading, generative-AI filing); the 备案-versus-登记 distinction for AI features and the disclosure
  surface it requires; the cross-border personal-information thresholds that decide between exemption,
  standard contract and security assessment; phone-first identity and the `unionid`-versus-`openid`
  data-model trap; WeChat Pay and Alipay mechanics that differ from a card processor — `out_trade_no`
  idempotency, the notification that is not a ledger, daily reconciliation, signing-based recurring
  billing; the network dependencies that resolve everywhere except inside the border; GCJ-02 versus
  WGS-84; and pre-publish content moderation. Nine build steps with scriptable acceptance criteria,
  and eleven pitfalls.
- **Linked from five shapes and four capabilities**, per the reciprocity rule — `saas-webapp`,
  `ecommerce-storefront`, `mobile-app`, `agent-app`, `content-community-platform`, plus `auth`,
  `payments-rails`, `deployment` and `ai-llm-integration`. Two new rows in
  `knowledge/stack-compatibility.md`: a Google-hosted runtime dependency against mainland users, and
  a hardcoded registry mirror against CI running outside the border.
- **The interview now asks.** Phase 1's constraints question and Phase 2's universal follow-ups both
  gained the mainland-China gate, with an explicit instruction not to infer it from the interview
  language — a Chinese-speaking founder building for a US market needs none of it.

### Why (China capability)

The filings are the whole point. Every one of them completes when a regulator acts, which makes them
exactly the acceptance criterion finding 17 forbids as a build gate — an autonomous builder either
stalls forever on "Done when: 备案通过" or quietly self-certifies. Writing them down as §20.1 manual
gates, with the engineering work that *can* be gated separated out, is the difference between a
blueprint that builds and one that stops at step three.

### Why

Detecting the user's language was never the problem. Three things broke silently underneath it: the
skill did not activate on Chinese phrasing, the validator's English-only regexes reported "0 vague
criteria" over a document it could not read, and a translated `status` value made `/architect-next`
report a build finished on turn one. Each failure looked like success.

---

## [2.0.0] — 2026-07-27

The Architect ships as a Claude Code plugin, the knowledge base is split on three axes so it stops
rotting, and every build step a blueprint emits now has a definition of done.

### Added

- **Plugin distribution.** Install without cloning:

  ```
  /plugin marketplace add Hainrixz/the-architect
  /plugin install the-architect@soyenriquerocha
  ```

  Marketplace name is `soyenriquerocha`; plugin name is `the-architect`. Surface lives in
  `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.
- **`skills/architect/SKILL.md`** — the plugin-mode entrypoint. Auto-activates on intent in English
  and Spanish ("design my app", "diseña mi app", "what stack should I use", "hazme un blueprint").
- **Six slash commands** in `commands/`:

  | Command | What it does |
  |---|---|
  | `/architect` | Full interview, DISCOVERY through GENERATE |
  | `/architect-quick` | Three questions, smart defaults, same confirmation gate |
  | `/architect-brownfield` | Reads an existing repo and blueprints a change or migration |
  | `/architect-next` | Resume protocol — hands the builder the next unblocked task from `tasks.json` |
  | `/architect-refresh` | Re-verifies every pin in an existing blueprint against live registries |
  | `/architect-audit` | Runs `blueprint-validator` over an existing blueprint or bundle |

- **Three bundled subagents** in `agents/`:
  - `stack-researcher` — verifies every version against live package registries *before* it is
    written into a blueprint. Its report for the current session outranks any cached file.
  - `blueprint-writer` — sole author of blueprint files. The main thread never writes them.
  - `blueprint-validator` — audits the written blueprint; nothing is presented until it returns PASS.
- **`knowledge/shapes/` — 14 shapes**, up from 6 archetypes. New: `agent-app`,
  `generative-media-app`, `ecommerce-storefront`, `cli-library-mcp`, `browser-extension`,
  `desktop-app`, `automation-bot-integration`, `data-pipeline-analytics`.
- **`knowledge/runtime-tracks/` — 5 tracks**: `ts-node`, `python`, `go`, `rails-laravel`,
  `mobile-native`. The only files in the repo permitted to contain a version number.
- **`knowledge/capabilities/` — 18 capabilities**, up from 8 building blocks. New: `agent-loop`,
  `ai-llm-integration`, `credit-metering`, `payments-rails`, `realtime-voice`, `sync-and-collab`,
  `availability-engine`, `enterprise-readiness`, `accessibility`, `observability`.
- **Acceptance criteria on every build step.** Written in EARS form — *WHEN `<trigger>` THE SYSTEM
  SHALL `<observable response>`* — and paired with a **runnable verify command that must exit 0**
  plus a **git checkpoint**. This is the anti-drift fix: v1 build steps had no definition of done,
  so an autonomous builder had no stopping condition and no way to tell a finished step from a
  half-finished one.
- **Brownfield mode.** Point The Architect at an existing repo instead of an idea. It maps the code,
  then blueprints the change or the migration — including a parity-and-cutover plan and a rollback
  per step. Not just from-scratch builds anymore.
- **Two output modes, chosen in Phase 4.**
  - *Bundle* — `./blueprints/<project-slug>/` containing `blueprint.md`, `tasks.json`, `epics/`, and
    a `workspace/` directory the builder copies wholesale into the target project root.
  - *Single file* — `./blueprints/<project-slug>-blueprint.md`, everything inline, resume is manual.
- **`tasks.json`** — a machine-readable task DAG, the substrate `/architect-next` uses to survive a
  build that spans many sessions.
- **New templates**: `templates/tasks-schema.md`, `templates/epic-template.md`.
- **`questions/phase-4-generate.md`** — generation is now a written seven-step procedure instead of
  improvised in the skill file.
- **`knowledge/stack-compatibility.md`** — known-bad combinations, checked before a stack is locked.
- **Pin provenance in every blueprint** — package, version, source URL, date checked. A package that
  was not researched says so rather than implying a verification that never happened.
- **`CHANGELOG.md`, `VERSIONING.md`, `CONTRIBUTING.md`, `SECURITY.md`.**

### Changed

- **BREAKING — the knowledge base was reorganized onto three axes.** v1 hardcoded a stack table into
  every archetype file, so refreshing one framework meant editing 13 files and each one drifted at
  its own rate. v2 separates *what it is* from *what it is written in* from *what it does*:

  | Axis | Directory | Answers | Version pins |
  |---|---|---|---|
  | Shape | `knowledge/shapes/` | What is it? | Never |
  | Runtime track | `knowledge/runtime-tracks/` | What is it written in? | **Only here** |
  | Capability | `knowledge/capabilities/` | What does it do? | Never |

  Refreshing a framework now edits one file.

- **BREAKING — paths moved.** `knowledge/archetypes/*` → `knowledge/shapes/*`;
  `knowledge/building-blocks/*` → `knowledge/capabilities/*`, with these renames:

  | v1 | v2 |
  |---|---|
  | `archetypes/content-platform.md` | `shapes/content-community-platform.md` |
  | `building-blocks/auth-patterns.md` | `capabilities/auth.md` |
  | `building-blocks/database-patterns.md` | `capabilities/database.md` |
  | `building-blocks/deployment-patterns.md` | `capabilities/deployment.md` |
  | `building-blocks/api-design-patterns.md` | `capabilities/api-design.md` |
  | `building-blocks/frontend-stacks.md` | `capabilities/frontend-architecture.md` |
  | `building-blocks/testing-patterns.md` | `capabilities/testing.md` |
  | `building-blocks/styling-systems.md` | `capabilities/styling.md` |
  | `building-blocks/state-management.md` | `capabilities/state-management.md` (content rewritten) |

- **BREAKING — the blueprint section contract went from 16 sections to 20.** All 20 headings appear
  in every blueprint; a section that does not apply says `NOT APPLICABLE — <reason>` under its
  heading rather than being deleted, because downstream tooling indexes by number. New sections
  include Security & Secrets (14), Accessibility (15), Observability & Cost (16), Model Routing (17),
  Agent Workspace (19), and Acceptance Gate, Risks & Decision Log (20). **The generated `CLAUDE.md`
  moved from Section 15 to Section 19.1.**
- **BREAKING — blueprints are written to the user's current working directory** (`./blueprints/`),
  never into this repo and never into the plugin cache. The plugin cache is not a writable workspace.
- **BREAKING — the phase model is now a five-state machine** with an explicit confirmation gate:
  `DISCOVERY → DEEP DIVE → ARCHITECTURE →(user confirms)→ GENERATE`, plus `BROWNFIELD` as an
  alternate entry that skips DISCOVERY and merges into ARCHITECTURE.
- **Version pins are verified live, not recalled.** `stack-researcher`'s report for the current
  session is authoritative; `knowledge/runtime-tracks/<track>.md` is a dated cache and the fallback
  for anything the researcher could not resolve, with its caveats carried into the blueprint
  verbatim. Never write a pin from memory.
- **Classification is now bidirectional.** Every shape's "No if" list points at the shape you
  actually want, and every confusable pair points back — so briefs stop funnelling into `saas-webapp`
  by default. `saas-webapp.md` is the declared sink and carries exits to all 13 other shapes.
- **`CLAUDE.md` is now explicitly the clone-mode entrypoint.** Cloning still works exactly as it did
  in v1. Plugin and clone are dual, supported modes reading the same `questions/`, `templates/` and
  `knowledge/` files — the interview flow is never duplicated between them.
- The generated target-project `CLAUDE.md` guidance is **under 200 lines** (was described as 120).

#### If you are coming from v1

1. **Your existing blueprints still work.** A v1 blueprint is a standalone `.md`; nothing in this
   release reaches into it. Keep building from it.
2. **Run `/architect-audit <path>` on it if you plan to keep using it.** It will FAIL — v1 blueprints
   have no acceptance criteria. That is the expected result, not a bug, and the report tells you
   which steps have no definition of done.
3. **Run `/architect-refresh <path>` before starting an old build.** Pins written months ago have
   drifted; this re-verifies each one against the live registry and reports what moved and what
   breaks.
4. **If you cloned, `git pull` is enough** — `CLAUDE.md` still boots The Architect.
5. **If you forked or edited `knowledge/archetypes/` or `knowledge/building-blocks/` locally,
   your edits will not merge.** Those directories are gone. Port each change into the axis it
   belongs to: project-type material into `knowledge/shapes/`, cross-cutting decisions into
   `knowledge/capabilities/`, and anything with a version number in it into
   `knowledge/runtime-tracks/`.
6. **Update any script or doc that reads `output/<project>-blueprint.md`** — output now lands in
   `./blueprints/` relative to wherever you ran the command.

### Deprecated

- **The v1 blueprint format** (16 sections, no acceptance criteria, no verify commands) is
  deprecated as an input. `/architect-audit` and `/architect-refresh` still read it, and will keep
  reading it through the 2.x line, but nothing produces it anymore. Regenerate rather than patch if
  a v1 blueprint is going to an autonomous builder.

### Removed

- **`knowledge/archetypes/`** (6 files) — superseded by `knowledge/shapes/`.
- **`knowledge/building-blocks/`** (8 files) — superseded by `knowledge/capabilities/`.
- **`output/` as the blueprint destination.** Generated work no longer lands inside this repo.
- **`.claude/commands/` is never emitted into a generated workspace**, in either output mode. A slash
  command only fires when a human types it, and an autonomous builder types nothing — emitting one
  produced a workflow that silently never ran. Repeatable project workflows are emitted as
  `.claude/skills/<name>/SKILL.md` instead.

### Fixed

- **Staleness.** One framework refresh used to mean editing every archetype that mentioned it, so in
  practice most of them stayed wrong. Pins now live in exactly one place and are re-verified against
  the registry at generation time.
- **Drift.** Build steps with no observable completion condition let an autonomous builder run past a
  broken step or stall on a finished one. Every step now carries EARS acceptance criteria, a verify
  command, and a checkpoint, and is sized to one sitting.
- **Unreachable shapes.** 20 of 60 classification edges were one-way, so the interview reliably
  resolved to about 8 of the 14 shapes. Reciprocity is now required in both directions for every
  confusable pair.
- **Orphaned capability.** Booking and scheduling briefs landed on `saas-webapp` and silently lost
  `capabilities/availability-engine.md`. The classifier now carries booking, scheduling, and
  appointment language, and the shape links the capability.
- **Un-checkable acceptance criteria.** A "done when" that waits on a store review queue, a
  certificate authority, or a human reviewer cannot terminate inside an autonomous build — the
  builder either stalls forever or self-certifies. Anything requiring an external party moved to a
  post-build launch checklist, clearly separated from the build order.
- **Turn-scoped tool grants.** `allowed-tools` in the skill front matter clears the moment the user
  sends their next message, so it could never cover a multi-turn interview. The skill no longer
  assumes a standing grant.

### Security

- Added [SECURITY.md](SECURITY.md) with a private reporting route and the threat model that actually
  applies to a prompt repo: injection through a contributed knowledge file, a malicious install
  command in the skills registry, and a blueprint that instructs a builder to do something harmful.
- Blueprints gained a dedicated **Security & Secrets** section (14), covering secret handling, env
  var boundaries, and what must never be committed. v1 had no equivalent section.
- Pin provenance (source URL + date checked) is recorded per package, so an unverified or
  typosquatted dependency cannot enter a build looking like a verified one.

---

## [1.0.0] — 2026-07-27

First public release.

### Added

- `CLAUDE.md` — the meta-agent prompt that turns a Claude Code session into The Architect, running a
  four-phase workflow: Discovery, Deep Dive, Architecture, Generate.
- `knowledge/archetypes/` — 6 project archetypes: SaaS/web app, marketing site, mobile app,
  API/backend, internal tool, content platform.
- `knowledge/building-blocks/` — 8 cross-cutting decision guides: auth, database, deployment, API
  design, frontend stacks, testing, styling, state management.
- `knowledge/skills-registry.md` — Claude Code skills mapped to workflow phases and blueprint
  sections, with licenses, star counts, and install commands.
- `knowledge/stack-compatibility.md` — what works together and what does not.
- `templates/blueprint-template.md` — the 16-section blueprint skeleton, with Build Order as the
  load-bearing section.
- `templates/claude-md-template.md` — the target project's `CLAUDE.md`.
- `questions/phase-1-discovery.md`, `questions/phase-2-branches.md`,
  `questions/phase-3-confirmation.md`.
- Bilingual EN/ES `README.md`; MIT `LICENSE`.

### Changed

- Branding corrected throughout: `iia.com` → `tododeia.com`.

### Fixed

- **Skill references were wrong in the initial commit and are now verified.** Every star count,
  license, and install command in `knowledge/skills-registry.md` was read from the live GitHub API
  or the upstream repo's own README — nothing estimated.
- **Removed skills that do not exist**: `/deep-research`, `/seo-audit`, `/pdf-design`, `/shadcn-ui`,
  `/chrome-bridge-automation`, `/web-reader`, `/humanizer`.
- **Corrected invocation forms.** A leading `/` means a real slash command; no slash means the skill
  auto-activates and must be named in prose. Writing `/ui-ux-pro-max`, `/frontend-design` or
  `/playwright-cli` is a silent no-op — nothing runs and the workflow quietly skips the step.
- **No skill is a hard dependency.** Every reference degrades gracefully: if a skill is absent, fall
  back to the knowledge base or built-in `WebSearch`/`WebFetch`, note it in one line, and keep going.
- Corrected the `pdf` skill's description — it does extraction, manipulation, and forms, not visual
  design.

[Unreleased]: https://github.com/Hainrixz/the-architect/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/Hainrixz/the-architect/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/Hainrixz/the-architect/releases/tag/v1.0.0
