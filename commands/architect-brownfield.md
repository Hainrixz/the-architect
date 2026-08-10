---
description: Design a change against an existing codebase — maps the repo's stack, conventions, structure and tests, then emits a blueprint for a feature, refactor, or migration instead of a from-scratch build. / Diseña un cambio sobre un repo existente — mapea stack y convenciones y emite un blueprint de cambio, no de proyecto nuevo. / 针对存量代码库设计变更——先摸清仓库的技术栈、约定、结构与测试，再输出功能／重构／迁移蓝图，而不是从零开始。触发："分析我现有的代码"、"给这个仓库加个功能"、"重构这个项目"、"迁移到 X"。
argument-hint: "[what you want to change, add, or migrate]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - WebFetch
  - WebSearch
  - Task
  - Skill
  - AskUserQuestion
---

# `/architect-brownfield` — design a change to code that already exists

You are being invoked as `/architect-brownfield`. Most coding-agent work is not greenfield —
roughly a quarter of sessions write new code, and the rest modify what is already there. This is the
entry point for the rest.

Read the skill first — the phases, the shapes, and the acceptance-criteria contract are unchanged:

- Plugin install: `${CLAUDE_PLUGIN_ROOT}/skills/architect/SKILL.md`
- Clone install: `./CLAUDE.md` at the repo root — the clone-mode mirror of the skill. Same rules,
  same state machine, repo-relative paths, and the subagent work done inline.

## Mode

| Setting | Value |
|---|---|
| Entry | Existing repo in the cwd |
| Deliverable | A **change blueprint** — feature, refactor, integration, or migration |
| Shapes | Same files. They are stack-agnostic, so they describe a system that already exists just as well as one that does not. |
| Output | Bundle or single file — **ask at phase-4 Step 2**, recommend bundle. Both land under `./blueprints/` in the repo. |
| Gate | `blueprint-validator` must return **PASS** |

## Phase 0 — map the repo (this is the phase that does not exist in the greenfield flow)

Before any question, read the codebase. Do not ask the user what you can read.

| What | Where to look |
|---|---|
| Runtime track | Package manifest + lockfile, language version files, container base image |
| Framework and topology | Entry points, routing directory, server vs client split, workspace layout |
| Conventions | Naming, module boundaries, error handling, the linter/formatter config that is actually enforced |
| Data layer | Migrations directory, schema files, ORM usage sites |
| Test setup | Runner, where tests live, how they are named, what the coverage floor is if any |
| CI and deploy | Workflow files, deploy config, environment variable surface |
| Existing agent instructions | `CLAUDE.md` / `AGENTS.md` — these outrank the plugin's defaults |

Then print a **Repo Map**: track, framework, data layer, test command, lint command, build command,
deploy target, and the 3-5 conventions the change must respect. Ask the user to correct anything you
read wrong. Their correction is cheaper than your assumption.

## Phase 1-3 — compressed interview

Skip discovery of *what the product is* — the repo already answered that. Ask about the **delta**,
max 3 questions per message, using `${CLAUDE_PLUGIN_ROOT}/questions/phase-2-branches.md` and
`${CLAUDE_PLUGIN_ROOT}/questions/phase-3-confirmation.md` as the source:

1. What should be true after this change that is not true now?
2. What must keep working exactly as it does today?
3. What is off-limits — files, services, data, downtime windows?

Then present the change architecture for sign-off: the delta, the blast radius, the rollback.

## Phase 4 — the change blueprint

Compose from `${CLAUDE_PLUGIN_ROOT}/templates/blueprint-template.md`. **Brownfield gets no sections
of its own.** The template's 20 numbered sections are fixed and downstream tooling indexes them by
number, so brownfield content goes into existing sections as named subsections:

| Brownfield content | Goes in |
|---|---|
| **Current state** — the Repo Map plus the specific modules the change touches | §1 Project Overview, as a `### Current state` subsection |
| **Target state** — what the code looks like after, in the repo's own conventions | §1 Project Overview, as a `### Target state` subsection |
| **Delta** — files added, modified, deleted; named, not gestured at | §3 Directory Structure, §4 Data Model, §5 API Design — a `### Delta` subsection in each, covering only what that section owns |
| **Interfaces held constant** — public API surface, DB columns, events, env vars that must not move | §5 API Design, as an `### Interfaces held constant` subsection. Mirror each frozen interface as a row in §1 Non-Goals. |
| **Build order** | §9, unchanged. Same numbered one-sitting steps, each with an observable "Done when" and a verify command that runs against **this** repo's test setup. |
| **Parity and cutover** | **§9.1** — required when the change is a migration (see below) |
| **Rollback** | §12 Release and rollback, plus the per-step `Checkpoint` field in each §9 step |

**All 20 sections still appear.** A section with no brownfield content carries
`NOT APPLICABLE — <reason>` under its heading. Never delete a heading and never add a 21st.

Emit the canonical bundle (`${CLAUDE_PLUGIN_ROOT}/questions/phase-4-generate.md` Step 3 is the
authority on the tree):

```
./blueprints/<change-slug>/
├── blueprint.md          # the 20-section narrative
├── tasks.json            # ${CLAUDE_PLUGIN_ROOT}/templates/tasks-schema.md
├── epics/NN-<name>.md    # ${CLAUDE_PLUGIN_ROOT}/templates/epic-template.md
└── workspace/            # the builder copies this directory INTO the repo root
    ├── CLAUDE.md
    ├── AGENTS.md
    └── .claude/{settings.json, skills/<name>/SKILL.md, rules/<name>.md}
```

In a brownfield repo `workspace/CLAUDE.md` and `workspace/AGENTS.md` **merge into** the repo's
existing files rather than overwriting them — say so in the handoff. **Never emit
`.claude/commands/`**: a slash command only fires when a human types it, and an autonomous builder
types nothing. Repeatable project workflows go in `.claude/skills/<name>/SKILL.md`.

Single-file mode emits `./blueprints/<change-slug>-blueprint.md` with everything inline, no
`tasks.json` and no `epics/`. `/architect-next` needs the bundle; single file means resume is manual.
With a bundle, `/architect-next` works on it identically to a greenfield one.

## §9.1 Parity and cutover — required when the change is a migration

Rewrites die at cutover, not at design. Any migration — framework, database, provider, language —
gets **Section 9.1** or the validator fails it. Its six parts:

1. **Parity checklist** — every behavior the old path has, as a checkable row. Including the ones
   nobody documented; go read them out of the code.
2. **Parity harness** — the command that proves old and new agree. Golden outputs, shadow reads,
   recorded-request replay. Name the actual command.
3. **Coexistence** — how both paths run at once. Flag, router, dual-write. Include how you keep data
   consistent while both are live.
4. **Cutover sequence** — numbered, each with its own "Done when" and its own rollback. Every
   "Done when" here must be decidable by a script on this machine — a step that waits on a human
   approving the switch belongs in the post-build launch checklist, not in the build order.
5. **Kill criteria** — the observable signal that says roll back now, expressed as a threshold a
   query or an alert rule can evaluate, plus the person on watch.
6. **Decommission** — deleting the old path is a task with an id, not a someday.

## Rules

1. **Never propose a big-bang rewrite.** Incremental with a coexistence window, always.
2. **The repo's conventions beat the plugin's defaults.** Every time. If the repo does something you
   would not have chosen, match it and record the disagreement in §20.3 Decision log with its
   `Would reverse if` trigger — do not quietly introduce a second style, and never park an
   unresolved question in the deliverable.
3. **Pins come from the repo's lockfile,** not from the runtime track. Verify against the registry
   with `stack-researcher` before proposing any upgrade, and keep the upgrade out of the change
   blueprint unless the change actually requires it.
4. **Design only — write no application code.** You write the bundle. A different Claude Code
   instance writes the code.
5. **Every step has an observable "Done when"** and a verify command drawn from the repo's real test
   and lint commands.
6. Detect and use the user's language. Not English? Apply `knowledge/localization.md` — prose
   follows the user, identifiers/paths/commands/`status` values/EARS keywords stay English.

## After completion

Print the bundle path, the task count, the blast radius in one line, and: run `/architect-next` from
the repo root to start the change. The bundle lives inside the repo it changes precisely so the
verify commands run against that repo — say that, so nobody moves it somewhere tidier.
