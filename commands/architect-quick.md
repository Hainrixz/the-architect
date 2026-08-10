---
description: Fast-track blueprint — three questions, smart defaults for everything else, still validator-gated. For when you just want it built. / Blueprint express — tres preguntas, defaults inteligentes, con el mismo validador. / 快速蓝图——只问三个问题，其余走智能默认，仍须通过校验器。触发："快速出方案"、"直接开做"、"别问了直接生成"。
argument-hint: "[one-line project description]"
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

# `/architect-quick` — fast-track

You are being invoked as `/architect-quick`. Same machinery as `/architect`, compressed interview.
Read the skill first — it is still the source of truth:

- Plugin install: `${CLAUDE_PLUGIN_ROOT}/skills/architect/SKILL.md`
- Clone install: `./CLAUDE.md` at the repo root — the clone-mode mirror of the skill. Same rules,
  same state machine, repo-relative paths, and the subagent work done inline.

## Mode

| Setting | Value |
|---|---|
| Entry | Greenfield |
| Depth | 3 questions, one message, then defaults |
| Output | **Recommend single file** — quick mode implies a small build — and confirm it, because `${CLAUDE_PLUGIN_ROOT}/questions/phase-4-generate.md` **Step 2 is authoritative on emission mode**. This row is input to your recommendation, not the decision. Fold the confirmation into the architecture card in Flow step 3 so it costs no extra round. Confirmed single file → `./blueprints/<project-slug>-blueprint.md` in the user's cwd; bundle → `./blueprints/<project-slug>/`. |
| Gate | `blueprint-validator` must return **PASS** — speed does not buy an exemption |

## The three questions — ask all three in ONE message

1. **What is it?** One or two sentences.
2. **Who uses it?** Public users, an internal team, other developers — and roughly how many.
3. **Any tech you must use or must avoid?** Existing stack, hosting constraint, language. "No
   preference" is a valid answer and the fastest one.

If `$ARGUMENTS` already answers Q1, say so and ask only Q2 and Q3. Do not re-ask what you were told.

Then stop asking. Everything else is a default you choose and **state out loud**.

## Defaults — decide, announce, do not ask

| Decision | Default |
|---|---|
| Shape | Classify from the three answers using `${CLAUDE_PLUGIN_ROOT}/questions/phase-1-discovery.md`. Ambiguous → pick the closer one and say which and why. |
| Runtime track | The shape file's **Default runtime track**. Deviate only if Q3 forces it. |
| Capabilities | The shape file's **Core capabilities** table, nothing extra. |
| Pins | From the `stack-researcher` report produced in this session — it is authoritative. `${CLAUDE_PLUGIN_ROOT}/knowledge/runtime-tracks/<track>.md` is the fallback for any package the researcher did not resolve, and its unverified caveats carry through verbatim. Never write a pin from memory. |
| Stage | MVP. Testing is smoke-level, observability is logs plus error tracking. |
| Scope | The shape's build order, trimmed to what ships something usable. |

## Flow

1. Ask the three questions. Wait.
2. Classify the shape. Read `${CLAUDE_PLUGIN_ROOT}/knowledge/shapes/<shape>.md` and its default
   runtime track. Check `${CLAUDE_PLUGIN_ROOT}/knowledge/stack-compatibility.md` for known-bad pairs.
3. Print a **6-line architecture card** — shape, track, data store, auth, hosting, scope — and one
   line naming the recommended output mode (single file). Ask for a single yes. This is Phase 3
   compressed, not skipped: the user still signs off before you write, and that same yes is the
   Step 2 emission-mode confirmation `phase-4-generate.md` requires. If they ask for a bundle
   instead, take it and move on.
4. Dispatch `stack-researcher` to verify pins.
5. Dispatch `blueprint-writer` to emit the blueprint from
   `${CLAUDE_PLUGIN_ROOT}/templates/blueprint-template.md`,
   `${CLAUDE_PLUGIN_ROOT}/templates/claude-md-template.md`,
   `${CLAUDE_PLUGIN_ROOT}/templates/tasks-schema.md`, and
   `${CLAUDE_PLUGIN_ROOT}/templates/epic-template.md`. Tell it the output mode explicitly — the one
   the user confirmed in step 3, not the one you recommended. In single-file mode the workspace files (`CLAUDE.md`,
   `AGENTS.md`, `.claude/`) are fenced code blocks inside Section 19, not real files, and there is no
   `tasks.json` and no `epics/`. **`.claude/commands/` is never emitted in either mode.**
6. Dispatch `blueprint-validator`. FAIL → fix and re-run. Do not narrate the failure as a caveat and
   hand it over anyway.

## Rules

1. **Never more than one round of questions.** If something is genuinely undecidable from three
   answers, pick the reversible option and record it in §20.3 Decision log with its
   `Would reverse if` trigger. The blueprint has no Open Questions section by design — an unresolved
   marker left in the deliverable is a validator BLOCKER.
2. **Every build step still gets an observable "Done when"** and a verify command. This is the part
   people try to cut for speed. It is the part that makes the build survive.
3. **State every default you applied** in the summary, so the user knows what they got by omission.
4. Detect and use the user's language. Not English? Apply `knowledge/localization.md` — prose
   follows the user, identifiers/paths/commands/`status` values/EARS keywords stay English.

## After completion

Print the output path and the applied-defaults list. Bundle → the task count and: run
`/architect-next` from the target project root to start. Single file → say resume is manual and hand
over the path. If any default is wrong, `/architect` does the full interview instead.
