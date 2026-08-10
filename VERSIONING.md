# Versioning

The Architect follows [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html). That is easy
to say and meaningless on its own, because SemVer only has meaning relative to a **public API** — and
this repo ships no code, no exported functions, and no importable module. It ships prompts, knowledge
files, and a plugin manifest.

So the public API is declared here, explicitly. Without this document, MAJOR is arbitrary and every
release is a coin flip for the people depending on us.

---

## What breaks, and for whom

Three groups consume this repo, and they break in different ways:

| Consumer | Depends on | Breaks when |
|---|---|---|
| **Plugin users** | The marketplace and plugin names, the command names, the skill's trigger behavior | A name changes and `/plugin install` no longer resolves, or a command they typed yesterday is gone |
| **Clone users** | `CLAUDE.md` at the repo root and every path it references | A knowledge or template path moves and `CLAUDE.md` reads a file that is not there |
| **Downstream tooling and forks** | The blueprint section contract, `tasks.json`, the output layout | A section is renumbered, or a bundle lands somewhere their script does not look |

A change that is invisible to all three is not a breaking change no matter how large the diff.

---

## The public API — declared

These are the surfaces covered by the compatibility promise. Anything not on this list is internal
and may change in a PATCH.

1. **The phase workflow contract.** The state machine and its gates:
   `DISCOVERY → DEEP DIVE → ARCHITECTURE →(user confirms)→ GENERATE`, plus `BROWNFIELD` as an
   alternate entry. The names of the states, the order, and the fact that GENERATE cannot be entered
   without an explicit user confirmation.
2. **The `knowledge/` directory paths.** `knowledge/shapes/`, `knowledge/runtime-tracks/`,
   `knowledge/capabilities/`, `knowledge/skills-registry.md`, `knowledge/stack-compatibility.md` —
   and the filename of every file inside them.
3. **The `questions/` paths.** `phase-1-discovery.md`, `phase-2-branches.md`,
   `phase-3-confirmation.md`, `phase-4-generate.md`.
4. **The `templates/` paths.** `blueprint-template.md`, `claude-md-template.md`, `tasks-schema.md`,
   `epic-template.md`.
5. **The plugin and marketplace names.** Marketplace `alan8918` (upstream: `soyenriquerocha`), plugin `the-architect`. The
   install pair is a user-facing contract:

   ```
   /plugin marketplace add alan8918/the-architect
   /plugin install the-architect@alan8918
   ```

6. **The command names.** `/architect`, `/architect-quick`, `/architect-brownfield`,
   `/architect-next`, `/architect-refresh`, `/architect-audit`.
7. **The blueprint section contract.** Twenty numbered sections, their numbers, and their meanings —
   including that the generated target-project `CLAUDE.md` is Section 19.1. Downstream tooling
   indexes by number, so renumbering is a break even when the content is identical.
8. **The output layout.** `./blueprints/<project-slug>/` for bundle mode (`blueprint.md`,
   `tasks.json`, `epics/`, `workspace/`), `./blueprints/<project-slug>-blueprint.md` for single-file
   mode, both relative to the **user's** working directory. Includes the `tasks.json` schema.

The `agents/` filenames (`stack-researcher`, `blueprint-writer`, `blueprint-validator`) are also
public — a fork or a project skill may dispatch them by name.

---

## The rules

### MAJOR — someone's setup stops working

- Renaming, moving, or deleting **any declared path** above. `knowledge/archetypes/` →
  `knowledge/shapes/` was exactly this, which is why v2.0.0 is a major.
- Changing the **phase model**: adding or removing a state, reordering them, or removing the
  confirmation gate.
- Changing the **blueprint section contract**: renumbering, removing a section, or changing what a
  numbered section means.
- Renaming the **plugin or the marketplace**. Installed users do not migrate automatically.
- Removing a **command**, or changing what an existing command does in a way that surprises someone
  who typed it from muscle memory.
- Changing the **output layout** — a different directory, a different bundle shape, a renamed
  `tasks.json`.
- Removing a blueprint section that was previously mandatory, or making a previously optional section
  mandatory in a way that fails existing validation.

### MINOR — new capability, nothing existing moves

- A new shape in `knowledge/shapes/`.
- A new capability in `knowledge/capabilities/`.
- A new runtime track in `knowledge/runtime-tracks/`.
- A new command in `commands/`, or a new subagent in `agents/`.
- A new **optional** blueprint section, or new optional content inside an existing section.
- New guidance, new pitfalls, new decision-matrix rows, expanded classification signals — anything
  that makes The Architect design better without moving a path or changing a contract.
- A new question in `questions/`, as long as the phase structure is untouched.

### PATCH — behavior is unchanged

- Typos, grammar, formatting.
- Broken link fixes.
- **Version-pin refreshes inside `knowledge/runtime-tracks/`.** See below.
- Rewording that clarifies without changing what The Architect does.
- Corrections to a star count, license, or install command in `knowledge/skills-registry.md`, as long
  as no skill is added or removed.

---

## Refreshing pins is PATCH, and it is expected

`knowledge/runtime-tracks/` is the only place in this repo allowed to contain a version number, and
those numbers are wrong the moment they are written. Every track carries a `Last verified:` line for
exactly that reason.

**Bumping an existing row to whatever the registry currently reports, and updating the
`Last verified:` date, is a PATCH.** It does not move a path, does not change the workflow, does not
change the blueprint contract. Nobody's setup breaks.

This is the maintenance loop the whole 3-axis split exists to make cheap. If refreshing a framework
required a MINOR, maintainers would batch it, batching would delay it, and the knowledge base would
rot — which is the exact failure v2 was built to fix. Keep the friction at zero.

Two consequences worth stating plainly:

- **A track file is a cache, not a source of truth.** It is correct on the day it is written and
  drifts after. At generation time, `stack-researcher`'s live registry check outranks it. A stale
  track file must never override a live check.
- **A track that gains a whole new layer** — a new ORM row, a new deployment target, a new
  recommended library — is a MINOR, because that is new guidance, not a refresh. Bumping the number
  on a row that already exists is a PATCH.

---

## Version fields must stay in sync

Three places carry the version. They must always agree:

| Where | Field |
|---|---|
| `.claude-plugin/plugin.json` | `"version"` |
| `.claude-plugin/marketplace.json` | `plugins[0].version` |
| Git | the `vX.Y.Z` tag |

**Set all three in the same commit, then tag.** A release where `plugin.json` says `2.0.0` and the
tag says `v2.0.1` is unresolvable from the outside — an installed user has no way to know which one
describes the files they actually have.

**Never omit the `version` field.** Claude Code uses it to decide whether an installed plugin is out
of date. With no version, the plugin has no stable identity across commits, and **every commit to
this repo reads as a new version to everyone who installed it** — perpetual update prompts for
changes that never touched the plugin surface. Adding a line to this file would look like a release.

### Release checklist

1. Land the changes.
2. Bump `version` in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — same value.
3. Move the `CHANGELOG.md` entry out of `[Unreleased]` into a dated version heading, and update the
   link block at the bottom.
4. Commit the bump and the changelog together.
5. `git tag -a vX.Y.Z -m "vX.Y.Z"` and push the tag.
6. Cut the GitHub release from the tag, pasting the changelog entry as the body.

---

## Pre-1.0 and pre-releases

There is no pre-1.0 line — v1.0.0 was the first public release. Pre-release identifiers follow SemVer
(`2.1.0-rc.1`) and may appear in a tag, but the plugin manifests only ever carry a released version.

## What is not versioned

Generated blueprints. A blueprint is **your** artifact, produced by whatever version of The Architect
you ran, and it does not track this repo afterward. Bumping The Architect never changes a blueprint
you already have. `/architect-audit` and `/architect-refresh` are how you bring an old one forward on
purpose.

## See also

- [CHANGELOG.md](CHANGELOG.md) — what actually changed in each release
- [CONTRIBUTING.md](CONTRIBUTING.md) — which of these categories your PR falls into
