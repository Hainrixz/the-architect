---
description: Run the blueprint-validator against an existing blueprint and report PASS/FAIL with specifics. Blueprints written before v2 will FAIL — they have no acceptance criteria. That is expected, not a bug. / Audita un blueprint existente con el validador; los blueprints v1 fallan porque no tienen criterios de aceptación. / 用校验器审计已有蓝图并给出 PASS／FAIL 及具体问题。v2 之前的蓝图必定 FAIL——它们没有验收标准，这是预期行为。触发："检查一下这份蓝图"、"蓝图有没有问题"。
argument-hint: "<path/to/blueprint.md | path/to/bundle/>"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Task
---

# `/architect-audit` — validate an existing blueprint

You are being invoked as `/architect-audit`. This command is **read-only**. It grades a blueprint;
it does not rewrite one.

**Say this before you dispatch anything, whenever the target has no `tasks.json` beside it:** a
blueprint written before v2 **will FAIL**, with a long finding list. It has no acceptance criteria at
all — that is the single thing the validator measures most of — plus no `tasks.json`, no epics, and
no verify commands. The failure is a diagnosis of a format gap, not evidence that the tool is broken
or that the design is bad. Setting that expectation up front is part of the job, because a
twenty-finding report delivered without it reads as an accusation.

The rubric lives in the validator, not here:
`${CLAUDE_PLUGIN_ROOT}/agents/blueprint-validator.md`. Read it before reporting, so your summary
uses the validator's own check names instead of invented ones.

## Resolve the target

`$ARGUMENTS` is a path. Handle these cases:

| Argument | Do |
|---|---|
| A `.md` file | Audit that file. |
| A directory | Glob it for the canonical bundle tree — `blueprint.md`, `tasks.json`, `epics/*.md`, and `workspace/` (`workspace/CLAUDE.md`, `workspace/AGENTS.md`, `workspace/.claude/`). Audit the bundle as a whole — a bundle missing `tasks.json` is itself a finding, and so is a `workspace/.claude/commands/` directory, which is never emitted. |
| Empty | Glob `./blueprints/*/` in the cwd, then `./blueprints/*-blueprint.md` for single-file mode. One match → use it. Several → list them and ask which. None → say so and point at `/architect`. |
| A path that does not exist | Say so. Do not guess a neighbor. |

## Run

Dispatch `blueprint-validator` with the `Task` tool, handing it the resolved path. Do not grade the
blueprint yourself — a second opinion from the same context is not a second opinion.

## Report

Print, in this order:

1. **Verdict** — `PASS` or `FAIL`, one line, no hedging.
2. **Findings table** — `Severity | Check | Where | What is wrong | Fix`. Blockers first. Point at
   section headings or task ids, never "somewhere in the build order".
3. **Coverage** — how many build steps have an observable "Done when", how many have a verify
   command, how many tasks in `tasks.json` have acceptance criteria. Report as `n/total`, not as a
   vibe.
4. **Next move** — one line:
   - Stale version pins only → `/architect-refresh <path>`.
   - Missing acceptance criteria, no `tasks.json`, no epics → the structure is not there to patch;
     re-run `/architect` and reuse the answers.
   - Existing codebase → `/architect-brownfield`.

## Expect v1 blueprints to fail

Anything generated before v2 **will FAIL**. Not might — will. It has no acceptance criteria on any
step, no `tasks.json`, no epic files, and no verify commands, and those are most of what the
validator checks. A twenty-finding report on a v1 file is the tool working correctly.

So when the audited file has no `tasks.json` beside it, open the report with one line to that effect
before the verdict, and close the coverage block by naming the format gap rather than listing the
same absence twenty times. A v1 blueprint is still a usable design document. It is just not an
autonomously buildable bundle, which is the only thing this command measures.

## Rules

1. **Do not edit the blueprint.** If the user asks for fixes after seeing the report, that is a new
   instruction — then use `/architect-refresh` for pin drift or `/architect` for structural gaps.
2. **No partial credit language.** "Mostly passes" is a FAIL with extra words.
3. **Every finding names a file and a location.** A finding you cannot locate is a hunch, and hunches
   do not go in the table.
4. Detect and use the user's language. Not English? Apply `knowledge/localization.md` — prose
   follows the user, identifiers/paths/commands/`status` values/EARS keywords stay English.
