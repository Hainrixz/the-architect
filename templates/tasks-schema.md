# Template: tasks.json

> The machine-readable build order. A dependency graph, not a checklist — which is what lets
> independent branches run in parallel and lets an interrupted build resume without a human.

Last verified: 2026-07-27

Emitted in **bundle mode** only (see `questions/phase-4-generate.md`):

```
./blueprints/<project-slug>/
├── blueprint.md          # the 20-section narrative artifact
├── tasks.json            # ← this file, at the bundle root next to blueprint.md
├── epics/
│   ├── 01-<name>.md
│   └── 02-<name>.md
└── workspace/            # copied INTO the target project root by the builder
    ├── CLAUDE.md
    ├── AGENTS.md
    ├── <verify-critical config>   # test/e2e runner config, compose file — blueprint §19.6
    └── .claude/
        ├── settings.json
        ├── skills/<name>/SKILL.md
        └── rules/<name>.md
```

`./blueprints/` is relative to the **user's current working directory**, never the plugin.
`tasks.json` sits at the bundle root, beside `blueprint.md` — not inside `epics/` and not inside
`workspace/`. `workspace/` is the only directory the builder copies anywhere: its whole contents go
to the target project root. The bundle itself stays where it was generated.

`workspace/` carries the config every `verify` command needs in order to run — runner configs, test
setup files, the compose file for any service the tests hit — as real files, emitted per
`blueprint.md` §19.6. That is why a task's `files` array does not list them: they are already at the
project root before task one. Everything else a `verify` command names is a task's responsibility.

`blueprint.md` explains *why*. `epics/*.md` explain *how*. `tasks.json` decides *what next* — it is
the only file the builder needs to answer "where am I?" after a context reset.

---

## Shape

A JSON array of task objects. No wrapper, no metadata envelope — the array *is* the file.

**This file is machine-read, so it is ASCII English regardless of the user's language.** Keys, `id`,
`epic`, `files[]`, `verify[]` and `status` never localize. Only `title` and `acceptance[]` prose
follow the user — and inside `acceptance[]`, the EARS keywords stay uppercase English:
`"WHEN \`pnpm test\` 运行，THE SYSTEM SHALL 退出码为 0"`. See `knowledge/localization.md`.

```json
[ { "id": "E1-T1", "...": "..." }, { "id": "E1-T2", "...": "..." } ]
```

## Fields

| Field | Type | Required | Rules |
|---|---|---|---|
| `id` | string | yes | `E{epic}-T{n}`, e.g. `E2-T3`. Unique across the file. Never renumber after emission — dependencies point at these. |
| `title` | string | yes | Imperative, ≤ 60 chars. "Add Stripe checkout session route", not "Stripe stuff". |
| `epic` | string | yes | Slug of the epic file, without extension: `02-auth` → `epics/02-auth.md`. |
| `dependencies` | string[] | yes | Task ids that must be `done` first. `[]` for roots. **This is what makes it a DAG.** May cross epics. |
| `priority` | string | yes | `p0` blocks a usable build · `p1` needed for launch · `p2` deferrable. **Metadata only — it does not select the next task.** See "Selection". |
| `acceptance` | string[] | yes | 1–6 EARS criteria. **WHEN** `<trigger>` **THE SYSTEM SHALL** `<observable response>`. Each one must be decidable by a script, on this machine, during the build. |
| `verify` | string[] | yes | **An ARRAY of shell commands — always, even when there is one.** Each is run from the target project root, **exits 0 when the task is correct**, and exits non-zero on failure. |
| `checkpoint` | string | yes | The version-control tag this task's commit carries: `step-{NN}-{slug}`, the **same tag** the corresponding `blueprint.md` §9 step carries. `NN` is the global two-digit step number, not the per-epic task number. This is the rollback target. |
| `files` | string[] | yes | Paths this task creates or edits. Globs allowed. Doubles as the parallel-safety check. |
| `status` | string | yes | `pending` · `in_progress` · `done`. Every task starts `pending`. **These three literals are never translated** — `/architect-next` compares them exactly, so a localized value makes every task invisible and the build reads as finished. |

### Field rules that matter

**`dependencies` are real ordering constraints, not narrative order.** If E3-T1 does not actually
need E2-T4's output, do not list it. Every false dependency serializes work that could have run in
parallel. Ask of each edge: *would this task fail if the other had not run?* If no, drop it.

**`acceptance` must be checkable BY A MACHINE, INSIDE the build.** The test: *could a script decide
this, today, without leaving the machine?* If no, it is not an acceptance criterion. A criterion
that waits on a human reviewer, an app-store review queue, a certificate authority, or a real
physical device cannot terminate inside an autonomous build — the builder either stalls forever or
silently self-certifies, and both are worse than having no criterion at all.

| Reject | Write instead |
|---|---|
| "WHEN the build is submitted THE SYSTEM SHALL be accepted into store review." | "WHEN `pnpm package` runs THE SYSTEM SHALL emit a store-ready artifact with every required manifest field non-empty." |
| "WHEN a clean machine opens the installer THE SYSTEM SHALL show no security warning." | "WHEN `codesign --verify --deep --strict` runs THE SYSTEM SHALL exit 0." |
| "WHEN a reviewer reads the README THE SYSTEM SHALL be understandable." | "WHEN `pnpm test tests/docs/examples.test.ts` runs THE SYSTEM SHALL assert command output byte-matches the documented example." |
| "WHEN the page loads THE SYSTEM SHALL have no clipped or overlapping text." | "WHEN the snapshot suite runs at min and max type scale THE SYSTEM SHALL report zero truncated text nodes." |
| "WHEN design approves the palette THE SYSTEM SHALL use it." | "WHEN `pnpm test tests/tokens.test.ts` runs THE SYSTEM SHALL assert every color literal resolves to a token in `tokens.ts`." |

Anything that genuinely requires an external party — store approval, a real device, a human
sign-off, a DNS propagation window — is **not a task**. It goes in the post-build launch checklist
in `blueprint.md`, clearly separated from the build order. It is still written down; it just is not
a build gate.

Also banned: a criterion the blueprint already satisfies before any code is written ("the stack is
documented"). That gates nothing.

**Write every `acceptance` string so it stands alone — this is the only place it is authored.** The
string is copied *verbatim* into the epic's task block, and the epic must be completable without
opening `blueprint.md`. A criterion whose meaning depends on a section the epic reader will not open
("…SHALL match the envelope in §5") is therefore unusable there, and because the two copies must be
byte-identical, it cannot be fixed on the epic side either. Fix it here: state the contract in the
criterion, and keep the section pointer as a parenthetical if it is useful — "WHEN a handler throws
THE SYSTEM SHALL return `{ error: { code, message } }` with `code` from the enumerated set (§5)."
`templates/epic-template.md`, *Self-contained vs. verbatim*, is the full rule.

**`verify` is an ARRAY of commands — every one of them runs, and every one must pass.** A consumer
that treats it as a single string silently runs only the first check and marks the task done on a
partial pass. Write `"verify": ["pnpm typecheck"]` for one command, never `"verify": "pnpm
typecheck"`. A task is `done` only when the **last** command in the array has exited 0.

**`verify` commands run from the TARGET PROJECT ROOT — not from the bundle directory.** The bundle
is a design artifact; `pnpm test` inside it fails for reasons that have nothing to do with the code.
Write every command relative to the project root, using only commands defined in the emitted
`workspace/CLAUDE.md`. If the bundle does not sit inside the project it builds, the runner must say
so and stop rather than executing commands in the wrong directory and reporting phantom failures.

**`verify` must be executable, not aspirational.** `pnpm test tests/auth.spec.ts` is a verify
command. `check that login works` is not. If a criterion genuinely cannot be checked by a command
(visual layout, copy tone), say so explicitly: `# manual: compare against epics/04-ui.md §Header`.
One escape hatch per epic, maximum — and it never gates a task, it only annotates one.

**Every `verify` command EXITS 0 when the task is correct — the converse of "non-zero on failure",
and the half that gets dropped.** The resume protocol reads exit status and nothing else: step 2 and
step 8 both decide `done` vs. not-done purely from it. No consumer reads your comment, and none can
tell "the tool errored" from "the tool correctly errored". So a command that gates a **documented
error path** — a non-zero exit code that is the *right* answer — must be wrapped so that the line
itself exits 0, with the expected code still visible:

| Silently fails the gate forever | Correct — the line itself exits 0 |
|---|---|
| `"mytool --bad-flag"` | `"mytool --bad-flag; test $? -eq 2"` |
| `"mytool query no-such-tag"` | `"mytool query no-such-tag; test $? -eq 1"` |
| `"grep -q FIXME out.txt"` (expecting no match) | `"! grep -q FIXME out.txt"` |
| `"curl -f localhost:3000/missing"` (expecting 404) | `"test \"$(curl -s -o /dev/null -w '%{http_code}' localhost:3000/missing)\" = 404"` |

Read every `verify` array top to bottom and ask, for a *correct* task, what status each line exits
with. Any line whose success case is a non-zero exit is a permanently red gate, and the builder's
only escape is to edit the command — which the resume protocol forbids. **The same wrapped string is
what gets copied into the epic's `Verify` block**, so the wrapping survives rendering rather than
being re-derived there.

**`checkpoint` is the rollback target, and it is required on every task.** One commit per task, one
tag per task. The tag is what `blueprint.md` §20.1's final gate counts (`git tag -l 'step-*'` lists
one per step), and it is what a bad task is undone with — `git reset --hard` to the *previous* task's
tag, never a debug-forward through broken work.

It is a field rather than a line of prose in the epic preamble for one reason: **a bundle-mode
builder resumes from `tasks.json` and is told the epic file is self-contained.** A requirement that
lives only in `blueprint.md` is therefore a requirement that builder never reads. It commits without
tagging, nobody notices, and §20.1 fails at the very end of the build with every rollback target
already gone. Put it where the builder actually looks: here, and in the epic's task block.

Write the tag `blueprint.md` §9's corresponding step already carries — do not invent a parallel
scheme, or §20.1's count and §9's rollback instructions stop agreeing. Adapt the two commands to
whatever VCS §9 uses; the obligation is the same.

**`files` is how two agents avoid colliding.** Two ready tasks may run concurrently only if their
`files` arrays do not intersect. Be honest about shared files — listing `src/db/schema.ts` on four
tasks correctly forces them to serialize.

**`files` and `verify` must close over each other. This is the check that decides whether a bundle
can run at all.** Every path named in a `verify` command — the test file it executes, the config it
loads, the fixture it reads — must be authored in exactly one of three places:

1. this task's `files` array,
2. an earlier task's `files` array, or
3. the bundle's `workspace/`, emitted per `blueprint.md` §19.6.

A path in none of the three is a gate that runs a file nothing creates. It does not fail like a bug;
it fails like `No test files found, exiting with code 1` or `Cannot find module`, which reads as
broken tooling, and the builder's only recovery is to invent the file and guess what it asserts —
at which point the criterion is being graded by the code it was supposed to grade. **Test files are
the common miss:** a task that adds a `verify` command running `tests/x.test.ts` must list
`tests/x.test.ts` in its `files`, because writing that test is part of the task. Extract every path
from every `verify` string and match it before emitting — it takes seconds and it is the difference
between a bundle that builds and one that stalls on its first gate.

**A `verify` command may not need a service the blueprint does not provision.** If it hits a
database, cache, queue, or object store, `blueprint.md` §19.6 must emit the file that starts that
service locally and §10 must define the variable pointing at it. Otherwise the command is
unrunnable on the builder's machine — which is not a failing gate, it is an absent one.

**`status` is written back immediately**, before and after the work — not batched at the end. A
crash mid-task must leave evidence. That evidence is the `in_progress` marker, and the resume
protocol below depends on it existing.

---

## Selection — array order IS the build order

**Emit the array in build order, then never re-rank it.**

The writer sorts tasks once, at emission time, into the order they should be executed. That order
already encodes priority, dependency depth, and the writer's judgement about what to build first.
By the time a builder reads the file, ranking is a solved problem.

So the rule is one sentence with no tiebreaker clause, because there is nothing left to tie:

> **Pick the first task in the ready set, in array order.**

`priority` is **metadata, not a selector**. It exists for two human-facing purposes:

- reading the file — knowing at a glance whether a task is load-bearing;
- the scope-cut conversation — "we ship in a week, what falls out?" is answered by dropping `p2`,
  then `p1`, and re-emitting the bundle.

A builder that re-sorts by `priority`, by its own sense of urgency, or by what looks quick will
diverge from the epic files, from `blueprint.md`'s build order, and from any parallel agent reading
the same file. If the order is wrong, fix the array — do not out-think it at runtime.

---

## The splitting rule

Split a task if **either** is true:

- more than **6** acceptance criteria, or
- it touches more than **5** files.

Agent success rate falls sharply and non-linearly with task length. A task that fails at 80% done
loses all of it; two tasks that fail at 80% lose half. Oversized tasks are the single most common
cause of an autonomous build stalling.

The split is almost always the same three ways:

| Symptom | Split into |
|---|---|
| Schema + API + UI in one task | data layer → server layer → interface |
| "Implement CRUD" | read path → write path → delete/edge cases |
| One task per external service | client + config → happy path → webhook/error handling |

If a task cannot be split without a circular dependency, the design is wrong — fix the boundary in
`blueprint.md`, not the task list.

---

## Worked example

Four epics of a subscription SaaS, shown **mid-build** so the statuses are visible — at emission
time every `status` is `pending`. Note that E2 (auth) and E3 (billing groundwork) both depend only
on E1-T2, so they are two independent branches that can run at the same time.

**This is an excerpt, not a bundle.** Two tasks per epic, trimmed so the graph fits on a page; a
real epic carries 5–9. Read the *shape* off it — field types, array-not-string, path closure,
branch structure — and never a count. `blueprint.md` §9's counting rule owns every number.

Five things to read off it: the array is already in build order, so no consumer sorts it; every
`verify` is an array even where it holds one command; every `acceptance` string names something a
command in that task's own `verify` array actually decides; **every test file any `verify`
command runs appears in that task's `files`** — writing the test is part of the task, so the paths
close over each other with nothing left to invent; and **every task carries a `checkpoint` tag**,
numbered by global step rather than by task id, which is the rollback target and the thing §20.1
counts at the end of the build.

```json
[
  {
    "id": "E1-T1",
    "title": "Scaffold project and pin toolchain",
    "epic": "01-foundation",
    "dependencies": [],
    "priority": "p0",
    "acceptance": [
      "WHEN `pnpm install --frozen-lockfile` runs THE SYSTEM SHALL exit 0 without modifying the lockfile.",
      "WHEN `pnpm typecheck` runs THE SYSTEM SHALL exit 0 with `strict` true in `tsconfig.json`.",
      "WHEN the boot smoke test requests `/` against the built server THE SYSTEM SHALL return 200 and log no console errors."
    ],
    "verify": [
      "pnpm install --frozen-lockfile",
      "pnpm typecheck",
      "pnpm build",
      "pnpm test tests/smoke/boot.test.ts"
    ],
    "checkpoint": "step-01-scaffold",
    "files": ["package.json", "tsconfig.json", ".nvmrc", "src/app/page.tsx", "tests/smoke/boot.test.ts"],
    "status": "done"
  },
  {
    "id": "E1-T2",
    "title": "Add database client and base schema",
    "epic": "01-foundation",
    "dependencies": ["E1-T1"],
    "priority": "p0",
    "acceptance": [
      "WHEN `pnpm db:migrate` runs against an empty database THE SYSTEM SHALL create the `users` and `organizations` tables.",
      "WHEN a query is issued through the exported client THE SYSTEM SHALL return typed rows with no `any`.",
      "WHEN `DATABASE_URL` is absent THE SYSTEM SHALL fail at boot with a named error, not at first query."
    ],
    "verify": [
      "pnpm db:migrate",
      "pnpm test tests/db/client.test.ts"
    ],
    "checkpoint": "step-02-db-client",
    "files": ["src/db/schema.ts", "src/db/client.ts", "src/lib/env.ts", "migrations/**", "tests/db/client.test.ts"],
    "status": "in_progress"
  },
  {
    "id": "E2-T1",
    "title": "Add email + OAuth sign-in",
    "epic": "02-auth",
    "dependencies": ["E1-T2"],
    "priority": "p0",
    "acceptance": [
      "WHEN a visitor submits a valid email THE SYSTEM SHALL send a magic link and create a pending session.",
      "WHEN a visitor completes the OAuth flow THE SYSTEM SHALL upsert a `users` row keyed on provider id.",
      "WHEN an unauthenticated request hits a protected route THE SYSTEM SHALL redirect to `/sign-in` with a `next` param."
    ],
    "verify": [
      "pnpm test tests/auth/session.test.ts",
      "pnpm test:e2e tests/e2e/sign-in.spec.ts"
    ],
    "checkpoint": "step-03-sign-in",
    "files": ["src/lib/auth.ts", "src/app/(auth)/sign-in/page.tsx", "src/app/api/auth/[...all]/route.ts", "tests/auth/session.test.ts", "tests/e2e/sign-in.spec.ts"],
    "status": "pending"
  },
  {
    "id": "E2-T2",
    "title": "Add organization membership and role checks",
    "epic": "02-auth",
    "dependencies": ["E2-T1"],
    "priority": "p1",
    "acceptance": [
      "WHEN a user signs up without an invite THE SYSTEM SHALL create an organization and assign them `owner`.",
      "WHEN a `member` calls an owner-only action THE SYSTEM SHALL return a 403 typed error and write no rows."
    ],
    "verify": ["pnpm test tests/auth/rbac.test.ts"],
    "checkpoint": "step-04-org-roles",
    "files": ["src/server/organizations.ts", "src/lib/permissions.ts", "src/db/schema.ts", "tests/auth/rbac.test.ts"],
    "status": "pending"
  },
  {
    "id": "E3-T1",
    "title": "Add billing schema and plan catalog",
    "epic": "03-billing",
    "dependencies": ["E1-T2"],
    "priority": "p1",
    "acceptance": [
      "WHEN migrations run THE SYSTEM SHALL create `subscriptions` with a unique index on `organization_id`.",
      "WHEN the plan catalog is imported THE SYSTEM SHALL expose price ids from env, never hardcoded."
    ],
    "verify": ["pnpm db:migrate", "pnpm test tests/billing/plans.test.ts"],
    "checkpoint": "step-05-billing-schema",
    "files": ["src/db/schema.ts", "src/server/billing/plans.ts", "tests/billing/plans.test.ts"],
    "status": "pending"
  },
  {
    "id": "E3-T2",
    "title": "Handle checkout webhook",
    "epic": "03-billing",
    "dependencies": ["E2-T2", "E3-T1"],
    "priority": "p0",
    "acceptance": [
      "WHEN a `checkout.session.completed` event arrives with a valid signature THE SYSTEM SHALL upsert one `subscriptions` row and return 200.",
      "WHEN the same event id arrives twice THE SYSTEM SHALL return 200 and leave the row count unchanged.",
      "WHEN the signature is invalid THE SYSTEM SHALL return 400 and persist nothing."
    ],
    "verify": [
      "pnpm test tests/billing/webhook.test.ts",
      "pnpm test tests/billing/webhook-replay.test.ts"
    ],
    "checkpoint": "step-06-checkout-webhook",
    "files": ["src/app/api/webhooks/stripe/route.ts", "src/server/billing/sync.ts", "tests/billing/webhook.test.ts", "tests/billing/webhook-replay.test.ts"],
    "status": "pending"
  },
  {
    "id": "E4-T1",
    "title": "Build dashboard shell and navigation",
    "epic": "04-dashboard",
    "dependencies": ["E2-T1"],
    "priority": "p1",
    "acceptance": [
      "WHEN a signed-in user visits `/app` THE SYSTEM SHALL render the shell with their organization name.",
      "WHEN the viewport is under 768px THE SYSTEM SHALL collapse navigation into a drawer with no horizontal scroll."
    ],
    "verify": ["pnpm test:e2e tests/e2e/dashboard-shell.spec.ts"],
    "checkpoint": "step-07-dashboard-shell",
    "files": ["src/app/(app)/layout.tsx", "src/components/nav/sidebar.tsx", "tests/e2e/dashboard-shell.spec.ts"],
    "status": "pending"
  }
]
```

The graph this describes:

```
E1-T1 → E1-T2 ─┬→ E2-T1 ─┬→ E2-T2 ──┐
               │         └→ E4-T1   ├→ E3-T2
               └→ E3-T1 ────────────┘
```

After `E1-T2`, three branches open. `E3-T1` and `E4-T1` share no files with anything ready at the
same time, so they can run concurrently.

Two deliberate choices in E3-T2 worth copying:

- Its `verify` replays a **recorded** `checkout.session.completed` fixture instead of running
  `stripe trigger`. A live trigger needs network and a logged-in CLI — it leaves the machine, so it
  fails D4 and it makes the build hang on someone else's infrastructure. The one-time live smoke
  test against Stripe's dashboard goes in the post-build launch checklist, not in `tasks.json`.
- Its acceptance names idempotency and the invalid-signature path, not "billing works". Each of the
  three strings maps to an assertion a listed test file makes.

One thing that is deliberately *absent*: no task lists the test-runner config, the e2e-runner
config, or the compose file for the Postgres that `pnpm db:migrate` and every integration test in
this example need. They are not missing — they were emitted under `workspace/` (`blueprint.md`
§19.6) and copied to the project root before `E1-T1` started, along with the `TEST_DATABASE_URL`
that points at them. If those files had *not* been emitted, `E1-T2`'s first `verify` command would
be unrunnable and every task downstream of it blocked, which is what "provision what the gates
assume" means in practice.

---

## Resume protocol

The builder runs this at the start of every session. It is the whole reason the file exists.

**Two definitions, used below and nowhere ambiguous:**

> **Ready set** — every task where `status == "pending"` **AND** every id in its `dependencies`
> array has `status == "done"`. Nothing else is ready. A task with `status: "in_progress"` is
> **never** in the ready set, and neither is a task whose dependency is itself `in_progress`.
>
> **Interrupted task** — a task left at `status: "in_progress"` by a session that ended before
> writing `done`: a crash, a killed process, a closed terminal, an exhausted context window. It is
> not ready work and it is not finished work. It is *unknown-state* work, and it is handled first,
> by re-running its `verify` array to find out which.

### Steps

1. **Read `tasks.json`.** Fresh from disk, every session. Never work from a remembered copy.

2. **Crash recovery — resolve every `in_progress` task before anything else.** For each one, run its
   full `verify` array from the target project root:
   - **Every command exits 0** → the work finished but the write-back did not. Set `done`, write the
     file, and go to step 3.
   - **Any command fails** → the work is partial. Re-open `epics/{epic}.md`, finish *that* task,
     re-run the whole `verify` array, then set `done`. **Start nothing else until it resolves.**
     Do not reset it to `pending` and do not delete the partial code — verify tells you what is
     missing far faster than a rebuild does.
   - **Verify cannot run at all** (missing deps, no lockfile install, wrong directory) → stop and
     report. A verify that errors is not a verify that failed; treat it as unknown, never as done.
   - **More than one `in_progress`** → legal only if parallel agents are genuinely running right
     now. Otherwise it is corruption from a multi-task crash: resolve every one of them by the rules
     above, in array order, before selecting new work.

3. **Compute the ready set** using the definition above.

4. **Ready set empty?** With nothing `in_progress` left after step 2, there are exactly two cases —
   distinguish them and stop either way:
   - every task is `done` → the build is complete. Report and stop.
   - some tasks are still `pending` → the graph is blocked behind a dependency that never completed.
     Report which task ids are blocked and on what. **Do not improvise a path around it**, do not
     edit `dependencies` to unblock yourself, do not start a `p2` "while we're stuck".

5. **Pick the first task in the ready set, in array order.** No tiebreaker: array order is the build
   order and already encodes priority — do not re-rank (see "Selection" above). Running agents in
   parallel: walk the ready set in array order and hand out tasks whose `files` arrays do not
   intersect, one task per agent.

6. **Set `in_progress` and write the file** *before* touching any code. This is what makes step 2
   possible next session — skip it and a crash leaves no evidence at all.

7. **Open `epics/{epic}.md`** and execute only that task. The epic file is self-contained — do not
   re-read `blueprint.md`.

8. **Run every command in the `verify` array.** All of them, in order, from the target project root.
   The task is done only when the last one exits 0. A failing verify is never a reason to edit the
   verify command.

9. **Set `done`, write the file, commit, and tag the checkpoint.** One commit per task, message
   prefixed with the id, then the task's `checkpoint` tag — both, every time:

   ```bash
   git add -A && git commit -m "E2-T1: add email + OAuth sign-in"
   git tag step-03-sign-in          # the task's `checkpoint` value, verbatim
   ```

   **The tag is not optional and it is not deferrable.** It is this task's rollback target, and
   `blueprint.md` §20.1's final gate counts one tag per step at the end of the build. Skipping it
   costs nothing now and fails the build's last gate with every rollback target already gone —
   by then the only fix is to re-tag from memory, which is not a rollback target, it is a guess.
   If the task has no `checkpoint` field, stop and report: the bundle is malformed, and inventing a
   tag scheme here guarantees it disagrees with §20.1's.

10. **Return to step 3.** Do not compact or summarize between tasks — re-read `tasks.json`, which is
    cheap and always current.

### Where this runs

Steps 2 and 8 execute shell commands. They run **from the target project root**, not from
`./blueprints/{project-slug}/`. The bundle is a design artifact with no `package.json` and no
dependencies; running `pnpm test` inside it fails for reasons unrelated to the code, and those
failures look exactly like real ones.

Before step 2, confirm the target project root — the directory holding the emitted
`workspace/CLAUDE.md` contents. **If the bundle is not inside the project it builds, say so and
stop.** Ask which directory is the project root rather than guessing; a resume that runs verify in
the wrong tree will mark good work as broken, or worse, mark broken work as fine.

---

## Validation checklist

Before emitting `tasks.json`:

- [ ] Valid JSON. Parses with `jq . tasks.json`.
- [ ] Every `id` unique and matching `E{n}-T{n}`.
- [ ] Every entry in every `dependencies` array exists as an `id`.
- [ ] No cycles. Walk it once; a task must never reach itself.
- [ ] At least one task with `dependencies: []`.
- [ ] Every task reachable from a root — an unreachable task will never run.
- [ ] Every `epic` value matches an emitted file in `epics/` — a sibling directory of `tasks.json`.
- [ ] The number of distinct `epic` values is legal for the task count, per `blueprint.md` §9's
      counting rule — at least `ceil(tasks ÷ 9)`, at most `floor(tasks ÷ 5)`. That subsection is the
      only place these numbers live; do not re-derive them here.
- [ ] No task exceeds 6 acceptance criteria or 5 files.
- [ ] Every `acceptance` string is in EARS form, observable, and **decidable by a script on this
      machine during the build** — nothing waiting on a human, a store queue, a certificate
      authority, or a physical device.
- [ ] **Every `acceptance` string is self-contained.** It carries the contract it asserts rather than
      pointing at a `blueprint.md` section for it, because the epic copies it verbatim and the epic
      reader never opens `blueprint.md`. A bare `§N` reference with no contract beside it is a defect
      here, not in the epic.
- [ ] Nothing that needs an external party is a task. It is in `blueprint.md`'s post-build launch
      checklist instead.
- [ ] `verify` is a JSON **array** on every task, including single-command ones. Zero string values.
- [ ] **Every `verify` command exits 0 when the task is correct.** Read each line and ask what status
      a *correct* task produces. Any command whose success case is a non-zero exit — a documented
      error code, an expected `grep` miss, an expected 404 — is wrapped so the line itself exits 0.
- [ ] **Every task has a `checkpoint`**, its value matches the tag on the corresponding `blueprint.md`
      §9 step, and the tags are unique across the file. Zero tasks without one — the epic task block
      renders this field, and a task missing it ships a step with no rollback target.
- [ ] Every `verify` command is runnable **from the target project root** using only commands
      defined in the emitted `workspace/CLAUDE.md`.
- [ ] **Every path named in every `verify` command is authored somewhere** — this task's `files`, an
      earlier task's `files`, or an emitted file under `workspace/`. Extract the paths and match
      them; a path in none of the three is a gate that runs a file nothing creates.
- [ ] **Every service any `verify` command needs is provisioned by the bundle** — the file that
      starts it locally is under `workspace/`, and the variable pointing at it is in `blueprint.md`
      §10 and in `.env.example`. No `verify` depends on infrastructure the blueprint only mentions.
- [ ] **Every package pinned in `blueprint.md` §11 is installed by something** — §10's Bootstrap
      block or a named step's install command. Grep each package name across both; zero hits is a
      defect (§11's `Installed by` column is where you check this).
- [ ] **No task's work breaks an earlier task's `verify`.** Walk the array in order and ask of each
      task: does it add a constraint — env validation, a lint rule, a schema check — that an earlier
      task's gate would now fail? If so, that constraint ships too early (`blueprint.md` §9 rule 9).
- [ ] The array is already in build order — the first task with `dependencies: []` is the one to
      build first. No consumer needs to sort it.
- [ ] Every `status` is `pending`. No `in_progress` at emission time.
- [ ] Total task count and the union of `epic` values match `blueprint.md`'s build order.

---

## See also

- `templates/epic-template.md` — the epic files these tasks execute against
- `templates/blueprint-template.md` — single-file mode, where the build order stays prose
- `templates/claude-md-template.md` — `workspace/CLAUDE.md`, whose commands the `verify` arrays call
- `questions/phase-4-generate.md` — when bundle mode is chosen over single-file mode
