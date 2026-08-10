---
description: Resume a build — reads tasks.json from a bundle, finds the first pending task whose dependencies are all done, and prints it with its epic, acceptance criteria, and every verify command. This is what lets a long build survive across sessions. / Reanuda una construcción desde tasks.json y muestra la siguiente tarea desbloqueada. / 恢复构建——读取蓝图包的 tasks.json，找出依赖已满足的第一个待办任务，连同所属任务群、验收标准和全部校验命令一起输出。触发："继续构建"、"下一步做什么"、"接着上次的进度"。
argument-hint: "[path/to/bundle] [--list | --task <id> | --start <id> | --done <id>]"
allowed-tools:
  - Read
  - Edit
  - Glob
  - Grep
  - Bash
---

# `/architect-next` — the resume protocol

You are being invoked as `/architect-next`. A build that spans sessions needs one question answered
in a fresh context with no memory: **what do I do next, and how do I know when it is done?**

This command answers it from `tasks.json`. Nothing else. No interview, no redesign.

**Field names are defined by `${CLAUDE_PLUGIN_ROOT}/templates/tasks-schema.md` — read it first and
use its names verbatim.** If a bundle's `tasks.json` disagrees with the schema, report the mismatch
instead of guessing what a field meant.

## Locate the bundle

| Argument | Do |
|---|---|
| A directory | Read `<dir>/tasks.json`. |
| A `tasks.json` path | Read it. |
| Empty | Glob `./blueprints/*/tasks.json`, then `./tasks.json`. One match → use it. Several → list them and ask. |
| Nothing found | Say so and point at `/architect` (new project) or `/architect-brownfield` (existing repo). Do not invent a task list. |

**Verify commands run from the target project root — the repo being built — not from the bundle
directory.** Every command in a task's `verify` array is written to be runnable from that root. If
the bundle is not inside the project it builds (a design session that produced
`./blueprints/<slug>/` in some other directory), say so and stop: running `verify` from the wrong
working directory produces failures that have nothing to do with the code, and a red result nobody
can trust is worse than no result. Tell the user to move the bundle into the project, or to re-run
`/architect-next <path-to-bundle>` from the project root.

## Handle an interrupted task first

Before selecting anything, look for a task whose status is `in_progress`. That is not a ready task —
it is **crash recovery**. A previous session wrote `in_progress` before touching code and never got
to finish, so the working tree may already be half-changed.

Exactly one such task can exist. When you find it:

1. Print its card, labelled `RESUMING`.
2. Run every command in its `verify` array. All pass → set its status to `done` and continue to
   selection. Any fail → this is the task to work on; report which commands failed and stop.
3. Never hand out a different task while one is `in_progress`. Re-handing an interrupted task out as
   fresh work is how a build ends up applying the same migration twice.

More than one `in_progress` → corrupt bundle. Report both ids; do not pick.

## Select

1. Read every task and its status.
2. A task is **ready** when its status is `pending` **and** every id in its `dependencies` array has
   status `done`. `in_progress` is not ready (see above) and `done` is not ready.
3. Pick the first ready task in array order. **Array order is the build order and already encodes
   priority — do not re-rank.** `priority` is metadata for humans and for the p0/p1/p2 scope-cut
   conversation; it is not a selector.
4. Edge cases, each reported plainly:
   - **All done** → say the build is complete, print the final verify command from the blueprint, stop.
   - **Nothing ready but tasks remain** → print the blocking graph: which task waits on which. Then
     check for a dependency cycle and name the cycle members if one exists.
   - **A dependency id does not exist** → that is a corrupt bundle. Report it; do not route around it.

## Print the task card

Exactly this, nothing padded around it:

```
TASK <id> — <title>
Epic:      <path to epics/*.md>
Status:    <status>   Depends on: <ids, or "nothing">

WHAT
  <the one-sitting scope — read it from this task's block in the epic file;
   tasks.json carries no scope or description field, only the 60-char title>

FILES
  <every entry in the task's files array>

ACCEPTANCE CRITERIA
  1. WHEN <trigger> THE SYSTEM SHALL <observable response>
  2. ...

VERIFY  (run from the target project root — all must exit 0)
  <verify[0]>
  <verify[1]>
  <...one line per entry; print every command in the array>

REMAINING  <n> of <total> tasks
```

`verify` is an **array** of commands, not one command. Print all of them, in order. A card that
shows only the first is how a task gets marked done with its integration test never run.

Then read the epic file and summarize in three lines max the context the builder needs that the task
card does not carry. Then **stop**. Selecting and briefing is the whole job. Do not start building
unless the user says go — a fresh session should get to choose.

## Flags

| Flag | Behavior |
|---|---|
| `--list` | Print the full queue: `id · title · status · blocked-by`. Done tasks collapsed to a count at the top. No card. |
| `--task <id>` | Print that task's card regardless of readiness. If its dependencies are not done, print the warning and list them first. |
| `--start <id>` | Set that task's status to `in_progress` in `tasks.json` with `Edit`, then print its card. Do this before any code is written — a crash mid-task must leave evidence, and `in_progress` is that evidence. Refuse if another task is already `in_progress`. |
| `--done <id>` | Run **every** command in that task's `verify` array with `Bash`, in order, from the target project root. **All exit 0** → set its status to `done` in `tasks.json` with `Edit`, confirm in one line, then print the next ready card. **Any non-zero** → print which command failed and its output, change nothing. |

`--done` never marks a task complete on assertion alone, and never on a partial run. Every command in
the array is the authority; that is the entire point of writing more than one.

## Rules

1. **Read-only except for `--start` and `--done`,** which write nothing but the `status` field, and
   the `in_progress` → `done` transition of a recovered task. No other flag writes to `tasks.json`,
   and nothing here ever writes application code.
2. **Never invent a task, a criterion, or a verify command.** If the bundle is missing one, that is a
   finding — report it and point at `/architect-audit`.
3. **Never re-plan.** The blueprint is the plan. If it is wrong, the fix is `/architect-refresh` for
   stale pins or a new pass of `/architect`, not silent improvisation here.
4. **One task per invocation.** The card exists because agent reliability falls off a cliff with task
   length. Handing over three tasks at once undoes the reason the bundle was split.
5. Detect and use the user's language for the surrounding prose (`knowledge/localization.md` for the
   full boundary, including the Simplified-Chinese specifics). Keep ids, commands, and file paths
   verbatim.
