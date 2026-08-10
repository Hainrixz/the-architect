# Phase 3: Architecture Confirmation

> Present the architecture, kill every silent assumption, and get explicit approval before anything
> is generated.

Last verified: 2026-07-27

This phase is a conversation, not a questionnaire. But it has **two hard gates**. Both must pass
before Phase 4 runs. Neither is optional.

---

## Order of operations

1. Draft the architecture internally (stack, capabilities, data model, build phases).
2. Check the draft against `knowledge/stack-compatibility.md`. A known-bad combination is a design
   bug, not a risk — fix it now, don't ship it into the Risk Register.
3. **GATE A** — emit `[NEEDS CLARIFICATION]` markers. Resolve all of them.
4. **GATE B** — adversarial pre-mortem. Survivors become Risk Register entries and Non-Goals.
5. Present ONE message. Get approval.
6. Only then: Phase 4.

---

## GATE A — `[NEEDS CLARIFICATION]` markers

Borrowed from spec-driven development. The failure mode it prevents: a blueprint that reads as
complete because the gaps were quietly filled with plausible guesses, and a builder agent that
implements the guess at 2am with no one to ask.

**Scan your draft and emit a marker for every decision that is still underspecified.** Do this
before you write a single line of the presentation message.

### Where the gaps hide

| Area | Marker if you cannot answer it from what the user actually said |
|---|---|
| Scope boundary | Is this feature in v1 or not? |
| Data model | What identifies a record? What happens on delete — cascade, soft, block? |
| Auth & permissions | Who can see whose data? Are there roles? Is there a tenant boundary? |
| Multi-user | Single user, team, or public? Invites? |
| Integrations | Which external service, and who owns the account and the keys? |
| Money | Real payments in v1, or stubbed? Who is the merchant of record? |
| Scale & limits | Rows, requests/sec, file sizes, concurrent users — order of magnitude only |
| Runtime target | Where does this deploy? Any hard hosting or region constraint? |
| Compliance | Personal data? Payments? Health? Minors? Any of these changes the design |
| Design direction | UI projects only — see Q2 below |
| Content ownership | Who writes the copy, seeds the data, supplies the assets? |

### Marker format

```
[NEEDS CLARIFICATION: <the specific question> — blocks <what it blocks>]
```

Concrete, not decorative:

- ✅ `[NEEDS CLARIFICATION: can a user belong to more than one workspace? — blocks the data model and every row-level policy]`
- ✅ `[NEEDS CLARIFICATION: are invoices immutable after send, or editable? — blocks the audit table and the PDF cache]`
- ❌ `[NEEDS CLARIFICATION: more detail on the database]`

### Resolving a marker

A marker is resolved exactly three ways. Pick one per marker, out loud:

| Resolution | Looks like |
|---|---|
| **Answered** | The user tells you. Record the answer in the blueprint. |
| **Defaulted** | You state the default AND the user confirms it: "I'm assuming one workspace per user for v1 — say so if that's wrong." Silence is not confirmation on anything load-bearing. In Chinese, "嗯"／"好的"／"了解" is acknowledgement, not assent — "可以"／"就这样"／"没问题" is. |
| **Deferred** | It becomes an explicit Non-Goal (Section 1). "No multi-workspace in v1" is a legitimate answer. |

Batch related markers into your questions — max 3 questions per message, same as every other phase.
Do not read the raw marker list at the user; translate it into plain questions.

> **Hard rule: entering Phase 4 with any outstanding marker is forbidden.** If the user says "just
> pick something," that is an *Answered* resolution — but write the choice into the blueprint as an
> explicit decision, not as an unstated assumption.

---

## GATE B — Adversarial pre-mortem

Try to kill the plan before you generate it. Cheaper here than in week three of the build.

**If `/abogado-del-diablo` is installed, use it** — hand it the drafted architecture and the project
brief. If it is not installed, run the same eight angles yourself, inline. **The gate is mandatory;
only the tooling is optional.**

| # | Angle | The question that kills |
|---|---|---|
| 1 | False assumptions | Which premise, if wrong, makes the whole thing pointless? |
| 2 | Market | Does anyone actually want this, or does the user just want to build it? |
| 3 | Competition | What already exists and does this for free? Why would anyone switch? |
| 4 | Viability | Does it survive contact with real usage — cost per user, support load, content supply? |
| 5 | Numbers | Do the unit economics and the scale estimates hold at 10x and at 0.1x? |
| 6 | Execution | Can *this* person ship *this* scope? What is the single hardest part and is it in v1? |
| 7 | Pre-mortem | It's six months out and this failed. Write the most likely obituary. |
| 8 | Blind spot | What is nobody in this conversation looking at? Legal, moderation, data loss, abuse, ops. |

Aim the critique at the plan, not the person. You are pressure-testing a design, not discouraging a
builder.

### What comes out

Keep the 3-7 findings that survive your own rebuttal. Discard the rest — a Risk Register nobody reads
is worse than none. Route each survivor:

| Survivor type | Destination |
|---|---|
| Real risk with a mitigation | **Risk Register (Section 20)** — risk, trigger, mitigation, and which build step carries it |
| Real risk with no cheap mitigation | **Risk Register (Section 20)**, marked *accepted*, stated plainly |
| Scope that will sink v1 | **Non-Goals (Section 1)** — cut it, name it, say when it comes back |
| Invalidates the architecture | Go back to step 1. Redesign. Do not paper over it. |

Surface only the top 2-3 in the presentation message. The full register lands in the blueprint.

---

## The presentation message

ONE message. Dense, scannable, **under 40 lines**. Structure:

1. **Stack table** — every layer, your pick, one-line rationale. Opinionated: "here's what I'd
   build," never "here are your options." Do not write version numbers here; Phase 4 verifies and
   pins them.
2. **How it fits together** — 3-5 lines or a small diagram. Request path, data path, background work.
3. **v1 includes** / **v1 explicitly excludes** — the exclusions are the valuable half. This is where
   your Non-Goals land.
4. **Build phases** — rough order with a reason for the order. "Auth first because every table's
   access policy depends on it."
5. **Top risks** — 2-3 lines from Gate B, each with its mitigation.
6. **Open questions** — the plain-language form of any Gate A markers still standing.

---

## Confirmation questions

Max 3. Skip any the user already answered.

### Q1 — Fit
"Does this match what you have in mind? Anything you'd change, add, or cut?"

### Q2 — Design direction *(UI projects only — skip entirely for API, CLI, pipeline, or bot shapes)*
"Any direction on the look? Minimal, bold, dense and utilitarian? Dark mode? A site you'd point at?"

After they answer, use `ui-ux-pro-max` for the visual system — palette, type pairing, spacing scale,
component style — and `emil-design-eng` for motion and interaction decisions. **Both auto-activate.
They are not slash commands; writing them with a slash is a silent no-op.** If neither is installed,
fall back to `knowledge/capabilities/styling.md` and commit to one explicit direction with real hex
values anyway. Never leave the design system for the builder to invent.

If the user names a reference site, analyze it with `agent-browser`, escalating to `browser-harness`
only if the site needs their logged-in session.

### Q3 — Hard constraints *(only if not already covered)*
"Anything you definitely want in the stack — or definitely won't accept?"

---

## Looping

If the changes are significant, adjust and re-present. **Do not loop more than twice.** On the third
pass, stop re-presenting and ask directly: "What's the sticking point?" Misalignment after two rounds
is almost always one unstated constraint, not a stack disagreement.

---

## Exit checklist

Every line must be true before Phase 4:

- [ ] Zero outstanding `[NEEDS CLARIFICATION]` markers
- [ ] Gate B run; survivors routed to Section 20 or Section 1
- [ ] Stack checked against `knowledge/stack-compatibility.md`
- [ ] Shape, runtime track, and capability files all read
- [ ] Design system decided (or explicitly N/A for a headless shape)
- [ ] User said yes

Then state: "Generating your blueprint now." and go to `questions/phase-4-generate.md`.

---

## See also

- `questions/phase-2-branches.md` — the deep-dive that feeds this draft
- `questions/phase-4-generate.md` — the generation procedure this gates
- `knowledge/stack-compatibility.md` — known-bad combinations to catch before presenting
- `templates/blueprint-template.md` — where Sections 1 and 20 live
