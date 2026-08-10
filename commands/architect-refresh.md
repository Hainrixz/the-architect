---
description: Re-verify every pinned version in an existing blueprint against the live registries and report what moved, what breaks, and what to change — un-rots a months-old blueprint without a redesign. / Reverifica cada versión fijada en un blueprint contra los registros y reporta qué cambió y qué romper. / 对照线上仓库重新核验蓝图里每一个锁定版本，报告哪些变了、哪些会破坏、该改成什么。触发："版本还是最新的吗"、"刷新一下依赖版本"。
argument-hint: "<path/to/blueprint.md | path/to/bundle/> [--apply]"
allowed-tools:
  - Read
  - Edit
  - Glob
  - Grep
  - Bash
  - WebFetch
  - WebSearch
  - Task
  - AskUserQuestion
---

# `/architect-refresh` — un-rot a blueprint

You are being invoked as `/architect-refresh`. A blueprint written months ago is still
architecturally sound and numerically wrong. This command fixes the numbers and flags the migrations
they imply. It does **not** re-open the design.

The research contract lives in `${CLAUDE_PLUGIN_ROOT}/agents/stack-researcher.md`. Read it first.

## Resolve the target

`$ARGUMENTS` is a path, optionally followed by `--apply`.

| Argument | Do |
|---|---|
| A `.md` file | Refresh that file. |
| A directory | Refresh `blueprint.md`, plus any pins that leaked into `workspace/CLAUDE.md`, `workspace/.claude/**` or `epics/*.md`. |
| Empty | Glob `./blueprints/*/`, then `./blueprints/*-blueprint.md` for single-file mode. One match → use it. Several → ask. None → say so. |
| `--apply` present | Skip the confirmation step in Phase 3 below and edit directly. |

## Phase 1 — extract

Grep the blueprint for every pinned version: the tech stack table, setup commands, lockfile
snippets, Dockerfile base images, CI runner versions, runtime/SDK minimums. Build the inventory:

`Layer | Package or runtime | Pinned in blueprint | Where it appears (section + line)`

Services without versions — Stripe, Supabase, Cloudflare and friends — are not pins. Skip them
unless the blueprint names a specific API version.

## Phase 2 — verify

Dispatch `stack-researcher` **once**, with the whole inventory. One batched subagent, not one per
package. It resolves each package against its live registry and returns current stable, whether the
gap crosses a major, and any rename or removal in between.

Never answer from memory. A pin you recall is a pin you are about to get wrong.

## Phase 3 — the drift report

Print one table:

`Layer | Pinned | Current | Gap | Breaking? | Action | Risk`

- **Gap** — patch, minor, major, or `unverified` if the registry could not be reached. Say
  `unverified` out loud; a confident guess is worse than an honest gap.
- **Breaking?** — yes only when the changelog says so. Cite what changed: a rename, a removed flag, a
  raised runtime floor.
- **Action** — `bump`, `bump + migrate`, `hold` (with the reason), or `replace` (package is dead).
- **Risk** — what breaks in *this* blueprint if the bump lands, in one clause.

Below the table, list **Migrations required** — for each `bump + migrate`, the concrete edit: the
symbol that was renamed, the config key that moved, the minimum runtime that rose. This is the part
worth reading.

Then ask for a yes before editing, unless `--apply` was passed.

## Phase 4 — apply

On confirmation:

1. Edit each pin in place. Do not restructure sections, do not reword prose, do not touch the build
   order unless a migration changes a step's verify command — then edit that command and say so.
2. Update the blueprint header's `Versions last verified:` line to today's date. If the line is
   absent (pre-v2 blueprint), insert it directly under `Blueprint version:` and say so in the report.
3. Record the refresh as rows in **§20.3 Decision log** — `Decision: bumped <pkg> <old>→<new>` ·
   `Why: registry drift, verified <date>` · `Would reverse if: <the migration proves costly>`. Do not
   append a new top-level section: the template's 20 numbered sections are fixed and downstream
   tooling indexes them by number.
   Update §11 Dependencies in place too, including each bumped row's source URL and checked date —
   a pin whose provenance cells still show the old date is a validator finding.
4. Re-run `blueprint-validator` (`${CLAUDE_PLUGIN_ROOT}/agents/blueprint-validator.md`). A refresh
   that leaves the bundle failing is not finished.

## Rules

1. **Pins live in exactly one layer of the knowledge base.** The `stack-researcher` report from this
   session is the authority; `${CLAUDE_PLUGIN_ROOT}/knowledge/runtime-tracks/` is a cache that drifts
   the day after it is written, and never overrides a live registry check. If the drift you found
   also affects a track file, say so in one line and stop — refreshing the plugin's own tracks is a
   separate, deliberate act, not a side effect of auditing a user's file.
2. **Never bump across a major silently.** A major with no migration note means you did not finish
   the research.
3. **Hold is a legitimate action.** Release-candidate lines and packages whose ecosystem has not
   caught up get `hold` with a reason, not a bump.
4. **A refresh never redesigns.** If the right answer is "this stack choice is wrong now, not just
   old", say that in one line and point at `/architect`. Then stop.
5. Detect and use the user's language. Not English? Apply `knowledge/localization.md` — prose
   follows the user, identifiers/paths/commands/`status` values/EARS keywords stay English.
