# Phase 4: Generate

> Say how long it will take, verify versions, decide the output format, compose, validate, write to
> the user's directory, hand off.

Last verified: 2026-07-28

**Paths in this file:** every bare path (`templates/…`, `knowledge/…`, `questions/…`) is relative to
the plugin root — open it as `${CLAUDE_PLUGIN_ROOT}/<path>`. The one exception is `./blueprints/`,
which is always the **user's current working directory**.

Phase 4 is a procedure, not a conversation. Follow the eight steps in order. **You do not interview
the user here.** The interview is over; every decision that remains is yours to make and *announce*.
The one thing you may stop to ask is destructive — overwriting an existing blueprint, in Step 7.
Every step below carries a *Done when* — you do not advance past a step whose condition you cannot
observe.

**You are the only participant in Phase 4 with a shell.** `blueprint-writer` has `Read, Write, Glob,
Grep`; `blueprint-validator` has `Read, Grep`. Neither can run a command. Every instruction anywhere
in this plugin that says a block must be *executed* before shipping — `templates/blueprint-template.md`
§10 Bootstrap, `knowledge/stack-compatibility.md` — resolves to **you, in Step 6**. If you do not run
it, nobody does.

---

## Preconditions

Do not start until all of these hold. They come out of `questions/phase-3-confirmation.md`:

- Zero outstanding `[NEEDS CLARIFICATION]` markers
- Gate B (adversarial pre-mortem) run, survivors routed
- The user approved the architecture

If any is false, go back to Phase 3. A blueprint generated over an open marker bakes the guess in
permanently — the builder agent has no one to ask.

---

## Before Step 1 — tell the user how long this takes

**One message, before you run anything.** Generation is the only stretch of this flow where the user
sits in silence, and it is by far the longest. Steps 1 through 7 together run **roughly 25–35
minutes for a bundle** and **12–20 for a single file**: live registry calls for every pin, a full
composition pass, at least one validator round trip, and a live smoke test in a scratch directory
that runs the bootstrap **twice**, builds step 1, **runs the thing the build produces**, and runs
**the first build step that touches the database** (Step 6). Nothing streams back while it happens.

A user who was not warned does not experience that as thorough. They experience it as hung, and the
previous version of this tool was liked precisely because it answered fast. Silence is the worst
option available to you here.

Say three things, in three lines, in the user's language:

1. **What you are about to do** — verify every version against the live registries, compose the
   blueprint, validate it, run its bootstrap *twice*, run *what it builds* and *its first database
   step* once for real, write the files.
2. **Roughly how long, as a range in minutes.** Give the real number. An honest half hour beats a
   cheerful "one moment" followed by twenty-five minutes of nothing.
3. **That there is no intermediate output** — the next thing they see is the finished path and the
   first command.

> "Generating now. I verify every version against the live registries, compose the bundle, run the
> validator over it, and run its setup commands plus its first database step once in a scratch
> directory to prove they work — about 30 minutes, with no output until it's done. What comes back
> is the file path and the first command to run."

Then start. Do not ask permission — the user already approved the architecture at the Phase 3 gate.
If a later step blows well past the range you gave (a validator loop that will not converge, a
registry that is down), say so in one line rather than extending the silence.

*Done when:* a time range and a one-line description of the work are in the conversation, before the
first registry call.

---

## Step 1 — Verify every version before pinning it

**Never write a version number from memory.** v1 of The Architect hardcoded pins into thirteen files
and went stale in four months. This step is the structural fix.

Delegate to the **`stack-researcher`** subagent. It checks live registries and returns current
versions with their compatibility notes. If it cannot be dispatched, do the registry checks yourself —
see *Never hard-depend on a subagent* at the end of this file. The delegation is optional; the
checking never is.

Hand it:

- The approved stack, layer by layer (framework, runtime, styling, ORM, test runner, package manager…)
- The chosen runtime track file, so it knows which ecosystem's registry to hit
- Any constraint the user gave ("must run on their existing hosting", "team is on the LTS line")

Get back, per layer: current version, whether it is stable or pre-release, peer-dependency ranges,
and any rename or breaking change since the last major.

Rules:

| Rule | Why |
|---|---|
| Every pin in the blueprint traces to a result from **this session** | Yesterday's answer is already a guess |
| Never pin a pre-release as if it were stable | A `^1` on a package still in RC breaks installs |
| Never assert a peer range you did not read | "Framework X requires Runtime Y" is the single most confidently-wrong claim models make |
| If a version cannot be verified, write "verify before install" in the blueprint | An honest gap beats a wrong pin |

Pins live in the blueprint's stack table and nowhere else. Prose says "the project's ORM", not the
number.

**Authority:** the `stack-researcher` report produced in *this session* wins. The chosen
`knowledge/runtime-tracks/<track>.md` is the **fallback** for any package the researcher did not
resolve — and its unverified caveats carry through into the blueprint verbatim. A track file is a
cache, not a source of truth: right the day it was written, drifting ever since. Never write a pin
from memory, and never let a cached file override a live registry check.

Every pin ships with its provenance — package, version, source URL, date checked — into the
blueprint's version-provenance table. A package that was not researched says so, rather than
implying a verification that never happened.

*Done when:* every layer in the stack table carries a VERIFIED or UNVERIFIED status from a
`stack-researcher` report dated today, each with a source URL and a checked date. Zero unlabeled
layers.

---

## Step 2 — Decide the emission mode, and announce it

**Do not ask.** This used to be a question, and it was the wrong one: it is the only decision in the
whole flow whose answer changes *zero* design decisions, it was asked last, after the architecture
was already approved, and it was phrased in the tool's own vocabulary rather than the user's.
Non-negotiable rule 3 says be opinionated and recommend one option — asking here contradicted it.

**Derive it from the step count** in the build order you drafted in Phase 3:

| Steps in the build order | Mode | Output |
|---|---|---|
| **12 or more** | **Bundle** | `./blueprints/<project-slug>/` |
| **11 or fewer** | **Single file** | `./blueprints/<project-slug>-blueprint.md` |

Why twelve: below it a builder holds the whole blueprint in one context and `tasks.json` buys
nothing but files to keep in sync. At or above it the build spans sessions, and resumable state is
the difference between finishing and starting over. The template's step budget is 10–18, so both
sides of this threshold are reachable — it is a real split, not a formality.

**An explicit user preference wins, always.** If the user said "just give me one file", "keep it in
a bundle", or anything equivalent at any point in this conversation, honor it and skip the
derivation entirely. Never re-ask something they already answered.

**Announce the choice in one line — a statement, not an opening for discussion:**

> "Bundle: 15 steps, so this build spans sessions and `/architect-next` needs `tasks.json`."

Both modes carry identical acceptance criteria and verify commands; the difference is packaging
only. If the user pushes back, re-emit in the other mode — that costs one generation pass, not a
redesign. Say so in one line rather than debating it.

**This step is authoritative on emission mode.** A Mode table in `commands/architect.md`,
`commands/architect-brownfield.md` or `commands/architect-quick.md`, or the emission-mode block in
`templates/blueprint-template.md`, is input to the threshold above — never a competing decision, and
never a reason to reopen it with the user. **Any of those files that still says to *ask*, *confirm*,
or *recommend and get a yes* on the mode is stale and does not override this step.** Derive, then
announce. The only input that outranks the threshold is an explicit preference the user already
volunteered.

*Done when:* the mode is stated in the conversation in one line with the step count it came from,
and that step count is the one that goes into the blueprint's build order.

---

## Step 3 — Output layout

Slug the project name to kebab-case. `<project-slug>` below is that slug. This is the canonical
layout — there are no variants.

### Bundle

```
./blueprints/<project-slug>/
├── blueprint.md          # the 20-section narrative artifact — templates/blueprint-template.md
├── tasks.json            # the machine-readable task DAG — templates/tasks-schema.md
├── epics/
│   ├── 01-<name>.md      # one file per epic — templates/epic-template.md
│   └── 02-<name>.md
└── workspace/            # copied INTO the target project root by the builder, as-is
    ├── CLAUDE.md         # templates/claude-md-template.md
    ├── AGENTS.md         # tool-neutral stub — claude-md-template.md, "Companion files"
    ├── <verify-critical config>   # every config a §9 Verify needs to run — blueprint §19.6
    └── .claude/
        ├── settings.json          # allowlist covering every verify command in blueprint §9
        ├── skills/<name>/SKILL.md # repeatable project workflows — blueprint §19.4
        └── rules/<name>.md        # paths:-scoped conventions — blueprint §19.5
```

`workspace/` exists for exactly one reason: the builder copies **one directory** into the target
project root and is done. Say that explicitly in the handoff.

**`<verify-critical config>` is a set, not one file, and it is not optional.** It expands to the real
files §19.6 names — test-runner config, e2e-runner config, test setup or env-bootstrap file,
path-alias config, the local service compose file, and every file a `Verify` command names as an
argument — each at the path it occupies in the target project, because `workspace/` mirrors the
target repo layout exactly. The fixed entries above are literal; this row is the variable one. It is
empty only when §19.6 itself says `NOT APPLICABLE`, and a bundle whose §9 runs a test runner with no
runner config under `workspace/` fails at its first gate with an error that looks like broken code.

**`.claude/commands/` is never emitted, in either mode.** A slash command only fires when a human
types it, and an autonomous builder types nothing — a scaffolded command is dead weight that is
never invoked once. Repeatable project workflows go in `.claude/skills/<name>/SKILL.md`. Resuming is
`/architect-next`, a plugin command; it never needed a per-project copy.

Every epic file cross-links to its tasks in `tasks.json`, and every task carries its epic id. If they
disagree, the validator fails the run.

### Single file

```
./blueprints/<project-slug>-blueprint.md
```

Everything inline, in template order. Section 19 emits the workspace files as **fenced code blocks**
the builder copies out, instead of as real files. No `tasks.json`, no `epics/` — resume is manual.
No external references: a reader with zero context must be able to build from this one file alone.

### Both modes carry

- **Acceptance criteria in EARS form** — WHEN `<trigger>` THE SYSTEM SHALL `<observable response>`
- **A verify command per step** — the exact command, and what passing looks like
- Never `Done when: it works`. Always `Done when: pnpm test src/auth passes 4 tests`.
- All **20 numbered sections** of the blueprint template, even the inapplicable ones — those get
  `NOT APPLICABLE — <reason>` under the heading. Downstream tooling indexes by number.

*Done when:* the target paths for the chosen mode are written out literally, and `./blueprints/`
exists in the user's cwd.

---

## Step 4 — Compose via `blueprint-writer`

Delegate composition to the **`blueprint-writer`** subagent. A full blueprint is long; generating it
in the main thread floods the interview context and degrades everything after it — but if the
subagent cannot be dispatched, compose it in the main thread anyway rather than blocking. See *Never
hard-depend on a subagent* at the end of this file.

Hand it a complete brief — the subagent has no memory of the interview:

- Project name, slug, one-paragraph vision, target user
- Chosen shape, runtime track, and capability files (by manifest path — it reads them itself)
- The stack table **with the verified pins from step 1**
- Data model, scope in / scope out, and the Non-Goals from Gate B
- The Risk Register survivors from Gate B
- The design system decided in Phase 3 — real hex values, type pairing, component style
- Output mode from step 2, and the exact target paths
- The user's language, so the blueprint is written in it — and, when it is not English, an explicit
  line telling the writer to apply `knowledge/localization.md`: numbered headings keep their number
  and their English name, the slug and every path are ASCII, `tasks.json` keys and `status` values
  and the EARS keywords stay English, and the Simplified-Chinese vague-word blacklist applies to
  every acceptance criterion exactly as the English one does
- **The verify-critical config files it must emit** — say it explicitly in the brief, because this is
  the part writers skip. Every config file a §9 `Verify` command needs in order to run is emitted in
  §19.6 as a real file with complete content: test-runner config, e2e-runner config, test
  setup/env-bootstrap file, path-alias config, the local service compose file with pinned image tags
  and a healthcheck, and every file a `Verify` names as an argument. In bundle mode those land under
  `workspace/` at their target-repo paths (Step 3's `<verify-critical config>` row); in single-file
  mode they are fenced blocks in §19.6, each labelled with its destination path. Naming a file in §3's
  tree is not emitting it.
- **Who authors the package manifest**, stated as a decision and not left open — see *Who authors the
  package manifest* immediately below. Say which of the two origins applies to this stack, in the
  brief, in one line.
- **That §10's Bootstrap block will be executed verbatim in Step 6 — twice, back to back, with the
  bundle sitting inside the scratch project at `blueprints/<slug>/`**, so every command in it must be
  non-interactive, in the order it is run, safe on a tree that already has everything, and exiting 0
  on its own no-op path — no TTY prompts, no "then configure as needed", no guard that reports
  failure when it correctly skips.
- **That Step 6 will RUN whatever the build produces** — the binary, the published entry point, the
  container, the served endpoint — at the earliest step that produces it, not merely build it. A
  build config and a manifest that disagree about where the artifact lands are caught there.

### Who authors the package manifest — decided here, not left to the writer

The **manifest** is whatever file the track's dependency manager reads: `package.json`,
`pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, `composer.json`. On a greenfield build nothing in
this plugin said who produces it, and two defensible readings existed — §19.6 requires emitting every
config a `Verify` needs, and §10's Bootstrap installs dependencies *before* step 1, which forces the
manifest to pre-date step 1; while validator finding #18 pushes against a build step that is already
satisfied by the blueprint. Only one of the two readings actually builds. **This is the ruling. Put it
in the brief.**

One question decides it: **does a command in §10's Bootstrap block generate the manifest?**

| Answer | Who authors it | What must NOT happen |
|---|---|---|
| **Yes** — Bootstrap runs a scaffolder that writes it (`pnpm create …`, `cargo new`, `uv init`, `go mod init`, `bundle init`) | **The scaffold command.** The blueprint owns the *edits* on top of it — added scripts, the engines/requires floor, an extra dependency — and each edit is a named command or an explicit written change inside §10, placed **after** the scaffold line and **before** step 1 | Do not also emit the manifest under `workspace/`. The copy would either be overwritten by the scaffolder or overwrite it, and which one wins depends on command order nobody stated |
| **No** — bare greenfield with no scaffolder, the normal case for a CLI, a library, an MCP server | **The blueprint.** Emit it as a real file in **§19.6**, under `workspace/` at the repo root in bundle mode, as a labelled fenced block in single-file mode. It is origin 2 under `templates/blueprint-template.md` §3, and the builder's single `workspace/` copy puts it in place before Bootstrap's first install command runs | Do not give §9 a step whose job is to write the manifest. Bootstrap already installed from it, so that step gates nothing — that is exactly finding #18 |

**Never split the file across both.** A stub under `workspace/` that step 1 "fills in" is two authors
and one file, which is the thing the writer rule below exists to prevent. One origin, complete
content, no `{placeholder}` fields, no "add your dependencies here".

**Finding #18 and §19.6 do not actually conflict once this is applied.** #18 is about a *build step*
that gates nothing; emitting a workspace file is not a build step and is never scored as one. The
combination #18 forbids is the third option neither rule wanted: a §9 step that authors a manifest
§10 already installed from.

The same ruling covers every other file Bootstrap consumes before step 1 exists — the lockfile
policy, the runtime-version file (`.nvmrc`, `.python-version`, `rust-toolchain.toml`), and the
compiler config when the install or the first `Verify` reads it. If Bootstrap touches it before step
1, a scaffold command makes it or `workspace/` ships it. Nothing else is available that early.

**The writer writes.** It composes *and* saves every file in the Step 3 tree. Never re-write those
files yourself afterwards — two authors with no arbiter is how a bundle ends up half-consistent.

*Done when:* `blueprint-writer` returned a path, a section count of **20/20 filled**, and an explicit
`Blocking gaps: none`. Anything less goes back to the writer before you validate.

---

## Step 5 — Validate via `blueprint-validator`

Delegate to the **`blueprint-validator`** subagent. It checks, at minimum: no unresolved markers, no
placeholder text, every build step has an observable "Done when", every task in `tasks.json` maps to
an epic, no orphan dependencies, every pin traceable to a provenance row, a complete
`workspace/CLAUDE.md`, and — the sweeps that exist because real builds died on them — that every
emitted config actually resolves what the gates import, that every standalone tool a gate invokes has
its environment loaded, that every number the blueprint asserts matches what the blueprint defines,
and that no step's `Verify` asserts repository state only that same step's `Checkpoint` could
produce.

If the subagent cannot be dispatched, run its sweeps yourself out of `agents/blueprint-validator.md`,
in order, and say in one line that the audit was self-run. See *Never hard-depend on a subagent* at
the end of this file. An unvalidated blueprint never ships; a self-validated one does.

- **Fails → fix and re-run.** Send the specific failures back to `blueprint-writer`, or patch small
  ones directly.
- **Never present a blueprint that has not passed.** A validated blueprint is the entire product.
- If validation fails three times on the same item, stop and ask the user — that item is a design
  gap wearing a formatting bug's clothes.

*Done when:* `blueprint-validator` returned PASS with 0 BLOCKER and 0 MAJOR findings.

---

## Step 6 — Smoke-test the bootstrap, the entry point **and the data layer**. Only you can run it.

The validator reads. **This step executes.** A blueprint can pass every sweep in Step 5 and still be
unbuildable, because the failures that kill step 1 are not visible from reading: a scaffolding tool
that ignores the flag it documents, an approval prompt with no `--yes`, a generator that installs a
package the blueprint said to skip, a peer range that only conflicts once the resolver runs. Reading
cannot catch any of those. Running catches all of them, in minutes, before the user has the file.

**Stopping at step 1 is not enough, and this is settled by evidence rather than opinion.** Four live
build tests ran this step, passed it, and died anyway — each one somewhere the previous version of
this step never reached:

| Build test | Where it died | What this step was missing |
|---|---|---|
| 1 and 2 | the first command that loaded something *other than the toolchain* — a migration CLI with no environment, a runner config that could not resolve a mandated import guard, a seed script that died before its first query | run 5, the data layer |
| 3 | the last line of Bootstrap, because the bundle's `workspace/` held a second root config and the formatter found two roots in one tree and exited 1 before checking a file | run 0 — the scratch tree never contained the bundle, so a defect that only exists *with the bundle in place* could not reproduce |
| 4 | step 8, on two blueprint-authored files that disagreed about where the binary lands — a build config emitting one path, a manifest declaring another, ~30 `Verify` commands naming the loser | run 4 — nothing invoked the binary until step 8, so seven steps of green gates preceded the discovery. A stuck builder's other move, re-running Bootstrap, hit a no-clobber copy that exits 1 on macOS and 0 on Linux — run 2 |

Every one of those is a one-line fix discovered too late. This step is where "too late" is bought
back, and it now costs about five minutes.

**Only the main thread can do this.** `blueprint-writer` and `blueprint-validator` have no `Bash`.
Every "run this before shipping" line in the templates and in `knowledge/stack-compatibility.md`
is addressed here.

### The runs, in order

Everything happens in **one scratch directory outside both the bundle and the user's project** —
`mktemp -d`, never the cwd, never the real `./blueprints/`. Each run depends on the one before it.
**Stop at the first failure and route it**; do not push past a broken run to collect more findings.

| # | Run | Ends when | Failure routes to |
|---|---|---|---|
| **0** | **Place the bundle** in the scratch tree at `<scratch>/blueprints/<slug>/` — the exact path §19 and Step 7 give it — **before any blueprint command runs** | the bundle is on disk *inside* the scratch project root | not a blueprint failure: if you cannot place it, the bundle is not the shape Step 3 defines → Step 7's tree check, then the writer |
| **1** | **§10's Bootstrap block, verbatim, in order** — every command, start to finish | every command exits the way the blueprint says it does | *On failure* → writer |
| **2** | **§10's Bootstrap block again**, unchanged, same directory, immediately after | **the whole block exits 0** and the tree is still usable | *On failure* → writer, validator finding #33 — a guard that fails on the path it guards against |
| **3** | **Step 1** — implement its deliverables **from the blueprint's own literal content**, then run its `Verify` block | every Verify command matches the expected result its trailing comment states | *On failure* → writer. **Anything you had to supply that the blueprint does not state is itself the finding** |
| **4** | **The entry point** — run whatever the earliest producing step produces: the binary, the published entry, the container, the served endpoint | it **runs** and exits 0 / returns the documented status | *On failure* → writer, validator findings #30 and #31 |
| **5** | **The data layer** — the earliest §9 `Verify` that executes against it, plus everything §9 says must run before it | it passes as written, in §9's order | *On failure* → writer. Environmentally blocked → **not smoke-tested past the toolchain** |
| **6** | **The formatter/linter check the blueprint itself mandates**, over the whole tree **with the bundle present** | exits 0 | *On failure* → writer, validator findings #22 and #32 |

If the bootstrap needs environment variables, seed them from the `.env.example` the blueprint
specifies, using §10's literal local values. If a command still needs a real secret, it is a
partial-run case below — never invent a credential to get past a gate.

### Run 0 — the scratch tree must contain the bundle

**Copy the emitted bundle into the scratch tree at `<scratch>/blueprints/<slug>/` before the first
Bootstrap command.** In single-file mode, place the blueprint at
`<scratch>/blueprints/<slug>-blueprint.md`. That is where Step 7 puts it and where §19 says it
lives — a smoke test run in an empty directory is testing a layout no builder will ever have.

Three of the things this step exists to catch are **only observable with the bundle in place**:

- §10's `workspace/` copy has nothing to copy *from* if `workspace/` is not in the tree — the copy
  either silently no-ops or fails for the wrong reason, and run 2's guard check tests nothing.
- The nested-config failure (validator finding #32) exists *only* because the project's own tools
  walk a tree that contains a second copy of their config. Absent bundle, absent defect, absent
  finding — this is exactly why build test 3 shipped and then died on Bootstrap's last line.
- Run 6's formatter check is meaningless over a tree the formatter would never have seen.

Two rules on the copy:

- **Copy, never symlink, and never run against the real bundle.** This step writes, installs and
  deletes; the artifact in the user's cwd is never touched by it.
- **Place it exactly where §19 says, not somewhere convenient.** `blueprints/<slug>/` from the
  project root. A bundle parked one directory up reproduces nothing.

### Run 2 — Bootstrap, a second time, must exit 0

Run the **entire** §10 block again, unchanged, in the same directory, the moment run 1 succeeds.
Execute it the way an unattended runner does: each command's status checked, the first non-zero
aborting the block. **The block must exit 0.**

This is the only way a guard that fails on its own no-op path ever surfaces, and re-running Bootstrap
is the first thing a stuck builder does. Build test 4 found a no-clobber copy that **exits 1 on
BSD/macOS when it skips an existing file and 0 on GNU** — the same line, passing on one platform and
aborting the recovery path on the other, with nothing about it visible from reading. Only a second
execution decides it, and only on this machine's platform.

Then prove the tree survived it, with two cheap checks:

1. The package manifest still lists every dependency the install put there — the guarded
   `workspace/` copy did not revert it.
2. One command from run 1's tail still finds its binaries.

A block that exits 0 but reverts the manifest is the same finding wearing different clothes: the
failure surfaces one command later as a missing binary and reads as a broken install. Both halves
route to *On failure* as validator finding #33, and §20.1's re-run gate is what they prove.

### Run 3 — implement step 1 from the blueprint, then gate it

**Bootstrap does not do step 1's work.** It installs the toolchain and lays down the tree; step 1
authors files. So "run step 1's `Verify` block" on its own is unsatisfiable for any blueprint whose
step 1 creates something — the gates fail on files nobody wrote, and a literal reader either reports
a false failure or quietly skips the run. Neither is the intent. **This is the intent:**

1. **Implement step 1's deliverables yourself, in the scratch tree, using ONLY the blueprint's own
   literal content** — the file bodies, fenced blocks, commands and instructions §9 step 1 contains,
   plus whatever it explicitly points at in §19.6, §3 or another section. You are standing in for a
   zero-context builder: you may copy from the blueprint, you may not supply from your own knowledge.
2. **Then run step 1's `Verify` block,** every command, each against the expected result its trailing
   comment states.
3. **Step 1 only — and this restriction is scoped to run 3.** Do not implement step 2 to make step
   1's gate pass, and do not repair step 1's content to make it compile. **Run 5 is governed by its
   own rule, not by this one:** it explicitly requires running the data-layer `Verify` *plus
   everything §9 says must run before it*, which for most blueprints means implementing the
   intervening steps. That is expected there and forbidden here. The two never collide, because they
   are answering different questions — run 3 asks whether step 1 is buildable *in isolation*, run 5
   asks whether the data layer is reachable *in §9's order*.

**If implementing step 1 requires anything the blueprint does not state — a file body it never gives,
a name it never fixes, a decision it leaves open, an import path you had to infer, a command you had
to invent — stop: THAT is the finding**, and it is the entire point of the exercise. This run tests
whether step 1 is buildable from the text, not whether you can build it. Send it back to the writer
naming exactly what you had to supply.

Two outcomes both count as a pass: the gates pass on what the blueprint told you to write; or step 1
is genuinely a no-op on top of Bootstrap (it only configures something Bootstrap already made) and
its `Verify` passes as written — say so in one line and move on.

### Run 4 — run the entry point, do not merely build it

**If this project produces an executable, a published entry point, a container image or a served
endpoint, something in this step has to RUN it.** Building proves the compiler was happy. Running
proves the output landed where the manifest says it lands, that the runtime can find it, that the
permission bit and the interpreter line are right, and that the two blueprint-authored files agree.

Find **the earliest §9 step that produces such an artifact** — blueprint-template §9 rule 13 requires
that step's own `Verify` to run it, so it should be step 1 or 2. Then:

| Artifact | What run 4 is |
|---|---|
| Binary / CLI | invoke it — `--version`, `--help` — assert **exit 0** and the documented output |
| Published entry point / library | import or require it through its declared entry field, assert it loads |
| Served endpoint | start it, request the documented path, assert the documented status |
| Container image | `docker run --rm <image> --version`, assert exit 0 |

Three situations, one verdict each:

| Situation | What to do |
|---|---|
| The producing step is step 1 or 2 | Implement it the way run 3 implements step 1 — literal blueprint content only — then run the artifact. Usually run 3 already produced it and run 4 is one command |
| The first runnable artifact appears **later than step 2** | That is itself a finding (validator #31 — the blueprint did not pull it forward). Record it. Then do the check statically rather than implementing five steps: take the literal output path from the build config emitted in §19.6, the literal entry path from the manifest, and **every** §9 `Verify` and §20.1 command naming either, and compare them character for character. A mismatch is validator finding #30 and blocks — it is precisely the `dist/cli.js` vs `dist/cli/index.js` defect, caught at generation instead of at step 8 |
| Nothing this build produces is ever run or imported by anything | Rare, and say so in one line. A library still has a published entry point; a site still serves a page. Before writing this, check §9 and §20.1 for a command that executes an output — if one exists, run 4 has a target after all |

The comparison rule from run 3 applies here too: **if you have to type a path the blueprint does not
contain in order to find the artifact, that is the finding.** The builder will not know to type it
either.

### Run 6 — the mandated formatter/linter, over the tree, with the bundle in it

This is the half of validator Sweep 12 handed to you by name, because the validator has no shell and
cannot run a formatter. Run **the check command the blueprint itself mandates** — `<formatter> check
.`, `<linter> .` — from the scratch project root, over the tree that run 0 put the bundle into and
run 1 copied `workspace/` out of.

Two distinct defects come out of one command:

- **The emitted files do not pass the project's own gates** (validator finding #22) — a live run
  caught two real mismatches here that no amount of reading had found.
- **The bundle is part of the tool surface** (validator finding #32) — the tool walks the tree, finds
  a second root config under `blueprints/<slug>/workspace/`, and exits 1 before checking a single
  file. Neither config is wrong; the defect is that both exist and nothing excluded the bundle path.
  This is the failure that killed build test 3's Bootstrap.

If §9 step 1 or §20.1 already runs that command, runs 3 and 4 covered this for free — read their
output rather than running it twice. Either way, **do not reformat the file or add the exclude line
yourself**: both route to *On failure* like every other finding.

### Finding the data-layer command (run 5)

Walk §9 from step 1 and take the **earliest** `Verify` command that does any of these. Do not take
the first one that merely mentions the database in prose — take the first one that *executes* against it:

| It qualifies when the command… | Canonical shapes |
|---|---|
| Applies or generates schema | a migrate, push, generate or sync command |
| Writes or clears data | a seed, reset or fixture-load script |
| Runs a test that imports a server-side module | the first integration/API/repository test, not a pure-UI or pure-unit test |
| **Reads or writes real files on disk** — the whole data layer for a tool whose storage *is* the filesystem | the first `Verify` that runs the tool against a real input file and asserts on the output file it produced |
| Starts the local service the gates depend on and proves it accepts connections | the compose up plus the first real query against it |

Then run **everything that command depends on**, in §9's order, or it is not a real test: the service
must be up, the schema applied, the env loaded exactly the way the blueprint says it is loaded.

**Implementing the intervening steps IS expected here.** When the data-layer gate sits at step 5,
you implement steps 2, 3 and 4 the way run 3 implements step 1 — from the blueprint's own literal
content — and then run the gate. Run 3's "step 1 only" is a rule about run 3 and does not bind this
run; if it did, run 5 would be unreachable for every blueprint whose storage gate is not at step 1 or
2, which is most of them. **The literal-content rule still binds, and it is the finding-producing
half:** anything you must supply that the blueprint does not state — for any of those intervening
steps, not just the data-layer one — is itself the finding, exactly as in run 3. Send it back to the
writer naming the step and what you had to supply.

If
you find yourself typing a command the blueprint does not contain in order to make this work — an
export, a `cd`, a flag, a wait — **stop: that is the finding.** The builder will not know to type it
either. Send it back to the writer rather than typing it yourself.

That last rule is the whole value of run 5 — and of runs 3 and 4, which apply it in their own words.
Every defect the last four build tests hit had this exact signature: a step that runs only if you
already know the one thing the blueprint never says.

**A data layer that needs no service is a CLEAN PASS of run 5, not a blocked one.** Plenty of correct
architectures store state without anything to start: an embedded or in-process database, a
single-file store, an append-only log, a content-addressed cache, an index built from files on disk,
and — the simplest case, and the one to be unambiguous about — **plain filesystem I/O: a tool whose
entire data layer is the files the user points it at**, read from and written to directly, with no
store of any kind in between. A CSV merger, a log processor, a static-site generator and a
file-format converter are all in this category, and every one of them has a data layer for run 5's
purposes. Do not reason by analogy from the embedded-database row: this row is named on its own.
For those, run 5 is *easier*, not skipped — the earliest `Verify` that applies schema, writes or
clears data, or runs a test importing the storage module still exists, and you still run it and its
prerequisites in §9's order. It passes or it fails on the blueprint's merits, exactly like any other.
This has been executed for real: a live run took this run against an embedded database, with no
service anywhere in the stack, and passed fully.

So do not reach for the "not smoke-tested past the toolchain" note because the stack has no container
in it. That note is for a data layer you **could not reach**, never for one that had nothing to
start. A blueprint whose store is in-process and whose run 5 passed is **smoke-tested**, full stop,
and reporting it any weaker under-sells a build that was verified end to end.

Three situations that look adjacent, one verdict each:

| Situation | Run 5 verdict |
|---|---|
| The store is embedded/in-process/file-based — **including plain filesystem I/O over the user's own files** — and its first `Verify` ran | **Clean pass.** Report it as `bootstrap ×2, step 1, <entry-point command> and <data-layer command> verified` |
| The data layer is genuinely serviceless *and* there is no schema, no write, no seed and no test that touches storage anywhere in §9 | Say so in one line — run 5 has no target, and that is a finding about §9, not about this machine. A blueprint with a data model and no gate that exercises it goes back to the writer |
| A service *is* required and this machine cannot start it | The environmental case below |

Rules for the run:

| Rule | Why |
|---|---|
| Scratch directory, deleted when the step ends | The user's cwd is not a test fixture, and a half-scaffolded project left behind is worse than no test |
| One scratch directory for runs 0 through 6, in that order | Runs 2, 4 and 6 test what the *previous* runs left behind. A fresh directory per run tests nothing they exist for |
| Non-interactive only — never answer a prompt by hand | A command needing a human here needs one during an unattended build too. The prompt *is* the defect |
| Cap each command; kill anything still running after ~5 minutes | A hang is indistinguishable from slow work, and this step must not blow past the time range you gave before Step 1 |
| Never run a command that writes outside the scratch directory or touches a live account | A blueprint's bootstrap can create real cloud resources — read it before you run it, and skip those commands under the partial-run rule below |
| Run every command **exactly as the blueprint writes it** — same working directory, same flags, same env loading | A command that only works with your improvement is a broken command. You are standing in for a builder who cannot improvise |
| Tear the local services down before deleting the scratch directory | A container left running holds a port, and the next run fails for a reason that has nothing to do with the blueprint |

### On failure

**A failed command is a validator finding.** Route it exactly the way Step 5 routes one — do not
patch the blueprint to match what happened to work, and do not soften the step so it stops failing:

1. Send `blueprint-writer` the failing command verbatim, its exit code, and the last ~20 lines of its
   output. That output is the specification for the fix — a real error message beats any guess.
2. Take back the corrected blueprint, **re-run Step 5**, then re-run this step **from run 0** in a
   fresh scratch directory. A fix that was not re-validated is not a fix, and a run that starts at
   run 3 is not this step.
3. **Three failures on the same command → stop and ask the user.** Same rule as Step 5, same reason:
   at three, it is a stack problem wearing a command's clothes, and the honest move is to say so.

**A missing instruction is a failure, even when nothing exited non-zero.** Runs 3 and 4 fail by
*needing something the blueprint does not contain* — a file body, a name, an inferred import path, a
command you had to invent. Report those with the same weight as a non-zero exit: name exactly what
you had to supply, and let the writer put it in the blueprint. That is the defect a builder hits at
step 1, and it has no error message attached.

### When it genuinely cannot run

Some bootstraps cannot execute here: no network, a toolchain this machine does not have, a paid
credential the user has not created yet, a command that would provision real infrastructure. Run 5
adds its own: **a data layer whose service this machine cannot start — no container runtime for the
local database, or a managed data service that needs the user's own credentials.** Run 4 can hit the
same wall when the artifact is a container image and there is no container runtime — report that the
same way, naming the entry-point command you could not run.

**Runs 0, 2 and 3 have no environmental excuse.** Placing a directory, executing a block a second
time, and typing what the blueprint literally says all work on any machine that got through run 1. If
one of them did not happen, it was skipped, and Step 8 says **not smoke-tested** — not a partial pass.

Read that blocker as *a service is required and unavailable*. A stack whose data layer needs no
service at all is the clean-pass case above and never lands here — an in-process store is a design
choice, not a degraded environment, and treating it as a blocker reports a verified build as an
unverified one. Then:

- **Run everything up to the blocking command** and report a partial pass — "bootstrap verified
  through `pnpm install`; `supabase link` needs the user's project ref". A partial run is worth far
  more than none, and it still catches the scaffolding failures, which are the common ones.
- **Skipping is allowed only with an explicit, unmissable note in the output.** Step 8's handoff
  carries one line, in the user's language, saying the blueprint was **not smoke-tested**, exactly
  which commands were not run, and why. Never let a skip pass silently — an untested bootstrap
  presented like a tested one is the failure this whole step exists to prevent.
- Never skip because the run looks slow, or because Step 5 passed. Step 5 passing is not evidence
  about this step; they check different things.

**When runs 0–4 pass and run 5 cannot execute, you say so in those words.** The blueprint is
**smoke-tested through the toolchain only, not past it** — not "smoke-tested". Name the data-layer
command you could not run and the environmental reason, and carry that exact wording into Step 8:

> "Bootstrap verified twice, step 1 and `cli --version` verified. **Not smoke-tested past the
> toolchain** — step 3's `pnpm db:migrate` needs a container runtime this machine does not have, so
> the data layer is unproven."

The distinction is the whole point of this revision. A blueprint whose toolchain runs and whose data
layer was never touched is exactly the artifact two failed build tests produced, and reporting it as
"smoke-tested" is what let it reach a builder twice. **Only an environmental blocker earns this
note** — a required service you cannot start, no credentials, no network. Two things are never
environmental blockers: a command that fails because the blueprint is wrong (that is *On failure*, and
it goes back to the writer), and a data layer that needs no service in the first place (that is a
clean pass — run it, and report it as verified).

*Done when:* runs 0 through 6 have each been executed in one scratch directory, in order — the bundle
placed at `blueprints/<slug>/` before anything ran, **§10's Bootstrap block executed twice with the
second run exiting 0**, step 1 implemented from the blueprint's own content and its `Verify` block
passed, **the artifact the earliest producing step produces actually run and exited 0**, the first
data-layer `Verify` and its prerequisites passed, and the mandated formatter/linter check passed over
the tree with the bundle in it — and either **all exited as the blueprint says they should**, or the
exact unexecuted commands, their environmental reason, and — if run 5 was among them — the words
**not smoke-tested past the toolchain** are written down for Step 8's handoff. The services are torn
down and the scratch directory deleted either way.

---

## Step 7 — Write location

**Write to the user's current working directory. Never inside the plugin.**

`${CLAUDE_PLUGIN_ROOT}` is a read-only cache that gets wiped on update. Anything written there is
lost and unreachable from the user's project.

- ✅ `./blueprints/<project-slug>/blueprint.md`
- ❌ `${CLAUDE_PLUGIN_ROOT}/blueprints/...`
- ❌ `output/` — the v1 location; it lived inside the repo and only worked in clone mode

If `./blueprints/` does not exist, create it. If a blueprint with that name already exists, say so
and ask before overwriting.

### Where the bundle sits relative to the code

**Every verify command in the blueprint runs from the TARGET PROJECT root** — never from the bundle
directory, never from an unrelated design directory. `pnpm test src/auth` means *that project's*
`pnpm` and *that project's* `src/`. Write the bundle where that relationship is unambiguous:

There is still only one location rule — `./blueprints/<project-slug>/`, relative to your current
working directory. The table below does not add a second one; it says **where that cwd should be**.

| Situation | Where the bundle goes |
|---|---|
| The project directory already exists (brownfield, or the user scaffolded it) | Run the design session **from inside the target project**, so the cwd *is* the project root and the one rule resolves to `<target-project>/blueprints/<project-slug>/`. If you are not already there, `cd` into it before writing. `/architect-next` then finds `tasks.json` and runs verify commands from that same root, with nothing to explain. |
| The project does not exist yet | The design session's cwd is fine — `./blueprints/<project-slug>/` there — but the handoff must say, in one line, that the bundle moves into the project root once it exists, so that the cwd-is-project-root relationship holds before `/architect-next` runs. |

`/architect-next` stops when the bundle is not inside the project it builds. That is correct: an
unrelated cwd makes every verify command fail for reasons that have nothing to do with the code.

*Done when:* every file in the Step 3 tree exists on disk under the user's cwd, and the bundle's
relationship to the target project root is settled. Check the tree the way Step 3 defines it: the
**fixed entries are literal** — `blueprint.md`, `tasks.json`, `epics/`, `workspace/CLAUDE.md`,
`workspace/AGENTS.md`, `workspace/.claude/settings.json` — and the `<verify-critical config>` row is
a **set**, satisfied when every file §19.6 lists exists under `workspace/` at its target-repo path.
An `ls` that matches the fixed entries and nothing else is a *failure*, not a pass: a bundle whose
§19.6 names a runner config that is not on disk cannot run its own first gate.

---

## Step 8 — Hand off

Short summary. Do not restate the architecture — the user just approved it.

1. **Path** — the exact absolute file or directory written
2. **Shape** — what it is and how many build steps, in one line
3. **Any "verify before install"** flags from step 1
4. **The smoke-test result from Step 6** — one line, and it must distinguish three outcomes, not two:

   | What happened | The line to write |
   |---|---|
   | Runs 0–6 all ran — bootstrap twice, step 1, the entry point, the data layer, the format check — **including every stack whose store needs no service to start** | "bootstrap verified twice, step 1, `<the entry-point command>` and `<the data-layer command>` verified in a scratch directory with the bundle in place" |
   | Runs 0–4 ran, run 5 blocked environmentally — a **required service** could not be started here | **"not smoke-tested past the toolchain"** — name the data-layer command and the reason |
   | Blocked before that, or skipped | **"not smoke-tested"** — name what was not run and why |

   Never omit this line; a silent omission reads as a pass. And never write plain "smoke-tested" when
   the data layer was never touched — that sentence is what shipped two unbuildable blueprints.
5. **Where the verify commands run from** — the target project root (see Step 7). If the bundle is
   not already inside that project, say it must be moved there first.
6. **The immediate next command**

Bundle:

```
/architect-next
```

One line before it: copy `blueprints/<project-slug>/workspace/` into the target project root — that
one directory is `CLAUDE.md`, `AGENTS.md` and `.claude/` in the places the builder expects them.

Single file: open a fresh Claude Code session **in the target project directory** and point it at the
blueprint file — a clean context is the whole reason the blueprint is self-contained. The workspace
files are fenced blocks inside it; the builder copies them out before step 1.

Then stop. Do not start building. The Architect designs; a different instance builds.

*Done when:* the user has the absolute path, the Step 6 smoke-test result, the workspace-copy
instruction, and the next command.

---

## Subagent rules

**`AskUserQuestion` is stripped from every subagent.** A subagent that needs a decision cannot get
one — it will guess, or stall.

- **Every user question happens in the main thread.** Never delegate a question.
- Resolve all ambiguity *before* dispatching. If a subagent comes back with an open question, answer
  it in the main thread — with the user if needed — and re-dispatch.
- Subagents return text and write files. They do not talk to the user.

### Never hard-depend on a subagent

Steps 1, 4 and 5 delegate to `stack-researcher`, `blueprint-writer` and `blueprint-validator`. That
is the preferred path and you should take it whenever you can. **It is not a requirement, and this
phase never blocks on it.** SKILL.md rule 11 already says never hard-depend on a third-party *skill*;
this is the same rule for *subagents*, and it exists because in two consecutive live runs the Task
tool was unavailable and all three were undispatchable, leaving the operator to improvise a fallback
that was never written down.

**If a subagent cannot be dispatched — no Task tool, the agent is not installed, dispatch errors —
do its job in the main thread, say so in one line, and keep going.**

| Subagent | Doing it yourself means | What you must not drop |
|---|---|---|
| `stack-researcher` (Step 1) | Hit the registries directly with `WebFetch`/`WebSearch`, or `npm view <pkg> version` and its ecosystem equivalents | Every pin still carries source and check date; an unresolvable pin is still written `UNVERIFIED`, never guessed from memory |
| `blueprint-writer` (Step 4) | Compose and write the Step 3 tree yourself, reading `templates/blueprint-template.md` section by section | All 20 sections filled, §19.6 emitted with real file bodies, `Blocking gaps: none` still true before you validate |
| `blueprint-validator` (Step 5) | Run the sweeps yourself against `agents/blueprint-validator.md`, as written | Every sweep, in order — most of all Sweep 10 and Sweeps 15–24. Self-auditing is weaker than an adversarial read, so slow down rather than skipping |

Three rules on the fallback:

1. **Say it once, in one line, in the user's language** — "Running the validation sweeps in the main
   thread; the subagent isn't available in this session." Not a paragraph, not an apology, and never
   silence: the user should know the audit was self-run, because that is materially weaker.
2. **The standard does not move.** A fallback changes *who* does the work, never *whether* it is
   done. Step 5's exit gate is still zero BLOCKER and zero MAJOR, and Step 6 still executes.
3. **Never stop and ask whether to proceed without a subagent.** Availability is an environment fact,
   not a design decision, and the user has nothing to add to it. Fall back and continue.

Doing the work in the main thread costs context, which is the whole reason these are subagents. So
when you fall back on Step 4, write the files section by section and keep the RUNNING BRIEF current —
a compaction mid-composition is the real risk here, not the missing agent.

---

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Builder agent asks clarifying questions | Blueprint is not self-contained | Validator should have caught it — re-run step 5 |
| `npm install` fails on a pin | Version came from memory | Re-run step 1; never skip it |
| Builder drifts mid-build | Steps too large, or no "Done when" | Split steps to one sitting each; add observable criteria |
| Blueprint contradicts itself | Composed in pieces without a final pass | Validator run must be on the finished artifact, not per-section |
| Subagent stalls or invents an answer | It hit a decision it could not make | You left ambiguity in the brief — resolve in main thread, re-dispatch |
| Step 1 of the build dies on a scaffolding prompt or an ignored flag | The bootstrap block was written from docs, never executed | Step 6 exists to catch exactly this — run it; a doc is not evidence |
| First gate fails with `No test files found` or `Cannot find module` | A verify-critical config was drawn in §3 but never emitted in §19.6 | Send it back to the writer with the missing paths — see Step 4's brief and Step 7's tree check |
| Step 1 passes, then the build dies at the first database step | Step 6 stopped at the toolchain; the data layer was never executed | Run 5 of Step 6 exists for this — run it, or report **not smoke-tested past the toolchain** |
| Seven steps of green gates, then the packaging step cannot find the binary | Two emitted files disagree about where the artifact lands, and nothing ran it until then | Run 4 of Step 6 — run the entry point at the earliest step that produces it. Validator findings #30 and #31 |
| Bootstrap's last line exits 1 with a duplicate-config or two-roots error | The bundle's `workspace/` is a second root config inside the project tree, and no emitted config excludes `blueprints/` | Runs 0 and 6 of Step 6 reproduce it — validator finding #32. Never smoke-test in a tree without the bundle |
| The builder re-runs Bootstrap to recover and the block aborts at the copy | A no-clobber guard exits non-zero when it correctly skips — `cp -Rn` on BSD/macOS | Run 2 of Step 6 — validator finding #33. The second run must exit 0 |
| A step's `Verify` exits 1 on `git status --porcelain`, or on `git ls-files --error-unmatch <file>` for a file that step just created | The gate asserts repository state that the same step's `Checkpoint` — which runs *after* `Verify` — is the only thing that could produce | Validator finding #34. Move the assertion into the `Checkpoint`, or assert the filesystem (`test -f <file>`) instead of the index. The same command is correct in the §20.1 gate |
| Run 5 looks unreachable because the data-layer gate is at step 5, not step 1 | Run 3's "step 1 only" was read as binding on run 5 | It is not — run 5 requires the intervening steps. Implement them from the blueprint's literal content; anything you must supply that the blueprint does not state is still the finding |
| Run 3 "fails" because step 1's files do not exist | Bootstrap never does step 1's work; the run was read as Verify-only | Implement step 1's deliverables from the blueprint's literal content first, then gate. Anything the blueprint does not state is the finding |
| Every server-side test dies at import, in a config that exists | The emitted runner config does not handle a package the blueprint mandates on every server module | Validator finding #25 — the config must name the package or its resolution mechanism in its own bytes |
| A migration or seed command exits 1 having created nothing | The tool reads an env var and nothing in the blueprint loads the env file for it | Validator finding #26 — the loading mechanism belongs in the command itself |
| A verify command greps for a count and gets a different one on every machine | A derived number was written from impression, not counted | Validator finding #27 — assert the named entities, not the cardinality |
| Step 1's checkpoint fails with `not a git repository` or an unresolvable `HEAD` | §10 never created the repo and its first commit | Validator finding #28 |
| Bootstrap's first install exits 1 with `no package.json` / `go.mod not found` / `no pyproject.toml` | Nobody was assigned the manifest on a greenfield build | Step 4's *Who authors the package manifest* — either a §10 scaffold command generates it or §19.6 ships it under `workspace/`. Never a §9 step |
| Generation blocks because a subagent will not dispatch | A step was read as requiring delegation | It never does — fall back to the main thread, say so in one line, continue |

---

## See also

- `questions/phase-3-confirmation.md` — the gates that must pass before this phase runs
- `templates/blueprint-template.md` — the structure `blueprint-writer` fills
- `templates/tasks-schema.md` — the `tasks.json` contract for bundle mode
- `templates/epic-template.md` — per-epic file structure
- `templates/claude-md-template.md` — the CLAUDE.md shipped with every blueprint
