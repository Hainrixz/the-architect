# Blueprint Template

> The skeleton of every blueprint The Architect emits. Fill **every** section. A Claude Code
> instance with zero prior context must be able to build the whole project from the output without
> asking a single clarifying question.

Last verified: 2026-07-28

**Path resolution:** every bare path in this file (`knowledge/…`, `templates/…`, `questions/…`) is
relative to the plugin root — read it as `${CLAUDE_PLUGIN_ROOT}/<path>`. The only exception is
`./blueprints/`, which is always the **user's current working directory**, never the plugin.

---

## How to use this template

**Read this block, then delete it from the generated blueprint.** Everything below the second
horizontal rule is the deliverable.

### Two emission modes

A blueprint may be emitted as **one file** or as a **bundle directory**. The architecture sections
are identical in both. Only Section 19 and the task manifest differ.

**The mode is derived, not asked — `questions/phase-4-generate.md` Step 2 is the authority.** It
comes from the §9 step count: **12 steps or more → bundle, 11 or fewer → single file**, announced in
one line. Never ask the user which mode they want; the answer changes zero design decisions. An
explicit preference the user volunteered earlier wins over the threshold — honor it silently. The
table below describes what each mode *is*, so you can say why the derived answer fits; it is not a
menu to present.

| Mode | What it is for | Layout |
|---|---|---|
| **Single file** | One builder, a build measured in days, nothing to resume | `./blueprints/{project-slug}-blueprint.md` |
| **Bundle** | Parallel builders, a build measured in weeks, or resumable state across sessions | `./blueprints/{project-slug}/` — the tree below |

**Bundle mode — this is the only layout. Do not invent a variant.**

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
    ├── <verify-critical config>   # test/e2e runner config, compose file — §19.6
    └── .claude/
        ├── settings.json
        ├── skills/<name>/SKILL.md
        └── rules/<name>.md
```

`workspace/` exists so the builder copies **one directory** into the target project root — one
guarded copy and the agent configuration is in place. That is why the files sit under `workspace/`
rather than loose at the bundle root: loose files force the builder to reason about which of them
belong to the blueprint and which belong to the project. **The copy is written as a re-runnable
command** — see §19, *The workspace copy must be safe to re-run*.

`workspace/` holds more than agent configuration. **Every config file a Section 9 `Verify` command
needs in order to run is a real file here too** — see §19.6. It mirrors the target repo layout
exactly, so a file's path under `workspace/` is its path in the project.

**And because the bundle lives *inside* the project, those files are visible to the project's own
tooling.** A root config under `workspace/` is a second root config in one tree, which is enough to
make a formatter or linter exit 1 before it checks anything. Every emitted config excludes the bundle
path — see §19.6, *The bundle sits inside the project*.

In bundle mode: Section 9 stays the human-readable build order, and each step is *also* a task in
the machine-readable manifest — see `templates/tasks-schema.md`. Steps group into epics — see
`templates/epic-template.md`. Section 19's files are emitted as real files under `workspace/`
instead of as fenced blocks.

**Single-file mode:** everything inline in `./blueprints/{project-slug}-blueprint.md`. Section 19
emits the workspace files as fenced code blocks instead of real files. No `tasks.json`, no `epics/`.
Resume is manual. One file to send, paste, or commit anywhere — that is the whole point of the mode.

**`.claude/commands/` is NEVER emitted, in either mode.** See Section 19.

### Rules for you, the generating agent

1. **No placeholder survives.** `{braces}` are instructions to you, not output. If you genuinely
   cannot fill one, write `NOT APPLICABLE — {one-line reason}`. Never leave a brace.
2. **Version pins come from the `stack-researcher` report produced in this session.** That report is
   the authority. `knowledge/runtime-tracks/{track}.md` is the **fallback** for any package the
   researcher did not resolve — and its unverified caveats carry through into the blueprint
   verbatim. **Never write a pin from memory.** A runtime track is a cache that was right the day it
   was written and drifts after; a stale cached pin never overrides a live registry check. Every pin
   carries its provenance — package, version, source URL, date checked — into Section 11. A package
   that was not researched says so rather than implying a verification that did not happen.
3. **Every build step has an observable, machine-checkable "Done when".** This is the release rule.
   See Section 9.
4. **Section numbering is fixed.** All 20 sections appear even when a section is not applicable —
   write `NOT APPLICABLE — {reason}` under the heading instead of deleting it. Downstream tools
   index by number.

   **A `NOT APPLICABLE` section may never be the source of a contract a later step depends on.** If
   a §9 step says "produce output matching the format in §7" and §7 reads `NOT APPLICABLE`, the
   format exists nowhere: the builder invents 100% of it, and every later step that measures against
   "the contract" is measuring against the invention. A real blueprint did exactly this — step 2 was
   told to commit two human-readable outputs *"byte-exact, as the contract later steps are measured
   against"*, while both sections that would have defined that format were `NOT APPLICABLE` and
   redirected elsewhere, and not one line of the format appeared anywhere in 1,967 lines.

   So before you mark a section `NOT APPLICABLE`, grep your own output for every reference to it —
   by number (`§7`, `Section 7`) and by name. Every hit is a defect with exactly two fixes and no
   third: **give the section real content** (it was applicable after all), or **move the contract to
   a concrete place** — a fenced block in the step itself, a fixture file emitted in §19.6, a table
   in a section that does apply — and point the step at that place, with the literal content
   present. A live cross-reference into a `NOT APPLICABLE` section is never left standing.
5. **Be opinionated.** Every table row with a "Why" column gets a real reason, not a restatement.
   "Postgres — because it's a database" is a defect.
6. **Language.** Write the blueprint in the user's language. Keep code, commands, file paths,
   env var names, and EARS keywords (`WHEN`, `THE SYSTEM SHALL`) in English regardless.
   When that language is not English, `knowledge/localization.md` is binding, and two rules decide
   whether the document survives its own validator:
   - **Numbered headings keep the number *and* the English name** — `## 9. 构建顺序 (Build Order)`.
     Validator finding #11 matches by number; the English name is what every name-based
     cross-reference and `/architect-audit` read.
   - **Nothing a machine reads is translated** — the slug and every path (ASCII, from an English
     short name or toneless pinyin), table and column names, API routes, git tags and branches,
     `tasks.json` keys and its `status` values `pending` / `in_progress` / `done`, and the tokens
     `[NEEDS CLARIFICATION: …]` and `NOT APPLICABLE`. Files are UTF-8 **without BOM**.

---
---

# {PROJECT_NAME} — Blueprint

> Generated by The Architect on {DATE}
> Shape: {shape} · `knowledge/shapes/{file}.md`
> Runtime track: {track} · `knowledge/runtime-tracks/{file}.md`
> Emission mode: {single file | bundle}
> Blueprint version: 1
> Versions last verified: {DATE} — see §11 for per-package provenance

---

## 1. Project Overview & Non-Goals

### Vision
{1-2 paragraphs: what this is, who it's for, what problem it solves. Concrete. If you can swap in a
competitor's name and the paragraph still reads true, it is too vague — rewrite it.}

### Users
| Persona | What they come to do | Frequency |
|---|---|---|
| {persona} | {primary job-to-be-done} | {daily / weekly / once} |

### Goals — v1 scope
1. {Goal — phrased as a capability the product has, not a task the builder does}
2. {Goal}
3. {Goal}

### Non-Goals — explicitly out of scope for v1

{**Mandatory. Never leave empty.** This is the scope fence. An autonomous builder with an ambiguous
boundary will build the ambiguity, and every extra feature is a new surface that can fail the
acceptance gate. List 5-8. Each gets a reason and a revisit trigger.}

| Not building | Why not now | Revisit when |
|---|---|---|
| {feature} | {reason — cost, risk, unvalidated demand, needs data we don't have} | {concrete trigger} |

**The builder must not implement anything in this table**, even if it seems like a small addition
while working on an adjacent step. If a step appears to require a non-goal, that is a blueprint
defect — stop and report it rather than expanding scope.

### Success metrics
| Metric | Target | How measured |
|---|---|---|
| {metric} | {number + date} | {the query, event, or dashboard that produces it} |

---

## 2. Tech Stack

**Runtime track: {track}.** This table names *choices*, not versions. Every version pin lives in
Section 11 and nowhere else — do not restate one here and do not maintain a second copy.

Pins come from the `stack-researcher` report produced in this session, which is the authority.
`knowledge/runtime-tracks/{file}.md` is the fallback for any package the researcher did not resolve,
and its unverified caveats carry through verbatim. Never write a pin from memory.

| Layer | Choice | Why this, over what |
|---|---|---|
| Language / runtime | {choice} | {rationale, naming the rejected alternative} |
| Framework | {choice} | {rationale} |
| Styling | {choice} | {rationale} |
| Component layer | {choice} | {rationale} |
| Database | {choice} | {rationale} |
| ORM / data access | {choice} | {rationale} |
| Auth | {choice} | {rationale} |
| Background work | {choice} | {rationale} |
| Payments | {choice or NOT APPLICABLE} | {rationale} |
| File storage | {choice or NOT APPLICABLE} | {rationale} |
| Email / notifications | {choice} | {rationale} |
| Hosting | {choice} | {rationale} |
| Package manager | {from the runtime track} | {rationale} |

### Compatibility check
{Confirm this combination against `knowledge/stack-compatibility.md`. State explicitly:
"Checked against knowledge/stack-compatibility.md — no known-bad combinations." Or name the conflict
and how this blueprint avoids it.}

---

## 3. Directory Structure

```
{project-name}/
  {Complete tree with an inline comment on every directory and every non-obvious file.
   Include config files, the .claude/ workspace from Section 19, tests, and CI.
   The builder creates exactly this — no improvised folders.}
```

**Boundary rules**
- {e.g. Nothing in `src/app/` imports from another route's folder. Shared code moves to `src/lib/`.}
- {e.g. `src/lib/db/` is the only place that opens a database connection.}

**If any boundary rule states how modules refer to each other** — a specifier or extension form, a
path alias, an import-order or barrel-file rule, a link mode — it is a *resolution convention*, and
it is only half-written here. **Reconcile it against every context that loads those modules, in
§19.6's resolution convention matrix**, and repeat the resolved form here only by pointing at that
table. A convention stated in this section and never checked against the script runner, the test
runner, and the compiler is the single most expensive defect a directory-structure section can
carry: it reads as a tidy rule and it stops two build steps dead.

**Every output path drawn in this tree is a value some emitted config also states** — the build
config's output directory, the manifest's entry point, the packaged binary. Draw it once here, take
the literal string from §19.6's *Cross-artifact value reconciliation* table, and never let this tree
and the config that produces the path disagree: a tree showing `dist/cli.js` above a build config
that emits `dist/cli/index.js` is two correct-looking artifacts and one unbuildable project.

**A tree entry is documentation. Drawing a file here does not create it.** Every file in this tree
must have exactly one of two origins, and you must be able to name which one for any file you draw:

1. **A §9 step authors it** — it appears by name in that step's **Do** list, and in bundle mode in
   that task's `files` array. One step owns it; "it gets created along the way" is not an origin.
2. **It is emitted as a real file under `workspace/`** (§19) and lands in the project via the single
   copy the builder runs before step 1.

**Any config file a `Verify` command depends on takes origin 2 unless a step explicitly authors
it** — test-runner config, e2e-runner config, path-alias or tsconfig-path setup, the local service
compose file, test setup/fixture files. A file that is drawn here and written nowhere fails at the
first gate that needs it, with `Cannot find module`, `No test files found`, or `No tests found` —
errors that look like broken code and are actually a missing artifact, and the builder's only
recovery is to invent the file. See §19.6, which is where those files are emitted — **and where the
obligation continues past existence**: an emitted config that cannot resolve a package this
blueprint mandates, or that never loads the env vars its tool reads, fails exactly like a file that
was never written.

---

## 4. Data Model

### Entities

**{Entity}** — {one line: what it represents, its lifecycle}

| Field | Type | Constraints | Notes |
|---|---|---|---|
| {field} | {type} | {PK / FK / unique / not null / default} | {meaning, not a restatement of the name} |

{Repeat for every entity. Include join tables. Include audit/soft-delete columns if used.}

### Relationships
{Every relationship as `A —(cardinality)→ B`, plus the cascade behavior on delete. Ambiguous
cascades are a top source of data-loss bugs.}

### Indexes
| Table | Index | Why |
|---|---|---|
| {table} | {columns} | {the query it serves} |

### Schema
```{sql | prisma | drizzle | whatever the chosen data layer uses}
{The real schema definition. Complete — the builder pastes this, not a sketch of it.}
```

### Migrations
{Migration tool, naming convention, how to run them, and the rule for production migrations
(e.g. expand-then-contract, never a destructive migration in the same deploy as the code change).}

### Seed data
{What a fresh local database gets so the app is usable on first run: an admin user, a demo tenant,
sample rows. Name the file and the command.}

---

## 5. API Design

### Conventions
- Base path: {/api/v1}
- Response envelope: {exact shape for success and for error — one shape, no exceptions}
- Error codes: {the enumerated set, with HTTP status mapping}
- Validation: {library and where schemas live}
- Pagination: {cursor or offset, param names, default and max page size}
- Idempotency: {which endpoints accept an idempotency key and how it's stored}
- Rate limits: {per-endpoint or global, the limit, the storage backend}

### Routes

| Method | Path | Description | Auth | Rate limit |
|---|---|---|---|---|
| {GET} | {/api/v1/...} | {what it does} | {public / user / role} | {limit} |

### Critical endpoints — full detail

{For the 3-5 endpoints the product cannot work without, write: request schema, response schema,
every validation rule, every error case with its status and code, and side effects (emails sent,
jobs queued, rows written). These become the acceptance criteria in Section 9.}

---

## 6. Frontend Architecture

{If this project has no frontend, write `NOT APPLICABLE — {reason}` and continue to Section 7.}

### Routes
| Route | Page | Data source | Auth |
|---|---|---|---|
| {/} | {name} | {server query / API call / static} | {public / user} |

### Rendering strategy
{Which routes are server-rendered, which are static, which are client-only, and the cache/revalidate
policy for each. Name the exact directives from the runtime track — do not describe them generically.}

### Component hierarchy
```
{Tree for the 2-3 most important pages, marking server vs client boundaries.}
```

### State management
{Server state vs client state, the fetching/caching library, form state, optimistic updates, and
what is deliberately NOT put in global state.}

### Loading, empty, and error states
{Every list and every async surface needs all three specified. Missing empty states are the most
common gap in agent-built UIs.}

---

## 7. Design System

{Produced with `ui-ux-pro-max` where available; otherwise from `knowledge/capabilities/styling.md`.
Every value below is a literal — no "a warm neutral", no "medium spacing".}

### Colors
| Token | Light | Dark | Usage |
|---|---|---|---|
| `--primary` | {#hex} | {#hex} | {buttons, links, focus ring} |
| `--primary-fg` | {#hex} | {#hex} | {text on primary} |
| `--background` | {#hex} | {#hex} | {page} |
| `--surface` | {#hex} | {#hex} | {cards, panels, modals} |
| `--border` | {#hex} | {#hex} | {dividers, inputs} |
| `--fg` | {#hex} | {#hex} | {body text} |
| `--fg-muted` | {#hex} | {#hex} | {secondary text} |
| `--destructive` | {#hex} | {#hex} | {errors, delete} |
| `--success` | {#hex} | {#hex} | {confirmations} |

**Contrast:** every foreground/background pair above meets WCAG 2.2 AA — 4.5:1 for body text,
3:1 for large text and UI component boundaries. State the measured ratio for the three riskiest
pairs. {If a pair fails, fix the hex here — do not ship it and fix it later.}

### Typography
| Role | Family | Size / line-height | Weight | Tracking |
|---|---|---|---|---|
| Display | {family} | {rem/px} | {weight} | {value} |
| Heading | {family} | {scale} | {weight} | {value} |
| Body | {family} | {size} | {weight} | {value} |
| Mono | {family} | {size} | {weight} | {value} |

**Font loading:** {self-hosted or CDN, subset, `font-display` value, and the fallback stack.}

### Spacing, radius, elevation
- Spacing scale: {base unit + the full ramp}
- Radius: {per component class}
- Shadows: {literal values per elevation level}
- Max content width: {value} · Breakpoints: {values}

### Motion
{Durations and easing per interaction class. Everything must respect
`prefers-reduced-motion: reduce`. Use `emil-design-eng` for anything with meaningful transitions.}

### Component style
{The aesthetic in two or three sentences, written so a builder can tell whether a new component
belongs. Name the reference if one was analyzed.}

---

## 8. Authentication & Authorization

{If the project has no accounts, write `NOT APPLICABLE — {reason}` and continue.}

### Provider and rationale
{From `knowledge/capabilities/auth.md`. One choice, one reason.}

### Flows
{Sign-up → verification → onboarding → first useful screen. Also: sign-in, password reset, session
expiry, sign-out, and account deletion. Every branch, including the failure branches.}

### Route protection
| Surface | Rule | Enforced where |
|---|---|---|
| {/dashboard/*} | {authenticated} | {exact file} |
| {/api/v1/admin/*} | {role = admin} | {exact file} |

**Enforcement rule:** authorization is checked server-side on every request. Client-side route
guards are cosmetic and never the only check. A hidden button is not a permission.

### Roles and permissions
| Role | Can | Cannot |
|---|---|---|
| {role} | {explicit list} | {explicit list} |

### Sessions
{Token type, storage, lifetime, refresh behavior, cookie flags (`HttpOnly`, `Secure`, `SameSite`),
and CSRF strategy.}

### Multi-tenancy / row-level isolation
{If applicable: how a query is prevented from crossing a tenant boundary. Name the mechanism —
RLS policy, mandatory scope in the data layer, or both. "Remember to filter by org_id" is not a
mechanism.}

---

## 9. BUILD ORDER

**This is the section the whole blueprint exists to produce.** Everything above is context; this is
the instruction set. A builder that follows Section 9 exactly and stops when each gate goes green
ships the project. A builder given steps without gates drifts, over-builds, and declares victory on
work that does not run.

### The rules of a step

1. **One sitting per step.** Agent success rate drops sharply and non-linearly with task length —
   long steps are where builds fail. If a step touches more than **5 files** or carries more than **6 acceptance criteria**, or needs
   more than one round of design thinking, split it. Those two numbers are the same ones
   `templates/tasks-schema.md` enforces and `blueprint-validator` files as finding #3 — one fact,
   stated identically everywhere, per rule 10.
2. **Every step carries all four fields:** `Do`, `Done when`, `Verify`, `Checkpoint`. A step missing
   any of them is not a step.
   - **`Checkpoint` is a literal shell block, not a sentence.** It commits the step's work and tags
     it: `git add -A && git commit -m "step {N}: {slug}"` then `git tag step-{NN}-{slug}`. The tag
     name is `step-` + the zero-padded step number + the step slug — `step-07-billing`, not
     `step-7` and not `billing`. That tag is the rollback target for the *next* step, so a step
     without one leaves the build with nowhere to fall back to. The validator checks for it, and
     §20.1 checks that every tag exists in git at the end.
3. **"Done when" conditions are observable and machine-checkable.** Prefer EARS form:
   **WHEN** `<trigger>` **THE SYSTEM SHALL** `<observable response>`.
4. **"It looks right" is banned.** So are: *works*, *is implemented*, *renders correctly*, *is
   wired up*, *functions as expected*, *is complete*. If a human has to squint at a screen to
   decide, it is not a completion criterion. Convert it: what number, what row, what status code,
   what exit code, what pixel value in a snapshot?
5. **`Verify` is literal shell.** Copy-pasteable, with the expected result in a comment. If a
   condition cannot be verified by a command, write the test that makes it verifiable — in the same
   step.
6. **A step is not done until its own verify commands pass** *and* the previous steps' still do.
7. **Checkpoint every step:** `git tag step-NN-{slug}`. This gives a rollback target. If a step goes
   wrong, `git reset --hard step-{N-1}` and retry — never debug forward through a broken checkpoint.
8. **Never skip ahead.** If step 7 is blocked, stop and report. Do not start step 8 "in the
   meantime" — later steps assume earlier gates are green.
9. **A step may never introduce a requirement that retroactively breaks an earlier step's
   `Verify`.** Rule 6 says the previous steps' gates must still pass; this is the design obligation
   that makes rule 6 satisfiable, and it is on you, not the builder. Whenever step N adds a check
   that runs inside an earlier gate — boot-time env validation, a lint or type rule, a schema
   constraint, a CI stage, a required header — that check must hold on the tree steps 1…N leave
   behind, with only what those steps built. Before you write step N+1, re-read every earlier
   `Verify` block and confirm it still exits 0.

   **The canonical violation is env validation.** A validator that requires every variable in §10's
   table the moment it is written makes `build` fail from that step until every secret exists —
   including the ones §10 itself says are not needed until step 16. The builder's only way forward
   is to fabricate values for services it has not integrated. **So env validation degrades by step:
   a variable is required only from the step §10's "Required by step" column names, and optional
   before it.** The feature that consumes a variable is also the step that promotes it to required.
   Same shape for everything else: a rule ships in the step whose code satisfies it, never earlier.

10. **Never assert a derived number you did not count.** Any number a `Verify` command checks — a
    table count, a test count, a route count, a migration count, a file count, a row count — is a
    *derived* fact about this blueprint's own content, and it is only true if you counted it from
    that content at write time. Writing a plausible number is not a shortcut; it is a gate that
    fails on **every** machine, for a reason that has nothing to do with the builder's code, and
    the builder's only recovery is to decide which of your two numbers to believe.

    A blueprint shipped `\dt | grep -c` **expecting 7 tables** while §4's schema defined **8**,
    and repeated `7` in five places. Every builder, everywhere, failed that gate at step 3.

    Two obligations, both on you:

    - **Count, then write.** Derive the number from the artifact it describes — count the `CREATE
      TABLE`s in §4's schema block, the `it(`/`test(` cases in the test file the step authors, the
      rows in §5's route table. If you cannot count it because the artifact does not exist yet
      (a runner reports a test count only once the tests are written), you may not assert it.
    - **One number, everywhere.** The same count appears identically in the step's **Do**, its
      **Done when**, its `Verify` comment, §20.1's gate, and — in bundle mode — the `tasks.json`
      and epic renderings of that criterion. Change the schema and you change all of them, in the
      same pass. Two different numbers for one fact is a defect even when one of them is right.

    **Prefer a property over a magic number wherever the property is what actually matters.** The
    real requirement is almost never "there are exactly N tables" — it is "every table §4 defines
    exists". Assert that instead, and the gate stops drifting the moment the schema changes:

    | Brittle — breaks on every edit | Durable — asserts the property |
    |---|---|
    | `psql -c '\dt' \| grep -c table` → `# expect: 7` | one `\d <table>` per §4 entity, each expected to exit 0 |
    | `# expect: 42 passed` | `# expect: exit 0, 0 failed, 0 skipped` |
    | "the app exposes 9 routes" | each route in §5's table returns its documented status |
    | `ls migrations \| wc -l` → `# expect: 3` | `db:migrate` exits 0 and the schema-drift check reports no diff |

    A count is still legitimate where the count *is* the requirement — "exactly 1 row after a
    replayed webhook" is a real invariant, not a tally of the blueprint's own contents. The test is
    whether the number would have to change when the blueprint is edited. If it would, count it and
    propagate it; better, assert the property.

11. **A `Verify` command exits 0 when the step is correct.** That is the entire signal a runner has.
    CI, the resume protocol, and any wrapper that chains steps read a non-zero exit as *this gate
    failed* — none of them can distinguish "the tool errored" from "the tool correctly errored".

    So the trap is testing a **documented error path**, which is a legitimate thing to gate: exit
    codes are a public interface, §5 may enumerate them, and the natural way to write the check is
    the way that breaks the build.

    | Silently fails the gate | Correct — the line itself exits 0 |
    |---|---|
    | `mytool --bad-flag`  `# expect: exit 2` | `mytool --bad-flag; test $? -eq 2`  `# expect: exit code 2 → this line exits 0` |
    | `mytool query 'no-such-tag'`  `# expect: exit 1, no results` | `mytool query 'no-such-tag'; test $? -eq 1`  `# expect: exit code 1 → this line exits 0` |
    | `grep -q FIXME out.txt`  `# expect: no match` | `! grep -q FIXME out.txt`  `# expect: no match → exits 0` |
    | `curl -f localhost:3000/missing`  `# expect: 404` | `test "$(curl -s -o /dev/null -w '%{http_code}' localhost:3000/missing)" = 404` |

    **Assert the code; never let the failure escape.** The expected code stays visible in the
    command or the comment — the point is not to hide it, it is to make the *assertion* the thing
    that decides the exit status. Read every `Verify` block top to bottom before you emit it and
    confirm that a correct step leaves the block with status 0, including under `set -e`.

12. **A check must be possible in the medium it runs in.** Before writing any `Done when` or
    `Verify`, confirm the property is *observable* by the thing doing the observing. A runtime check
    cannot see constructs erased before runtime — types, interfaces, type-only exports, macros,
    comments, stripped annotations, dead-code-eliminated branches. A static parse cannot see a value
    computed at runtime. A type check cannot see I/O. A linter cannot see network behaviour. A
    snapshot cannot see a race.

    A blueprint asked a script to compare a **runtime module namespace** against a documented export
    surface whose rows were largely **type-only**. Types are erased at runtime, so the runtime object
    could never contain them: the check failed in one direction and, inverted, failed in the other.
    It was unsatisfiable in principle — and the builder's only way past it was to rewrite the check,
    which means the gate proved whatever the builder decided it proved.

    When the property is not observable in that medium there are exactly two moves and no third:

    | Move | Example |
    |---|---|
    | **Change the medium** | assert type-only exports with the type checker or a static parse of the source, not with a runtime import |
    | **Change the property** | assert only the runtime-visible subset, and say in the criterion that it *is* the subset |

    State the medium in the criterion when it is not obvious — "WHEN `{typecheck command}` runs" and
    "WHEN `{script command}` runs" are different gates with different visibility. Never emit a check
    the medium cannot decide.

13. **The first step that produces an executable, a published entry point, or a served endpoint
    RUNS it in that same step's `Verify`.** Building is not exercising. A step whose gate compiles
    the binary but never invokes it proves the compiler was happy and nothing else — not that the
    output landed where the manifest says it lands, not that the runtime can find it, not that it
    starts.

    **This is the fail-fast rule, and it is about ordering as much as about gates.** A contradiction
    between two emitted artifacts costs one line to fix and seven steps to discover if nothing runs
    the thing until step 8. A real blueprint emitted a build config that wrote its output to one
    path and a package manifest that declared the entry point at another; ~30 `Verify` commands, the
    packaging step and the install smoke test all named the manifest's path. No step before 8 ran the
    binary, so the mismatch surfaced after seven steps of green gates — and both files were
    blueprint-authored and declared off-limits to the builder, so there was no legal way forward.
    Seven steps of work were unreachable because of one line.

    Two obligations, both on you:

    - **Exercise, don't merely produce.** The step that first emits an artifact meant to be *run*
      gates on running it: invoke the binary (`--version`, `--help`) and assert exit 0, curl the
      endpoint and assert the documented status, import the published entry point and assert it
      loads, start the container and assert the healthcheck goes green. A stub that prints a version
      string is enough — the point is that the path, the manifest, the permissions and the
      interpreter line are all exercised while the fix still costs one line.
    - **Order so every cross-artifact contract is exercised at the earliest step where both sides
      exist.** If step 1 writes the build config and step 2 writes the manifest, the contract between
      them is testable at step 2 — gate it there, not at the step that finally consumes it. Ask of
      every pair of emitted artifacts that reference each other: what is the earliest step at which a
      command could catch a disagreement, and does that step's `Verify` run it? The values that get
      claimed twice are enumerated in §19.6's *Cross-artifact value reconciliation* table; this rule
      is what makes a disagreement in that table fail early instead of late.

14. **A `Verify` may not depend on what its own `Checkpoint` produces.** Every step in this blueprint
    runs its four fields in one fixed order — **Do → Done when → Verify → Checkpoint** — and the
    Checkpoint is last *on purpose*, so the tag points at a state the gate already proved. The
    consequence is absolute: at the moment a `Verify` block runs, **the step's own commit has not
    happened.** Its new files are untracked. The working tree is dirty by exactly the amount of work
    the step just did. Its tag does not exist. A gate that asserts otherwise fails on a **correct**
    step, on every machine, for a reason the builder cannot fix inside the step.

    Stated generally, because the shape recurs beyond version control: **a gate may not assert a
    post-condition of a later phase of the same step.** If the thing that makes the assertion true is
    scheduled after the assertion, the assertion is misplaced, not merely unlucky.

    The concrete family — each of these is a defect *when it sits in the `Verify` of the step that
    causes the state*:

    | Written in the step's own `Verify` | What actually happens |
    |---|---|
    | `test -z "$(git status --porcelain)"` in a step that authored files | The tree is dirty *because the step did its job*. Clean would mean the step built nothing. Exits 1, listing the step's own files. |
    | `git ls-files --error-unmatch LICENSE` where this step creates `LICENSE` | `error: pathspec 'LICENSE' did not match any file(s) known to git` — the file exists on disk but nothing has been `git add`ed yet. |
    | `git diff --quiet` / any "nothing uncommitted" assertion inside a step that writes code | Same defect wearing a different command. |
    | `git tag -l 'step-11-*' \| grep -q .` for the step's own tag | The tag is created two lines later, in the Checkpoint. |
    | `git show step-11-licensing:path/to/file` for the step's own artifact | The commit that tag would point at does not exist yet. |
    | Any gate reading a release, changelog entry, or artifact the Checkpoint publishes | Same shape: phase 4 output asserted in phase 3. |

    **Two legal fixes, and the first is preferred:**

    1. **Move the assertion into the `Checkpoint` block, after the commit.** Preferred, because the
       property genuinely *is* a post-commit property and belongs where it becomes true. The
       Checkpoint is a shell block like any other; it may carry assertions after its `git commit` and
       `git tag` lines, and an assertion there still fails the step if it fails.

       ```bash
       git add -A && git commit -m "step 12: license and versioning policy"
       git tag step-12-licensing
       git ls-files --error-unmatch LICENSE VERSIONING.md   # expect: exit 0 — now that they are committed
       test -z "$(git status --porcelain)"                  # expect: exit 0 — the commit above took everything
       ```

    2. **Restate the gate so it does not depend on commit state at all.** Assert the property the step
       is actually responsible for, in a medium available during `Verify` (§9 rule 12):

       | Depends on the Checkpoint | Independent restatement |
       |---|---|
       | `git ls-files --error-unmatch LICENSE` | `test -f LICENSE` — the file exists on disk |
       | `test -z "$(git status --porcelain)"` | `! git check-ignore -q LICENSE` — nothing will swallow it at commit time |
       | "the working tree is clean" | `diff -u expected/report.txt out/report.txt` — the output matches what this step promised, which is the real requirement |
       | `git show <own tag>:config.json` | `cat config.json \| jq -e '.name == "…"'` |

    **A third move is not legal: telling the builder to commit early.** "Run `git add -A && git
    commit` before this check" reorders the step's phases and destroys the invariant the Checkpoint
    exists to provide — a tag that points at a **verified** state. The Checkpoint is last in every
    step template in this document, and no step may quietly opt out.

    **Do not simply delete a useful check.** A tracked-files assertion is a reasonable thing to want:
    it is exactly what catches a forgotten `.gitignore` negation, and §10's *Files that must be
    committed* table exists because of that failure mode. It has two legal homes, both after the
    commit that makes it decidable:

    - **The step's own `Checkpoint` block**, per fix 1 — when the assertion is about files that step
      created.
    - **§20.1's manual gates**, which run after *every* step has committed — the right home for the
      whole-repository version ("every file §10 names is tracked in a clean checkout"). §20.1 already
      carries this line; add paths to it rather than inventing a per-step gate.

    **The mechanical self-check.** Grep every `Verify` block for git-state assertions — `git status`,
    `git ls-files`, `git diff --quiet`, `git tag -l`, `git show`, `porcelain`, `untracked`,
    `--error-unmatch`. For each hit, ask one question: **does the step this gate sits in create,
    modify, or delete any file named in it — or any file at all, for the whole-tree checks?** If yes,
    it is misplaced; apply fix 1 or fix 2. If no — the step only reads, and some earlier step
    committed the file — it is legitimate and stays.

15. **An expected output authored before its producer is reconciled twice — against this blueprint,
    and against the pinned runtime.** Writing the golden file, the byte-exact example, or the fixture
    in an early step, *before* the code that emits it exists, is a **good** practice: it makes the
    contract real instead of retrofitted, and rule 4 above prescribes exactly that move whenever the
    section that would have held the format does not apply. Keep doing it. But bytes authored ahead
    of their producer are a **prediction**, and no other rule in this document checks a prediction.
    Two reconciliations discharge it, and **both are checkable while you write** — neither needs the
    code to exist.

    **Reconciliation 1 — against every definition elsewhere in this blueprint that constrains the
    content.** Field semantics and path relativity from §4, the envelope and error codes from §5, key
    order, number and date formatting, units, casing, sort order, line endings, trailing newline.

    > *The observed failure was twelve lines wide.* §4 defined the field as *"path relative to the
    > run root"*, and §4's own example agreed. The golden file authored eleven lines later wrote a
    > **parent-directory prefix** into that same field. Both statements were in the blueprint;
    > nothing had read one against the other. The step that diffs real output against that golden
    > fails byte-for-byte on every machine, forever.

    **Reconciliation 2 — against the PINNED RUNTIME that will actually produce it.** Exception and
    parse-error wording, stack-trace shape, object key ordering, float formatting and rounding,
    locale and timezone rendering, sort collation — all of it is **runtime-version-specific**. A
    string written from memory, or copied from a version other than the one §11 pins, is a gate that
    fails on every machine forever and names the builder's code as the culprit.

    > *The observed failure embedded a parse-error message the pinned engine cannot emit.* The string
    > was the previous major's format; the pinned version emits two mutually exclusive message
    > families, neither matching — verified empirically across seventeen candidate inputs. The
    > builder's only escape was to **judge that the blueprint's own format was wrong**, which is
    > exactly the clarifying decision a self-contained blueprint forbids.

    **Reconciliation 2 is checkable at authoring time with a single command: run the producing call
    on the pinned runtime and read what it actually says.** One `JSON.parse` of a malformed string,
    one divide by zero, one date rendered, one object serialised with the real keys — executed on the
    pin from §11, not recalled. You have no shell, so the call goes to whoever does: **name the exact
    command and the literal it must produce in your return value, as a gap the main thread executes
    before this blueprint ships** — or do not write the literal at all. Never transcribe a
    runtime-produced string from memory into an artifact something will diff.

    **Record both reconciliations in §19.6's *Byte-exact artifact reconciliation* table**, which is
    where the golden files and fixtures are emitted and where the mechanical self-check lives. A row
    whose last two columns cannot be filled is a literal you may not write: replace the byte-exact
    comparison with a property the step can actually assert — a schema check, field-by-field
    assertions, or a diff normalised to drop the runtime-specific part — and say in the criterion
    that it *is* the property (rule 12).

    **Predicting the failure in §20.2 is not a fix.** A risk register that forecasts "the step-7 diff
    may fail" and an epic that states the repair procedure convert a hard block into a two-deviation
    repair — but the repair still requires the builder to decide which of two blueprint-authored
    facts to believe. Fix the bytes; do not document the wound.

16. **A gate must fail for the right reason.** Rule 11 makes a *correct* step exit 0. This rule makes
    an *incorrect* one exit non-zero **for the reason the gate claims**. They are the two halves of
    one property and they catch opposite defects: rule 11 catches a gate that can never pass, this
    one catches a gate that can never fail.

    **A command that errors on its own usage still exits non-zero.** Wrong arity, an unknown flag, a
    missing argument, a flag illegal in combination with another, a file it cannot open — every one
    of those exits non-zero *before the command ever evaluates the property*. So a gate whose pass
    condition is *"exits non-zero"* passes **vacuously**, and keeps passing after the thing it checks
    breaks.

    > *The observed failure:* a §20.1 manual gate ran `git check-ignore -q <pathA> <pathB>` to prove
    > two paths were not ignored. `-q` is legal only with a **single** pathname, so git exited **128**
    > for usage — never **1** for "no path matched an ignore rule". The gate's condition was
    > "non-zero", so it passed. It would have passed identically if both files *were* ignored, which
    > is the entire thing it existed to detect.

    | Passes vacuously | Fails for the stated reason |
    |---|---|
    | `! git check-ignore -q a b`  `# expect: non-zero` | `git check-ignore -q a; test $? -eq 1` and the same line for `b`  `# 1 = not ignored · 128 = usage, and now that fails` |
    | `! mytool validate config.json`  `# expect: invalid` | `mytool validate config.json; test $? -eq 1`  `# 1 = invalid · 2 = bad usage` |
    | `mytool --bad-flag; test $? -ne 0` | `mytool --bad-flag; test $? -eq 2` |
    | `! curl -f "$URL/missing"`  `# expect: 404` | `test "$(curl -s -o /dev/null -w '%{http_code}' "$URL/missing")" = 404` |

    **The rule:** any gate whose success condition is a **non-zero** exit must either **assert the
    specific code** the property produces, or be restructured so success is exit 0 and the property
    is read from *output* rather than from status. `!` and `test $? -ne 0` accept every failure
    equally — including the ones that mean your command was malformed — so neither is a gate. Where
    the tool documents no stable code, assert on its output (`grep -qx`, `jq -e`, a diff) and let
    that assertion decide the status.

    **The mechanical self-check.** Grep every `Verify` block and every §20.1 line for a leading `!`,
    for `test $? -ne`, and for any comment reading *expect: non-zero*, *expect: fails*, or *expect:
    error*. For each hit answer two questions: **which exit code does the property produce**, and
    **which codes does this command emit for usage errors** — arity, unknown flag, illegal flag
    combination, unreadable file. If those two sets overlap, or if you cannot name the first, it is
    not a gate. Then re-read the command's **arity and flag rules against its documentation**, since
    a vacuous gate is invisible to every audit that merely runs it and sees green.

### One step, one unit — the counting rule

**This subsection is the single source of truth for step counts and epic counts.**
`templates/epic-template.md` and `templates/tasks-schema.md` point here and assert no counts of
their own. If you are about to write a number of steps or epics anywhere else, link here instead.

> **One Section 9 step = one `tasks.json` task = one task block in an epic file.**

There is only ever one number. A blueprint with 14 steps has 14 tasks in `tasks.json` and 14 task
blocks distributed across its epic files — not 14 steps and 30 tasks. Epics are a *grouping* of
these same steps, not a finer subdivision of them. If you find yourself splitting a step into
sub-tasks while writing an epic, the step was too big — split it here in Section 9 first, renumber,
and keep the three views identical.

**Range: 10-18 steps, zero to deployed.** Under 10 steps usually means steps are too coarse to
finish in one sitting; over 18 usually means the v1 scope is too wide and Section 1's Non-Goals
table needs more rows. The range does not change with emission mode. Causality runs the other way:
the step count you write here *determines* the mode — 12 or more emits a bundle, 11 or fewer a
single file (`questions/phase-4-generate.md` Step 2). So never adjust a step count to reach a
packaging you prefer; split steps on build reality and let the mode fall out.

**The epic count is derived from the step count, never asserted.** One epic holds **5-9 steps**:
fewer than 5 and it is really part of its neighbour, more than 9 and it is two epics. Split on the
natural seam, which is almost always a layer boundary (data / server / interface) or a surface
boundary (public site / authed app / admin). Because both ranges are fixed, the step map decides how
many epics are legal:

> At least `ceil(steps ÷ 9)` epics, at most `floor(steps ÷ 5)`.

| Steps in §9 | Legal epic counts | Example splits |
|---|---|---|
| 10-14 | exactly **2** | 12 → 6 + 6 · 14 → 7 + 7 |
| 15-18 | **2 or 3** | 15 → 5 + 5 + 5 · 18 → 9 + 9, or 6 + 6 + 6 |

Write the step map first, then divide it. A bundle outside that table has a wrong split or a wrong
step count — fix the split, never the rule. There is no line budget on an epic file: an epic is as
long as its steps require, and `templates/epic-template.md` explains why it must not be compressed
by removing its repeated preamble.

### Step map

| # | Step | Depends on | Touches | Gate |
|---|---|---|---|---|
| 1 | {Scaffolding} | — | {files} | {the one command that proves it} |
| 2 | {…} | 1 | {files} | {…} |

{Ordering heuristic: scaffold → data layer → auth → the single highest-value vertical slice
end-to-end → remaining features → polish → hardening → deploy. Get one slice fully working before
broadening; a vertical slice validates the whole stack while it is still cheap to change.

**Then pull forward whatever closes a cross-artifact contract** (rule 13). If this project ships an
executable, a published entry point, a container, or a served endpoint, the step that first produces
it comes early enough that its own gate can *run* it — a stub whose only job is to print a version
string and exit 0 is a legitimate step 1 or 2. That single reordering turns a defect discovered seven
steps deep into one discovered at the first gate, and it costs nothing: the stub is replaced by real
behaviour in the step that was going to build it anyway.}

---

### Worked example — the pattern to follow exactly

{This is a real, fully-specified step. Every step you write matches this shape and this level of
precision. Delete this example from the generated blueprint and replace it with the project's own
steps — it is here so you can see the standard, not to be shipped.}

#### Step 7 — Stripe checkout and subscription webhook

**Do**
Wire paid signup end to end. Create:
- `src/lib/stripe.ts` — the SDK client, reading `STRIPE_SECRET_KEY`
- `src/app/api/checkout/route.ts` — creates a Checkout Session for the signed-in user
- `src/app/api/webhooks/stripe/route.ts` — signature-verified webhook receiver, raw-body parsing
- `src/lib/billing/sync-subscription.ts` — the single writer to `subscriptions`
- migration `007_subscriptions.sql` — `subscriptions` + `webhook_events` (dedupe ledger)
- `src/app/api/webhooks/stripe/route.test.ts`

**Done when**
- [ ] WHEN a POST arrives at `/api/webhooks/stripe` with an invalid `Stripe-Signature` header THE SYSTEM SHALL respond `400` and write zero rows to `subscriptions`.
- [ ] WHEN `checkout.session.completed` is received for a known customer THE SYSTEM SHALL upsert exactly one `subscriptions` row with `status='active'` and a non-null `current_period_end`.
- [ ] WHEN the same Stripe event `id` is delivered twice THE SYSTEM SHALL return `200` both times and leave the `subscriptions` row count unchanged.
- [ ] WHEN `customer.subscription.deleted` is received THE SYSTEM SHALL set that row's `status='canceled'` and SHALL NOT delete the row.
- [ ] WHEN an unhandled event type arrives THE SYSTEM SHALL return `200` and log at `info` — never a 5xx, because a non-2xx makes Stripe retry for days.
- [ ] WHEN `STRIPE_WEBHOOK_SECRET` is unset at boot THE SYSTEM SHALL fail startup with a named error, not serve traffic that silently accepts unsigned payloads.
- [ ] WHEN the webhook test suite runs THE SYSTEM SHALL report 6 passing tests and 0 skipped.

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

curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  localhost:3000/api/webhooks/stripe \
  -H 'Stripe-Signature: t=0,v1=deadbeef' -d '{}'
# expect: 400
```

**Checkpoint**
```bash
git add -A && git commit -m "step 7: stripe checkout + subscription webhook"
git tag step-07-billing
# rollback target if step 8 goes wrong: git reset --hard step-07-billing
```

---

### Step template

{Repeat this block for every step in the step map.}

#### Step {N} — {Short imperative title}

**Do**
{What gets built and the exact files created or modified — including the tests this step's `Verify`
runs, which are part of the step, not a by-product of it. Reference the sections above by number
rather than restating them — "the `orders` entity from §4", "the envelope from §5", "the pinned
version of the SDK from §11". Never restate a version pin inside a step; §11 is the only place a
version appears, and every package this step imports has an install command here or in §10.

**Never point a step at a section you marked `NOT APPLICABLE`.** A reference is only load-bearing if
the referent has content: "the format defined in §7" is an instruction when §7 defines a format and
an invitation to invent one when §7 says `NOT APPLICABLE`. If this step needs a contract — an output
format, a schema, a fixture, a wire shape — that contract exists concretely somewhere before this
step reads it: a fenced block in the step itself, a file emitted in §19.6, or a table in a section
that does apply. Check every `§N` you write in this step against what §N actually contains.

**If this step authors expected output that a later step diffs — a golden file, a fixture, a
snapshot baseline — write the literal bytes here, and reconcile them twice before you do** (rule 15):
against every blueprint rule that constrains their content, and against the §11-pinned runtime that
will actually produce them. Give the artifact a row in §19.6's *Byte-exact artifact reconciliation*
table. Authoring the contract ahead of the producer is correct; shipping it unchecked blocks every
step downstream of the diff.

Never invent the name of a file a generator produces — migrations, codegen output, lockfiles. Tools
name those themselves, usually with a hash or a random suffix, so a step that says
`0006_reservations.sql` names a file that will not exist. Write "the migration `db:generate` emits"
and describe what must be in it.}

**Done when**
- [ ] WHEN {trigger} THE SYSTEM SHALL {observable response}
- [ ] WHEN {trigger} THE SYSTEM SHALL {observable response}
- [ ] {…3-7 conditions. Include at least one failure/edge case, not only the happy path.}

**Verify**
```bash
{literal commands, each with the expected result in a trailing comment.

Nothing here may depend on this step's own Checkpoint (rule 14). At this moment the step's files are
written but untracked, the tree is dirty, and the tag does not exist — so no `git status --porcelain`
clean check, no `git ls-files --error-unmatch` over a file this step created, no `git diff --quiet`,
no lookup of this step's own tag. Assert the file on disk instead, or move the assertion below.

No line here may pass on *any* non-zero exit (rule 16). If the correct outcome of a command is a
failure, assert the specific code — `{cmd}; test $? -eq {N}` — so a usage error (wrong arity, unknown
flag, bad combination) fails the gate instead of satisfying it. A bare `!` or `test $? -ne 0` is not
a gate.

If a line here diffs real output against a literal this blueprint authored earlier, that literal has
a row in §19.6's *Byte-exact artifact reconciliation* table, checked against both the blueprint rules
that constrain it and the §11-pinned runtime that produces it (rule 15).}
```

**Checkpoint**
```bash
git add -A && git commit -m "step {N}: {slug}"
git tag step-{NN}-{slug}
{Any assertion about committed state belongs HERE, after the commit that makes it true — tracked
files, a clean tree, the tag resolving. Each with its expected result in a trailing comment.}
```

---

### 9.1 Parity and cutover

{**Required whenever this blueprint describes a migration** — replacing a running system, moving
between frameworks, changing a database, or splitting/merging services. A greenfield build has
nothing to cut over from: write exactly `NOT APPLICABLE — greenfield build, no system is being
replaced.` and continue to Section 10. Do not delete the heading.

A migration fails in a way a greenfield build cannot: the new thing works, the old thing was doing
something nobody wrote down, and the difference is discovered by users. Sections 1 and 5 record what
must not change; this subsection records how that is *proved* before the old path is switched off.}

#### Parity set

{The behaviors the new implementation must reproduce exactly, each with the check that proves it.
Derived from the "Interfaces held constant" rows in §1's Non-Goals table and the frozen contracts in
§5. Every row is machine-checkable — a human comparing two screens is not parity evidence.}

| # | Behavior held constant | How parity is proved | Tolerance |
|---|---|---|---|
| 1 | {endpoint / query / export / calculation} | {the shadow-run diff, golden-file test, or replay command} | {exact match, or the named acceptable delta and why} |

**Shadow period:** {how long both systems run side by side, what traffic the new path sees, and
where the diffs are recorded. State the diff rate that counts as parity — "zero diffs over N
requests across M days" — and what a non-zero diff blocks.}

#### Cutover

| Phase | What changes | Who is affected | Reversible by | Verify |
|---|---|---|---|---|
| {dark launch} | {new path deployed, receives no user traffic} | {nobody} | {feature flag off} | `{command}` |
| {canary} | {N% of traffic} | {that cohort} | {flag → 0%} | `{command}` |
| {full} | {100%} | {everyone} | {flag → 0%, old path still deployed} | `{command}` |
| {decommission} | {old path deleted} | {everyone} | {redeploy from `{tag}` — no longer instant} | `{command}` |

**The kill switch:** {the exact mechanism and command that returns 100% of traffic to the old path,
and how long it takes. If reverting requires a deploy, say so and state the deploy time — an
"instant rollback" that takes twelve minutes is a twelve-minute outage.}

#### Abort criteria

{The observable conditions that stop the cutover and trigger the kill switch, decided **before** the
cutover starts so nobody is negotiating thresholds during an incident. Each is a number a monitor
can evaluate, wired to the alerts in §16.}

- [ ] WHEN {the parity diff rate exceeds {N}%} THE SYSTEM SHALL {revert to the old path and hold}.
- [ ] WHEN {p95 latency on {endpoint} exceeds {N}ms for {N} minutes} THE SYSTEM SHALL {revert}.
- [ ] WHEN {error rate exceeds {N}× the pre-cutover baseline} THE SYSTEM SHALL {revert}.

#### Data migration

{If data moves: the backfill command, whether it is resumable and idempotent, how the two stores
stay consistent during the shadow period (dual-write, CDC, or read-through), the reconciliation
query that proves row counts and checksums match, and the point of no return — the moment after
which the old store is no longer authoritative. Name that moment explicitly; it is the highest-risk
line in the whole build. If no data moves, write `NOT APPLICABLE — no data migration.`}

#### Decommission

{What gets deleted, when, and what must be true first — the retention window on the old data, the
final backup and where it lives, and the DNS/credential/vendor-account cleanup. A migration that
never decommissions leaves two systems to maintain forever, which is worse than not migrating.}

**Decommission is never a build step in §9.** It happens after a soak period that outlasts the
build, so it belongs on the post-build launch checklist. The build gate is reaching full cutover
with the old path still deployed and revertible.

---

## 10. Environment Setup

### Prerequisites
| Tool | Version | Check |
|---|---|---|
| {tool} | {from the runtime track} | `{command that prints the version}` |

### Accounts to create first
{Every third-party service the build needs, with the signup URL and which step first requires it.
The builder should create all of them before step 1, not discover them mid-build.}

### Environment variables
| Variable | Purpose | Where to get it | Required by step | Secret? |
|---|---|---|---|---|
| `{VAR}` | {what it does} | {URL or path in the provider console} | {N} | {yes/no} |

`.env.example` is committed with every key present and every value blank or obviously fake.
`.env` and `.env.*.local` are gitignored. The app validates required env vars at boot and fails
loudly — never falls back to a default for a secret.

**"Required by step" is a contract with §9, not a note.** The boot validator treats a variable as
required only from the step named in that column and as optional before it, so that no step's
arrival breaks an earlier step's gate (§9 rule 9). Include every variable a `Verify` command needs,
not only the ones the product needs — the local service URLs from §19.6 belong in this table with
their literal local values.

**Listing a variable here does not load it.** The application framework reads `.env` for the
application; nothing reads it for a standalone CLI, a migration tool, a seed script, or a test
runner invoked outside the framework. Every such tool this blueprint's commands invoke must be
given an explicit loading mechanism — see §19.6, *Every tool that reads env vars must be given a
way to load them*, which is where that mechanism is written into the emitted config.

### Files that must be committed

{Every file this blueprint describes as *committed* — `.env.example`, the emitted §19.6 configs, a
lockfile, a CI workflow, seed fixtures, `.claude/` — is listed here, and for each one this section
states that the ignore file does **not** exclude it.

**Scaffolders ship broad ignore patterns and they will swallow these.** A generated ignore file
routinely carries `.env*`, `*.local`, `.claude/`, `*.config.*` or a bare `dist` glob, so a blueprint
that says "`.env.example` is committed" in four places and never touches the ignore file has told
the truth about intent and shipped the opposite. The failure is silent: everything works on the
machine that generated it, and the clean-checkout premise of §20.1 quietly stops holding — the
acceptance gate runs against a tree the builder never actually has.

Write the ignore-file exception as a literal line, in the ignore file's own syntax, in the Bootstrap
block below — a negation (`!.env.example`) placed **after** the pattern it overrides, or the removal
of the offending pattern. Prose ("make sure `.env.example` is committed") is not an exception.

**And the ignore file itself, with these exception lines in it, exists before Bootstrap's first
commit** — see *The ignore file precedes the first commit* below. An exception line written into a
`.gitignore` that a §9 step delivers after the first commit corrects nothing: the paths it was meant
to govern were already tracked, and tracked paths ignore the ignore file.}

| File | Why it is committed | Ignore-file exception line |
|---|---|---|
| `.env.example` | {reason} | `!.env.example` after the `.env*` pattern |
| {file} | {reason} | {literal line, or `— not matched by any ignore pattern`} |

### Bootstrap
```bash
{Exact commands from clone to a running dev server with seeded data, in the order they are run.
Copy the scaffolding block from the runtime track verbatim, then add: **the version-control
initialisation §9's checkpoints require** (see below), starting the local services from §19.6, any
post-install approval or setup step the package manager requires, any runner asset download (browser
binaries, native toolchains), the ignore-file exception lines from the *Files that must be committed*
table above, and the migration + seed commands.

**Bootstrap must create what the checkpoints need.** Every §9 step ends in a `git tag` Checkpoint and
§20.1's manual gate counts those tags, so the repository has to exist before step 1 — and this block
is the only thing that runs before step 1. **Do not assume the scaffolder created it.** Scaffolders
initialise a repo only sometimes, skip it when they detect an enclosing one, and abort the
initialisation entirely on any of the prompts and failure modes this blueprint already documents —
in which case the first Checkpoint dies on `not a git repository` and every later rollback target is
gone. Write the initialisation explicitly and idempotently, before the first command that could
create a Checkpoint-worthy file — two literal lines, in this block, in this order:

    git rev-parse --git-dir >/dev/null 2>&1 || git init -b main   # idempotent: no-op if the scaffolder already did it
    git add -A && git commit -m "chore: scaffold" --allow-empty   # a tag needs a commit to point at

Adapt the syntax to whatever version-control system §9's Checkpoints use; the obligation is the
same — **if the build order uses version-control checkpoints, this block creates the repository and
the first commit.** If §9 uses no checkpoints at all, say so here explicitly rather than leaving the
question open.

**The ignore file precedes the first commit.** Those two lines create a commit, and `git add -A`
takes **everything on disk at that moment**. Any path the ignore rule was written to exclude that
already exists is tracked from then on — and **an ignore rule never applies to a path git already
tracks.** No later edit to the ignore file un-tracks it; nothing errors; every subsequent `git add
-A` keeps it.

> *The observed failure:* the blueprint delivered `.gitignore` in a **§9 step**, while this block's
> first commit ran before step 1. **19 files** the ignore rule was meant to exclude went into the
> first commit and stayed there for the whole build. Both artifacts were correct in isolation; only
> their order was wrong.

So the ignore file is delivered **here, in this block, before the first commit** — written inline,
emitted under `workspace/` and landed by the guarded copy above, or produced by a scaffolder line
earlier in this same block — together with every exception line from the *Files that must be
committed* table. **A §9 step may not be the first place the ignore file appears.** A later step may
tighten it; that is an edit to a file that already exists, and the step says so.

**Generalise past version control: any file whose *purpose* is to change what a later command sees
must be in place before the first command whose behaviour it governs.** Ignore files before the
first commit, `.dockerignore` before the first image build, formatter/linter config and its exclude
lists before the first `format --check`, the env file before the first tool that reads it, workspace
globs before the first install. **State the ordering out loud in this block** rather than leaving it
implied by line order — a reader cannot tell a deliberate sequence from an accidental one, and the
next person to add a line has nothing to place it against. One comment line does it:

    # order matters: ignore file + exceptions → repo init → first commit → install → services → migrate → seed

**Every command here must be safe to run twice.** Bootstrap is the first thing a stuck builder
re-runs, so treat re-running it as a supported path, not an accident: guard the version-control
initialisation as shown above, use the idempotent form of every scaffold and migration command, and
— most importantly — **guard the `workspace/` copy** (§19). An unguarded recursive copy over an
already-bootstrapped tree silently reverts every file §19.6 emitted, and if the package manifest is
one of them the tree loses every dependency the build installed. The failure surfaces one command
later as a missing binary, which reads as a broken install, so the builder reinstalls tooling
instead of restoring the manifest. Write the copy in its non-clobbering form and put the reason in a
trailing comment.

**And a guard must exit 0 on the path it guards against** (§19). A no-clobber copy that returns
non-zero when it skips a file — `cp -Rn` does exactly this on BSD/macOS — turns the second run into
an aborted run under `set -e`, which is the failure the guard was written to prevent. Read every
guarded line here and ask what status it exits with when the thing it guards is already there;
anything other than 0 gets the portable form or an explicit `|| true` with the reason beside it.

Every command here must be non-interactive. A command that opens a TTY prompt hangs an unattended
build forever, which is indistinguishable from a slow one — pass the flag that answers it, and if
no such flag exists, replace the command with one that writes the file it would have generated.

**This block gets executed, verbatim, before the blueprint is presented — by the main thread, in a
scratch directory** (`questions/phase-4-generate.md` Step 6). You are not the one who runs it: you
have no shell. Write it so it survives that run, because whatever it does there is what the builder
gets. A command that fails, hangs on a prompt, or leaves the directory in a state step 1 cannot use
comes straight back to you as a finding, with the real error attached.

Scaffolding tools ignore flags, abort on approval prompts, and install things they were told not to;
every one of those is a first-gate failure and none of them are visible from reading the docs. That
is the whole reason the block is run rather than reviewed.}
```

---

## 11. Dependencies

**This section is the version-provenance table. It is the only place in the blueprint's prose where
a version number appears** — the one exception being the executable files emitted in §19, which must
carry real values (an image tag in a compose file, an engine field in `package.json`). Those get a
row here too; the file carries the value, this table carries the provenance.

{Every row's `Version`, `Source` and `Checked` cells come verbatim from the `stack-researcher` report
produced in this session — that report is the authority. For a package the researcher did not
resolve, fall back to `knowledge/runtime-tracks/{file}.md`, put that file's path in `Source`, its
`Last verified` date in `Checked`, and carry any unverified caveat through into `Purpose` rather
than dropping it. **Never write a pin from memory.**

A row with an empty `Source` or an empty `Checked` fails the validator — a pin that contradicts its
own stated provenance is a BLOCKER. If a version genuinely could not be verified, write
`UNVERIFIED — verify before install` in `Source` and say so; an honest gap is fine, an implied
verification that never happened is not.

**Every row must be traceable to the step that installs it.** The `Installed by` cell names the §10
Bootstrap block or the §9 step number whose **Do** list carries the install command for that exact
package — and that command must literally exist there. A pin nobody installs is not a dependency,
it is a note: the builder reaches the step that imports the package, the import fails, and the
version this table so carefully verified was never applied.

**This is your obligation to check while writing, not a box the validator will tick for you.** Do it
mechanically, before you consider §11 finished: for each package name in these tables, search §10's
Bootstrap block and every §9 step's install commands. Zero hits means one of the two places is
wrong — either add the install to the step that first imports the package, or delete the row.
`blueprint-validator` carries a finding for exactly this shape (a §11 row whose package appears in no
install command anywhere in the blueprint), so an untraceable row comes back to you as a FAIL and
costs a full round trip. Catching it here costs one grep.}

### Runtime
| Package | Version | Source (registry URL or track file) | Checked | Installed by | Purpose |
|---|---|---|---|---|---|
| {package} | {pin} | {https://registry.npmjs.org/{package} — or `knowledge/runtime-tracks/{file}.md`} | {YYYY-MM-DD} | {§10 Bootstrap, or step N} | {why it's here — a package with no stated purpose gets removed} |

### Development
| Package | Version | Source (registry URL or track file) | Checked | Installed by | Purpose |
|---|---|---|---|---|---|
| {package} | {pin} | {source} | {YYYY-MM-DD} | {§10 Bootstrap, or step N} | {purpose} |

### Deliberately not used
| Rejected | Instead | Why |
|---|---|---|
| {package} | {what we use} | {reason — stops the builder from adding it back} |

---

## 12. Deployment Strategy

### Hosting
{Platform, region, plan tier, and why. Include the build command, output directory, and runtime
setting so the builder configures it in one pass.}

### Environments
| Environment | Branch | URL | Database | Third-party mode |
|---|---|---|---|---|
| Local | — | localhost | {local/branch db} | {test keys} |
| Preview | {any PR} | {auto} | {branch db} | {test keys} |
| Production | `main` | {domain} | {prod db} | {live keys} |

### CI/CD
{Pipeline stages in order, with the exact commands. The pipeline runs the same acceptance gate as
Section 20 — if a check is in the gate it is in CI, no exceptions.}

### Release and rollback
{How a deploy is promoted, how a bad deploy is reverted, and how long that takes. Migrations:
state the ordering rule relative to code deploys.}

### Domain, DNS, TLS
{Records to create, certificate handling, redirect rules (apex ↔ www).}

---

## 13. Testing Strategy

{From `knowledge/capabilities/testing.md`. Tests exist to make Section 9's "Done when" conditions
checkable — write the ones the gates need, not a coverage number for its own sake.}

| Layer | Framework | What it covers | Where | Runs |
|---|---|---|---|---|
| Unit | {framework} | {pure logic, validators, formatters} | {path} | {every commit} |
| Integration | {framework} | {API routes against a real test database} | {path} | {every commit} |
| E2E | {framework} | {the critical flows below} | {path} | {pre-deploy} |

### Critical flows to cover E2E
1. {Flow — the one that costs money or trust when it breaks}
2. {Flow}
3. {Flow}

### Test data
{How a test database is created, seeded, and reset between runs. Tests never share mutable state
and never depend on execution order.

If any layer above runs against a real service rather than a fake, the file that provisions it
locally and the variable that points at it are emitted in §19.6 — name them here. "Integration tests
hit a real database" is a requirement, and a blueprint that states it without shipping the database
turns every integration gate into an unrunnable command.}

### What is deliberately not tested
{Name it. An untested area chosen on purpose is a decision; an untested area by accident is a bug.}

---

## 14. Security & Secrets

| Concern | Control | Implemented in |
|---|---|---|
| Secret storage | {manager / platform env, never in the repo} | {where} |
| Secret rotation | {procedure and cadence} | {where} |
| Input validation | {schema library, applied at every boundary} | {where} |
| Output encoding / XSS | {mechanism} | {where} |
| SQL injection | {parameterized queries only — no string-built SQL} | {where} |
| AuthN / AuthZ | {see §8 — server-side on every request} | {where} |
| CSRF | {mechanism} | {where} |
| Rate limiting / abuse | {limits, backend, what happens on trip} | {where} |
| Webhook verification | {signature check + replay/dedupe ledger} | {where} |
| Dependency audit | {command, cadence, who fixes} | CI |
| Security headers | {CSP, HSTS, X-Content-Type-Options, Referrer-Policy — literal values} | {where} |
| PII handling | {what is stored, where, retention, deletion path} | {where} |
| Logging hygiene | {never log secrets, tokens, full PII — the redaction mechanism} | {where} |

**Hard rules**
- No secret is ever committed, printed in a log, sent to an error tracker, or embedded in a
  client bundle. Anything reaching the browser is public — treat it that way.
- All server-side authorization checks run before the work, not after.
- Third-party webhooks are verified by signature before their body is parsed as trusted.

{If the project handles regulated data (health, financial, children's, EU personal data), name the
regime and the specific obligations it creates for this build. If it does not, say so explicitly.}

---

## 15. Accessibility

**Target: WCAG 2.2 Level AA.** This is not polish — in the EU, UK, US public sector, and a growing
set of private-sector jurisdictions it is a legal gate, and retrofitting it costs several times what
building it in costs. Accessibility criteria belong in Section 9 "Done when" lists, not in a
someday backlog.

### Baseline requirements
| Requirement | Rule |
|---|---|
| Semantic HTML | Landmarks (`header`/`nav`/`main`/`footer`), one `h1` per page, headings in order, lists for lists |
| Keyboard | Every interactive element reachable and operable by keyboard; logical tab order; no traps; skip-to-content link |
| Focus visible | A visible focus indicator on every focusable element, ≥3:1 against its background |
| Contrast | Text 4.5:1, large text and UI boundaries 3:1 — the §7 palette already satisfies this |
| Forms | Every input has a programmatic label; errors are text, not color alone; errors are announced |
| Images | Meaningful images have alt text; decorative images have `alt=""` |
| Motion | Everything animated respects `prefers-reduced-motion: reduce` |
| Zoom / reflow | Usable at 200% zoom and at 320 CSS px wide without horizontal scrolling |
| Live regions | Async status changes announced via `aria-live` or a role that implies it |

### WCAG 2.2 additions — the ones most often missed
| SC | Requirement |
|---|---|
| 2.4.11 Focus Not Obscured (Min) | A focused element is never fully hidden behind sticky headers, cookie bars, or toasts |
| 2.5.7 Dragging Movements | Any drag interaction has a single-pointer alternative (buttons, menu action) |
| 2.5.8 Target Size (Min) | Pointer targets are at least 24×24 CSS px, or adequately spaced |
| 3.3.7 Redundant Entry | Information already given in a multi-step flow is auto-filled or selectable, not retyped |
| 3.3.8 Accessible Authentication (Min) | No cognitive-function test with no alternative — allow password managers, never block paste |

### Verification
```bash
{The automated a11y command — e.g. an axe run in the E2E suite. Expect: 0 violations.}
```
Automated checks catch roughly a third of real issues. Add these manual passes before launch:
keyboard-only traversal of every critical flow, one screen-reader pass over the primary flow, and a
200% zoom pass on the narrowest breakpoint.

---

## 16. Observability & Cost

### Instrumentation
| Signal | Tool | What it captures | Who looks at it |
|---|---|---|---|
| Errors | {tool} | {unhandled exceptions, with release + user context, PII redacted} | {who} |
| Logs | {tool} | {structured JSON, with a request id on every line} | {who} |
| Metrics | {tool} | {the four below} | {who} |
| Uptime | {tool} | {which endpoint, from where, how often} | {who} |

### The metrics that matter for this project
{4-6 only. Each with a target and an alert threshold. Generic dashboards nobody reads are worse
than none — pick the numbers that would make you change something.}

| Metric | Target | Alert at |
|---|---|---|
| {p95 latency on the critical endpoint} | {value} | {value} |
| {error rate} | {value} | {value} |
| {the one business metric} | {value} | {value} |

### Health check
{Path, what it actually verifies (database reachable, migrations current — not just "returns 200"),
and who polls it.}

### Cost model
| Service | Free tier | Cost at {expected v1 scale} | Cost at 10× | Cliff to watch |
|---|---|---|---|---|
| {service} | {limit} | {$/mo} | {$/mo} | {the usage line where price jumps} |

**Estimated monthly cost at launch: {$X}.** {Name the single largest line item and the cheapest
lever for cutting it. If any service scales superlinearly with usage, say so here — surprise bills
kill more small projects than outages.}

---

## 17. Model Routing

{**Include this section with real content only if the project calls an LLM at runtime.** If it does
not, write exactly: `NOT APPLICABLE — this project does not call an LLM at runtime.` and move on.
Do not delete the heading; section numbers are fixed.}

**Before writing any model ID, price, context limit, or API parameter, invoke the bundled
`claude-api` skill and copy from it.** Never hand-maintain a model table from memory — model IDs
and their parameter rules change, and a wrong ID is a runtime failure with a confusing error.

### Routing table
| Task in this product | Model tier | Why this tier | Fallback |
|---|---|---|---|
| {high-stakes reasoning, code generation, agentic loops} | {tier} | {reason} | {tier} |
| {everyday generation, summarization} | {tier} | {reason} | {tier} |
| {high-volume classification, extraction, routing} | {cheapest capable tier} | {reason} | {tier} |

### Prompt and context strategy
{Where prompts live as files, how they are versioned, what goes in the system prompt vs the turn,
and the caching strategy for the stable prefix.}

### Cost controls
{Per-user and global spend caps, token budgets per call, what happens when a cap is hit, and the
metric that tracks cost per active user. Wire this into §16.}

### Failure handling
{Timeouts, retries with backoff, what the user sees on degradation, and the rule for refusals and
truncated output. An LLM call is a network call to a nondeterministic service — treat it like one.}

### Evaluation
{The fixed set of inputs with expected properties that runs before any prompt or model change ships.
Without it, prompt edits are unfalsifiable.}

---

## 18. Skills to Use During Build

{Every row **must** carry a verbatim install command — the builder's machine is not the designer's,
and a blueprint that names an uninstallable skill is not self-contained. Copy names, invocation
form, and install commands from `knowledge/skills-registry.md`. Never invent either.

A leading `/` means it really is a slash command. **No slash means it auto-activates** — name it in
prose. Writing a slash form for an auto-activating skill is a silent no-op, and the build step
quietly skips.

Never hard-depend on a skill. If one is unavailable the builder falls back to the blueprint's own
guidance, notes the fallback in one line, and continues.}

| Skill | Build steps | Why | Install |
|---|---|---|---|
| {skill} | {step numbers from §9} | {what it contributes there} | {verbatim command} |

---

## 19. Agent Workspace

{The target project ships with its own agent configuration. This is not one CLAUDE.md — it is a
workspace, because a single flat instruction file stops being read as it grows and cannot express
path-scoped rules.

**Single-file mode:** emit each artifact below as a fenced block for the builder to write by hand.
**Bundle mode:** write them as real files under `workspace/` in the bundle:

```
./blueprints/<project-slug>/workspace/
├── CLAUDE.md                    # §19.1
├── AGENTS.md                    # §19.2
├── <verify-critical config>     # §19.6 — test runner, e2e runner, compose file, test setup
└── .claude/
    ├── settings.json            # §19.3
    ├── skills/<name>/SKILL.md   # §19.4
    └── rules/<name>.md          # §19.5
```

`workspace/` mirrors the target repo layout exactly, so the builder's first move is one copy — and
the whole agent configuration *and* every config file the gates need is in place. Nothing else in
the bundle gets copied into the project.

**The workspace copy must be safe to re-run, and this blueprint writes the guard.** Re-running
bootstrap is the most natural recovery action a stuck builder has, and a bare `cp -R workspace/.
<project-root>/` over an already-bootstrapped tree overwrites every file the build has since
changed. **The expensive case is the package manifest**, which §19.6 emits: the copy reverts it to
the dependency-free version it had before install, taking every dependency entry with it. Nothing
reports an error at that moment. The *next* command fails naming a missing binary — which reads as a
broken install, not a clobbered manifest — so the builder reinstalls tooling, gets the same failure,
and burns the step on the wrong problem.

Write the copy so a second run is a no-op on anything already present, in one of these forms, and
put the reason in a trailing comment so nobody "simplifies" it back:

| Guard | Command shape | Use when |
|---|---|---|
| Copy only what is missing, portably | `rsync -a --ignore-existing workspace/ <project-root>/`  `# skips existing files and exits 0 either way` | `rsync` is available — the form with no exit-code surprise |
| Copy only what is missing | `cp -Rn workspace/. <project-root>/ \|\| true`  `# -n: never clobber a file the build has since changed. BSD/macOS cp exits 1 when it skips; skipping is the intended outcome` | the platform's `cp` supports `-n` |
| Gate on a marker | `[ -e <project-root>/.workspace-applied ] \|\| { cp -R workspace/. <project-root>/ && touch <project-root>/.workspace-applied; }` | portability matters, or the copy must happen exactly once |
| Copy, then re-derive | copy unconditionally, then re-run the install/regenerate command that rebuilds whatever the copy overwrote | the overwritten files are all machine-generated |

Whichever form you choose, name in one line **which files are deliberately never overwritten** —
the package manifest and the lockfile, at minimum, once anything has been installed.

**A guard must not itself fail.** The guard exists so the block is safe to run twice; a guard that
exits non-zero on exactly the path it guards against destroys that property under `set -e`, which is
how every unattended runner executes these blocks. **`cp -Rn` is the live example: on BSD/macOS it
exits 1 when it skips an existing file**, so the second bootstrap run — the entire reason the guard
was added — aborts at the copy. GNU `cp -n` exits 0 in the same situation, so the same line passes on
one platform and fails on the other, and neither is visible from reading it.

So check both halves of every guarded command in this block, not only the copy — conditional creates,
idempotent migrations, `mkdir`, marker checks, `grep`-based tests:

1. **The guarded path exits 0.** Second run, the thing already exists, nothing to do → status 0. If
   the command cannot promise that, neutralise it explicitly (`… || true`, with the reason in the
   comment) or pick a form that can.
2. **The exit codes are the same on every platform this build targets.** Where they differ, either
   write the portable form or state which platform the block assumes. "It works on my shell" is the
   same class of claim as "it works in the app".

**Everything emitted here must pass the project's own gates.** The lint and format config the
blueprint tells the builder to generate applies to these files the moment they land — the copy in
§19 happens before step 1, so a space-indented `settings.json` under a tab-default formatter fails
`lint` on the builder's first command. Match the emitted files to the conventions the blueprint
mandates, or the blueprint's first instruction breaks the blueprint's first gate.

**`.claude/commands/` is NEVER emitted — not here, not in the bundle, not in either mode.** A slash
command only fires when a human types it, and an autonomous builder types nothing, so a scaffolded
command is dead weight that is never invoked once. Every repeatable project workflow goes in
`.claude/skills/<name>/SKILL.md` (§19.4), which activates on intent — the only trigger a headless
build actually has. If you are about to write a `commands/` path anywhere in this blueprint, you are
writing a defect.}

### 19.1 `CLAUDE.md`

{Under 200 lines. Dense and scannable. Follow `templates/claude-md-template.md`. Commands first —
the builder needs to know how to run things before anything else. Explain how things connect, not
just where files sit. Every rule is specific and checkable: "max 300 lines per component" is a rule;
"keep files short" is a wish.}

```markdown
{Complete CLAUDE.md content}
```

### 19.2 `AGENTS.md`

{A short bridge file so agent tools that do not read CLAUDE.md still get the essentials. Do not
duplicate CLAUDE.md — 15-40 lines: what the project is, the commands, the three rules that matter
most, and a pointer to CLAUDE.md as the source of truth.}

```markdown
{Complete AGENTS.md content}
```

### 19.3 `.claude/settings.json`

{Pre-approve the project's own read-only and verification commands. Every command that appears in a
Section 9 `Verify` block belongs in `permissions.allow` — otherwise the builder stalls on a
permission prompt at every gate, which is exactly where an unattended build dies. Deny the things
that should never be automatic: reading `.env`, pushing, force-pushing, destructive database
commands.

The skeleton below is the starting set, not the finished file. Add one `allow` entry for every
remaining §9 `Verify` command — migrations, seeds, the service up/down commands from §19.6, project
CLIs, `curl` to localhost — and replace the placeholder deny row with this stack's real destructive
commands. **Emit valid JSON: every array element is a permission string. A sentence of prose inside
the array is not a comment, it is a rule that silently never matches** — unlike a half-filled prose
section, a half-substituted JSON file fails invisibly at runtime.}

```json
{
  "permissions": {
    "allow": [
      "Bash({pm} test:*)",
      "Bash({pm} typecheck)",
      "Bash({pm} lint:*)",
      "Bash({pm} build)",
      "Bash({pm} dev:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git tag:*)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Bash(git push:*)"
    ]
  }
}
```

### 19.4 Project skills — `.claude/skills/<name>/SKILL.md`

{One skill per repeatable project workflow — the things a builder or maintainer will do more than
once. Typical set: `add-migration`, `add-api-route`, `add-component`, `release`. Each is a procedure
with its own verification step, not a description.

Every SKILL.md needs YAML frontmatter with `name` and a `description` that says *when to use it* —
the description is the only thing loaded until the skill triggers, so a vague one never fires.}

````markdown
---
name: {skill-name}
description: {What it does and exactly when to use it — the trigger conditions, in the words a
  developer would actually type. Under 400 characters.}
---

# {Skill Name}

## When to use
{Concrete triggers.}

## Steps
1. {Step}
2. {Step}

## Verify
```bash
{command}   # expect: {result}
```

## Do not
- {The mistake this skill exists to prevent.}
````

{Repeat per skill. List them here:}

| Skill | Triggers on | What it automates |
|---|---|---|
| {name} | {phrase} | {procedure} |

### 19.5 `.claude/rules/*.md`

{Path-scoped conventions. A rule file applies only to the files matching its `paths` globs, so the
agent gets the database conventions when editing the data layer and the component conventions when
editing UI — instead of one giant file where everything competes for attention.}

```markdown
---
paths:
  - "{glob}"
  - "{glob}"
---

# {Rule set name}

- {Specific, checkable convention.}
- {Specific, checkable convention.}
```

{Emit one file per boundary. Suggested set for this project:}

| File | `paths` globs | Covers |
|---|---|---|
| `.claude/rules/{name}.md` | {globs} | {conventions} |

### 19.6 Verify-critical config and local infrastructure

{**Every config file a Section 9 `Verify` command needs in order to run is emitted here as a real
file, with complete content** — exactly the way `CLAUDE.md` and `settings.json` are. Naming it in
§3's tree is not emitting it. This subsection is what decides whether the gates can execute at all,
so fill it before you consider Section 9 finished.

**Bundle mode:** real files under `workspace/`, at the path they occupy in the project.
**Single-file mode:** one fenced block per file below, each labelled with its destination path.
No placeholders, no "configure as needed" — the builder runs these, it does not finish them.

| Emit | Whenever |
|---|---|
| Test-runner config | any `Verify` invokes the unit/integration runner |
| E2E-runner config | any `Verify` invokes the e2e runner |
| Test setup / env-bootstrap file | any test imports a module that validates env at import time |
| Path-alias config | any test or source file resolves an alias the runner does not inherit |
| Local service provisioning | any `Verify` needs a database, cache, queue, broker, or object store |
| Golden / expected-output file, fixture, snapshot baseline | any `Verify` diffs real output against a stored literal — and every one gets a row in *Byte-exact artifact reconciliation* below |
| Any file a `Verify` command names as an argument | always — including every test file it runs |

That last row is the one that fails silently. Cross-check it mechanically before emitting: **every
path that appears in any `Verify` command must be authored by some §9 step (in bundle mode, present
in some task's `files` array) or emitted here.** A test file the gate runs and no step writes
returns `No test files found` and exits 1 — a green-looking spec that can never pass.

#### A resolution convention is decided once and reconciled against every loader

**Whenever this blueprint states an import, include, or link convention — a module specifier or
extension form, a path alias, a package-export condition, a barrel-file rule, a link mode — that
convention is a contract with *every* resolver in the project, and the resolvers do not agree with
each other by default.** Deciding it once for the application source and repeating that decision in
four places is not deciding it; it is asserting it four times against one of the four consumers.

**The observed failure was a plain script, not a test.** A blueprint mandated one specifier form
throughout the source tree, restated in four separate sections. The application's compiler was
configured for it and accepted it. Then a standalone script imported that source: the bare runtime
strips types but resolves specifiers **literally**, found nothing at the mandated path, and died
with a module-not-found error. Switching the script to the other specifier form made the **compiler**
reject it, because the compiler config this blueprint emitted lacked the one flag that permits that
form. Neither form worked in both contexts. The blueprint never noticed the second context existed,
and the two build steps gated on that script were unbuildable — the same root cause blocking two
separate steps, in a build with no services and no environmental excuses.

**So enumerate the contexts.** There are always at least four, and every one of them is a loader
with its own rules:

| Context | Resolved by | What you must confirm |
|---|---|---|
| **Application source** | the framework's or compiler's resolver | the convention holds, and you can name the config setting that makes it hold |
| **Test files** | the test runner's resolver | the runner config carries the matching alias, condition, or transform |
| **Standalone scripts** | the bare runtime — no framework, no bundler, often no compiler | the specifier form the runtime resolves **literally**, *and* any config flag the compiler needs so the same file still type-checks |
| **Build / bundle** | the bundler or the emitting compiler | the convention survives into the emitted output and the output still resolves |

Add a row for every additional loader this project has: a lint plugin that resolves imports, a
codegen tool that reads the source tree, a container entrypoint, a docs extractor, a REPL.

**If one context needs a different setting, the config emitted for THAT context carries it, and the
matrix below says so — here, next to the convention, not three sections away.** A compiler flag that
permits the script's specifier form belongs in the compiler config this subsection emits, written
into the file, not described in a sentence in §9. The builder who hits the error is reading the
error and the file, not the blueprint's table of contents.

**The self-check, one pass, before §19.6 is done:** for every row of the matrix, name the literal
command that exercises it — the build command, the test command, the script invocation, the bundle
command — and confirm the convention works under that exact command with only the configs this
blueprint emits. **A context you cannot name a command for is a context you have not checked.** This
is the same obligation as *An emitted config must be complete* below, extended from test runners to
every loader, and it is the one that was missed.

#### An emitted config must be complete for the stack this blueprint chose

**Emitting the file is half the obligation. The content has to work with the packages this blueprint
mandates.** A config that exists but cannot load the modules the gates import fails exactly like a
missing one, and worse: the builder sees a config file sitting right there and concludes the error
must be in the code.

The trap is **non-trivial resolution behaviour** in a mandated dependency. Whenever this blueprint
requires a package that is not a plain import — one gated behind an **export condition**, one with a
**bundler-only entry point**, one that ships a **native binary** or a platform-specific artifact, one
that only exists after a **codegen step**, one resolved through a **path alias**, or one that is
**ESM-only in a CommonJS context** (or the reverse) — every config that must load that package has to
be written to handle it, in that config's own syntax.

**The canonical shape, stated generically:** a module that is valid *only* under the framework's
bundler will throw at import inside a plain test runner or a plain script runner, because those
runners resolve modules with different conditions than the bundler does. The fix is a line in the
runner config — the matching resolution condition, an alias to the real entry point, a stub module,
or an exclusion from the transform. Which line depends on the runner. That there must **be** a line
does not.

The failure is total rather than partial. A single mandated import that the runner cannot resolve
kills **every** test in **every** file that transitively reaches it, plus every seed, reset, and
maintenance script that shares the module graph — one missing config line can take out the majority
of the build's steps at once, all reporting an import error that names the package rather than the
config.

**Walk the emitted configs against the mandated dependency list before you consider §19.6 done.**
For each config you emit here, and for each package §11 pins or §19.1/§9 mandates, ask one question:

> *Would this config file successfully load every module the gates import?*

If the answer is "yes, because the package resolves plainly", say nothing. If the answer depends on
a condition, alias, transform, loader, or generated artifact, **write that into the config now** and
record it in the table at the end of this subsection. Pay particular attention to any package this
blueprint makes **mandatory by rule** — a rule of the form "every server module imports X" means X is
in the import graph of every server-side test and script, so X's resolution requirements are the
runner's requirements too.

#### Every tool that reads env vars must be given a way to load them

**A framework loads the env file. A standalone tool does not.** Application frameworks read `.env`
automatically as part of booting, which trains everyone to assume env loading is ambient. It is not.
Migration CLIs, schema-diff tools, seed and reset scripts, one-off maintenance scripts, test runners
invoked outside the framework, container entrypoints and CI steps generally start with whatever is
already exported in the shell — and in an unattended build, that is nothing.

The result is the sharpest possible failure: a §9 step's **literal first command** exits non-zero and
creates nothing, with an error about a missing connection string that the builder can only read as
"the environment is broken", when the environment is fine and the tool was simply never told to look.

**So: if a `Verify` command, a Bootstrap command, or a §9 **Do** command invokes a tool that reads an
env var, this blueprint states the loading mechanism explicitly.** Exactly one of:

| Mechanism | Where it is written | Use when |
|---|---|---|
| An explicit loader import at the top of the tool's own config file | the config emitted in this subsection | the tool has a config file it always evaluates — the most durable option, works from any shell |
| A runner flag that loads the file | the command itself, everywhere it appears | the runtime supports it natively and the flag is available in every place the command is written |
| An export line preceding the command | the same fenced block as the command | one-off shells and CI steps, where no config file is evaluated |

Whichever you pick, it must appear in **every** place the command is written — §10 Bootstrap, the §9
step, §19.1's command table, §20.1's gate — because a mechanism present in one and absent in another
is a gate that passes for the writer and fails for the builder. Prefer the config-file loader for
exactly this reason: it is written once and cannot be forgotten at a call site.

This applies in every ecosystem. The names differ; the failure does not.

#### Services a gate depends on must be provisioned by the blueprint

Listing Docker in §10's prerequisites is not provisioning. If a `Verify` command needs a running
service, emit all four of these or the step is not buildable:

1. **The thing that starts it** — `docker-compose.yml` (or the platform equivalent) under
   `workspace/`, with pinned image tags, a named volume, and a healthcheck, so the up command
   returns only once the service accepts connections rather than racing the first test. An image tag
   is a version pin: give the image a row in §11 with its source and check date, then write that
   literal tag into the emitted file. §11 stays the place a version is *justified*; an emitted file
   is executable and must carry the real value, exactly like a lockfile.
2. **The variable that points at it** — a row in §10's environment table with its literal local
   value (`TEST_DATABASE_URL`, `REDIS_URL`, …), the same key in `.env.example`, and a statement of
   which store tests use. Tests never share a database with dev.
3. **The commands** — up, down, and reset, written into §10's Bootstrap block and §19.1's command
   table, ordered before the first `Verify` that needs them.
4. **The permission** — those same commands in §19.3's `permissions.allow`, so an unattended gate
   does not stall on a prompt. Never pre-approve a command for a file this blueprint does not emit.

If a service genuinely cannot run locally, no `Verify` block may depend on it: move that check to
§20.1's manual gates and say why. An unrunnable gate is worse than a missing one — it reports
failure for a reason that has nothing to do with the code.

#### The bundle sits inside the project, so every emitted config must exclude it

**In bundle mode this blueprint lives at `<project>/blueprints/<slug>/`, and this subsection emits
real config files under its `workspace/`. That puts a second copy of the project's configuration
inside the project's own tree.** A large family of tools discovers configuration by *walking
directories* rather than by being told where to look — formatters, linters, type-checkers, test
runners, editor config, package-manager workspace resolution, in several ecosystems. To those tools
the bundle is not documentation; it is part of the tool surface.

The observed failure, reproduced live: `workspace/` carried a root-level formatter config, the
formatter found **two** root configs in one tree, and it exited 1 **before checking a single file** —
killing the last line of §10's Bootstrap block, which is the very first command the builder runs.
Neither config was wrong. The defect was that both existed in one tree, and nothing said so.

**So every config this blueprint emits — and every config a §10 scaffold generates that a gate
depends on — excludes the bundle path, written in that config's own syntax:**

- the formatter's and linter's ignore/exclude lists
- the type-checker's exclude list
- the test runner's and e2e runner's exclude / ignore patterns
- the package manager's workspace globs, if it has any
- any tool that globs the tree for sources, fixtures, or snapshots

Write the exclusion as a literal line in the emitted file, using the bundle path as it appears from
the project root (`blueprints/`), and record it in this subsection's table. Prose — "the blueprints
directory should be ignored" — excludes nothing. If the bundle is deliberately not committed, the
ignore file from §10's *Files that must be committed* table carries the same path, and that is a
second line, not a substitute for the first.

**A read-only review cannot see this defect.** Both files are individually correct; only running the
gate from the project root, with the bundle present, produces the failure. §10's Bootstrap block is
executed before this blueprint ships (`questions/phase-4-generate.md` Step 6) precisely so this class
surfaces — write the exclusions now rather than collecting them there as findings.

If nothing in this subsection applies, write `NOT APPLICABLE — {reason}` and keep the heading.}

| File | Path in the project | Which `Verify` commands need it | Resolution/env handling it carries | Bundle-path exclusion |
|---|---|---|---|---|
| {file} | {path} | {step numbers} | {the export condition, alias, transform, or env loader written into it — or `none needed: every mandated package resolves plainly and this tool reads no env var`} | {the literal exclude line naming `blueprints/`, or `n/a — this tool never walks the tree`} |

#### Resolution convention matrix

{**Required whenever this blueprint states any import, include, or link convention anywhere** — §3's
boundary rules, §19.1's conventions, a §9 step, a §19.5 rule file. One row per context that loads
project modules, no row omitted because it "obviously works". If this blueprint states no resolution
convention at all, write `NOT APPLICABLE — this blueprint states no import or link convention.` and
keep the heading.}

**The convention, stated once:** {the literal form — the exact specifier shape, alias prefix, or
extension rule — written here and referenced from everywhere else, never restated}

| Context | Command that exercises it | Convention as it appears there | Config + literal setting that makes it work |
|---|---|---|---|
| Application source | `{build or dev command}` | {the specifier form} | {file — the setting} |
| Test files | `{test command}` | {form} | {file — the setting} |
| Standalone scripts | `{the actual script invocation}` | {form the bare runtime resolves} | {file — the setting, including any compiler flag the same file needs to type-check} |
| Build / bundle | `{build command}` | {form} | {file — the setting} |
| {additional loader} | `{command}` | {form} | {file — the setting} |

{Every "Config + literal setting" cell names a file this blueprint emits or a step authors, and the
setting is present in that file's content above. A cell reading "works by default" is only honest
when you can say which resolver's default it is.}

#### Cross-artifact value reconciliation

{**Required whenever this blueprint emits more than one artifact — which is always, once §19.6
exists.** The matrix above reconciles one *convention* across many loaders. This table reconciles one
*value* across many files: same idea, different axis, and it is the axis nothing was checking. Never
write `NOT APPLICABLE` here unless this blueprint emits exactly one file and names no path, port, or
identifier twice.}

**Any value that appears in more than one emitted artifact is the same claim made twice, and nothing
compares the two copies for you.** A build config that says where output lands and a manifest that
says where the entry point is are describing one file from two directions. Each is individually
correct — the completeness rule above passes both — and together they contradict each other. That is
a defect no per-file check can find, because no per-file check ever looks at two files at once.

**The observed failure:** the emitted build config compiled `src/cli/index.ts` to `dist/cli/index.js`
while the emitted manifest declared its binary at `dist/cli.js`. Around thirty `Verify` commands, the
packaging step and the install smoke test all named `dist/cli.js`. Both files were authored by this
blueprint and both were declared off-limits to the builder, so every escape either contradicted an
explicit instruction or invented a mechanism. **Half the build order was unreachable because one path
was written two ways.**

**So enumerate the shared values, name the single source for each, and confirm every other appearance
matches it character for character.** The classes that recur in every ecosystem:

| Value class | Where the same value shows up |
|---|---|
| Output directory / built artifact path | build or compiler config, manifest entry/bin/main/exports, packaging step, ignore file, deploy config, `Verify` commands |
| Entry point / binary / command name | manifest, the build config's input, §9 step commands, §19.1's command table, §19.3's allowlist, the install smoke test |
| Module root / source directory | compiler config, test-runner roots, path aliases, lint and coverage globs, §3's tree |
| Package / project / image name | manifest, container image tag, deploy config, install command, §11 |
| Port | server config, compose file, healthcheck, `Verify` curls, §10's env table |
| Service name, database name, connection URL | compose file, §10's env table, `.env.example`, test setup, migration config |
| The bundle's own path | every exclude list — see *The bundle sits inside the project* above |

| Shared value | Single source — the file that decides it | Literal value | Every other place it appears | Compared |
|---|---|---|---|---|
| {value class} | {file — the field that owns the decision} | {the literal string, not a description} | {file — field · file — field · §9 steps N, M · §20.1} | {yes} |

{One row per value that appears two or more times. **Literal value** is a string: "the dist
directory" is not a value, `dist/cli.js` is. **Compared** is `yes` only when you opened every listed
appearance and matched the strings character for character — not when they merely sound the same.}

**The mechanical pass, before §19.6 is done:** for each artifact you emit, list every path, name,
port, and identifier it contains. Merge the lists. Every value appearing twice or more gets a row
above and a literal comparison. **Values that differ by a prefix, a suffix, a separator, or a
pluralisation are the entire failure mode** — `dist/cli.js` and `dist/cli/index.js` differ by exactly
that much and read as the same thing at a glance. Then check rule 13: the earliest step where both
sides of each contract exist is the step whose `Verify` must exercise it.

#### Byte-exact artifact reconciliation

{**Required whenever this blueprint authors literal bytes that something later compares character
for character** — a golden file, an expected-output fixture, a snapshot baseline, a sample a `Verify`
runs `diff` against, a `jq -e` equality on a literal string. If it authors none, write
`NOT APPLICABLE — this blueprint authors no byte-exact expected output.` and keep the heading.

Authoring those bytes **before** the code that produces them is the right move — it makes the
contract real instead of retrofitted, and it is what rule 4 asks for when the section that would have
held the format does not apply. This table is the obligation that comes with it (§9 rule 15): the
bytes are a **prediction**, and a prediction has to be reconciled twice.

The two matrices above reconcile a *convention* across loaders and a *value* across files. This one
reconciles **authored content against the two things that determine it**: the blueprint's own rules,
and the runtime that will actually emit it. Both were wrong at once in the observed failure — one
artifact carrying two independently wrong facts, each checkable at authoring time.}

| Byte-exact artifact | Authored by | First diffed at | Blueprint rules that constrain it | Runtime call that produces it, on the §11 pin | Both confirmed |
|---|---|---|---|---|---|
| {path} | {§19.6, or step N} | {step M} | {§4 field semantics · §4 path relativity · §5 envelope · key order · number format — name each rule and the literal it dictates} | {the exact call and the literal it returns on the pinned version} | {yes} |

**Column 4 — the blueprint against itself.** Open every section that constrains a byte in this
artifact and read it against the artifact, field by field. §4 saying a path field is *"relative to
the run root"* and the golden file writing a parent-directory prefix into that field is the observed
defect, and the two statements sat **twelve lines apart**. A rule you cannot name is a rule you have
not checked.

**Column 5 — the blueprint against reality.** Every runtime-produced string in the artifact —
exception and parse-error text, stack-trace shape, key order, float formatting, locale and timezone
rendering, sort collation — is version-specific. **Run the producing call on the pinned runtime and
read what it says.** One `JSON.parse` would have caught the observed defect, where the embedded
message was the previous major's format and the pinned engine emits two other families, neither
matching. You have no shell: name the command and the expected literal in your return value as a gap
the main thread executes, or drop the literal.

**A row you cannot complete is a literal you may not write.** Replace the byte-exact comparison with
a property the step can assert — a schema validation, field-by-field assertions, or a diff normalised
to strip the runtime-specific part — and state in the criterion that it *is* the property (§9 rule
12). Never leave the mismatch and describe the repair in §20.2: a builder who must decide that the
blueprint's own format was wrong is making the clarifying decision this blueprint promised it would
never require.

---

## 20. Acceptance Gate, Risks & Decision Log

### 20.1 Global acceptance gate

The project is **done** when every command below exits 0 on a clean checkout, and not before. This
is the same set CI runs and the same set every Section 9 step is measured against.

```bash
{pm} install --frozen-lockfile
{pm} typecheck        # expect: exit 0, zero errors
{pm} lint             # expect: exit 0, zero errors and zero warnings
{pm} test             # expect: exit 0, 0 failed, 0 skipped
{pm} test:e2e         # expect: exit 0, 0 failed
{pm} build            # expect: exit 0
{run the built entry point — the binary, the published entry, or a request to the served endpoint}
                      # expect: exit 0 / the documented status — proves the artifact the manifest
                      #         declares is the one the build actually produced (§9 rule 13)
{a11y command}        # expect: 0 violations
```

**Every expectation above is a property, not a tally** — §9 rule 10. Where a count genuinely belongs
in this gate, it is counted from this blueprint's own content and matches the number every step and
every epic states for the same fact. A gate that greps for a number the blueprint never computed
fails on every machine.

**Every line above exits 0 on a correct build** — §9 rule 11. This gate is read by a runner, so a
check whose correct outcome is a non-zero exit is wrapped in an assertion (`{cmd}; test $? -eq 2`)
rather than written bare; the expected code stays in the comment. And every line is decidable in the
medium it runs in — §9 rule 12. A gate command that cannot pass, or cannot fail, is not a gate.

**And every line that fails, fails for the reason it claims** — §9 rule 16. Nothing here, and nothing
in the manual list below, may treat *"exits non-zero"* as a pass condition: a usage error — wrong
arity, an unknown flag, an illegal flag combination — exits non-zero too, and a gate written that way
passes vacuously and would keep passing after the property it checks broke. Assert the **specific**
code (`{cmd}; test $? -eq 1`), or restructure the check so success is exit 0. `!` and `test $? -ne 0`
accept every failure equally, including the ones that mean the command was malformed.

Plus these manual gates, each checked once before launch:

- [ ] Every step in §9 has its checkpoint tag in git (`git tag -l 'step-*'` lists one per step).
      The repository these tags live in is created by §10's Bootstrap block, not by a scaffolder.
- [ ] Every file §10's *Files that must be committed* table names is present in a clean checkout
      (`git ls-files --error-unmatch <path>` exits 0 for each — **one path per invocation**, so a
      failure is the file's, not the command's) — no ignore pattern swallowed it.
      **This is the home for whole-repository tracked-file assertions** (§9 rule 14): this gate runs
      after every step has committed, so it can decide what no step's own `Verify` can. A per-step
      version of the same check belongs in that step's `Checkpoint` block, after the commit — never
      in its `Verify`. **Write the not-ignored companion check per path and assert the code** —
      `git check-ignore -q <path>; test $? -eq 1` for each, never `! git check-ignore -q <a> <b>`,
      which exits 128 for usage and passes whatever the ignore file says (§9 rule 16).
- [ ] The ignore file was in place before the first commit: `git log --diff-filter=A --format=%H`
      shows it added in §10's Bootstrap commit, not in a §9 step's commit. Once a path is tracked, no
      ignore rule ever excludes it — see §10, *The ignore file precedes the first commit*.
- [ ] Every row of §19.6's *Byte-exact artifact reconciliation* table reads `Both confirmed: yes` —
      each golden file, fixture and snapshot baseline was read against the blueprint rules that
      constrain it **and** against the output of the producing call on the §11-pinned runtime
      (§9 rule 15).
- [ ] §10's Bootstrap block has been re-run once on an already-bootstrapped tree, **exited 0**, and
      changed nothing that mattered: the package manifest still lists every installed dependency and
      the next command still finds its binaries. This proves the guarded `workspace/` copy (§19)
      holds — and that the guard itself does not fail on the path it guards against.
- [ ] Every row of §19.6's *Cross-artifact value reconciliation* table reads `Compared: yes`, and the
      lint/format/typecheck gates above were run from the project root **with the bundle present** —
      the exclusions in §19.6 are what keep the bundle's own configs from breaking them.
- [ ] If §9.1 applies: every parity row proved, the kill switch exercised once on purpose, and the
      old path still deployed and revertible.
- [ ] Every non-goal in §1 is still un-built.
- [ ] Every environment variable in §10 is set in production and absent from the repo.
- [ ] The critical E2E flows in §13 pass against the production URL.
- [ ] Keyboard-only pass and one screen-reader pass over the primary flow (§15).
- [ ] Error tracking receives a deliberately triggered test error (§16).
- [ ] A rollback has been performed once, on purpose, in a preview environment (§12).

**No warnings are ignored.** A tolerated warning becomes a permanent warning, and the next real one
hides inside it.

### 20.2 Risk register

| Risk | Likelihood | Impact | Early signal | Mitigation |
|---|---|---|---|---|
| {risk} | {H/M/L} | {H/M/L} | {what you'd observe first} | {the concrete action, with an owner} |

{5-8 rows. Include at least one technical risk, one dependency/vendor risk, and one scope risk.
"The build takes longer than expected" is not a risk — it is an outcome. Name the cause.}

### 20.3 Decision log

{The most valuable table in the blueprint six months from now. Every non-obvious choice above gets a
row: what was decided, what was rejected, and — critically — **what new information would reverse
it**. Without the reversal trigger, future maintainers cannot tell a considered decision from an
accident, so they either cargo-cult it forever or rip it out blind.}

| # | Decision | Rejected alternative | Why | Would reverse if |
|---|---|---|---|---|
| 1 | {what was chosen} | {the real runner-up, not a strawman} | {reason} | {the observable condition that flips it} |
| 2 | {…} | {…} | {…} | {…} |

{Cover at minimum: runtime track, database, auth provider, hosting, styling approach, state
management, and any capability where `knowledge/capabilities/*.md` offered a close call.}

### 20.4 What to build next

{3-5 items from §1's non-goals, ordered, each with the trigger from that table. This is the v2
backlog, and it is the last thing in the blueprint so nobody mistakes it for scope.}

---

*End of blueprint. Build order is §9. Stop when §20.1 is green.*

---
---

## See also

- `templates/claude-md-template.md` — the CLAUDE.md emitted in §19.1
- `templates/tasks-schema.md` — machine-readable step manifest for bundle mode (§9)
- `templates/epic-template.md` — grouping build steps into epics for bundle mode
- `knowledge/skills-registry.md` — authoritative skill names and install commands for §18
- `knowledge/stack-compatibility.md` — known-bad combinations, checked in §2
- `questions/phase-4-generate.md` — the procedure that fills this template
