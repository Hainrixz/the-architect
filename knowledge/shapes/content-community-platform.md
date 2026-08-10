# Shape: Content & Community Platform

> Content plus identity plus a social graph: publications, membership communities, and course platforms.

Last verified: 2026-07-27

## Is this your project?

**Yes if:**
- Someone logs in and *creates* something — a post, a comment, a course completion, a profile.
- The content model has more than one type (article + author + category, or lesson + module + cohort).
- The user says "like Substack / Circle / Skool / Discourse / a blog with comments / an LMS / a members area".
- Content is gated: free tier reads teasers, paying members read everything.
- Members book a seat in something scheduled — a live class, a cohort call, office hours, a community event — and capacity, waitlists, and timezones follow. Stay here and add `knowledge/capabilities/availability-engine.md`.
- Success is retention and posts-per-week, not conversions or invoices.

**No if:**
- Nobody logs in and nothing is user-generated — it is a brochure with a blog. → `knowledge/shapes/marketing-site.md`
- The core object is a per-tenant workspace with seat-based billing, not a feed. → `knowledge/shapes/saas-webapp.md`
- The primary transaction is buying a product, with cart and fulfillment. → `knowledge/shapes/ecommerce-storefront.md`
- Only employees use it, and the value is operating on internal records. → `knowledge/shapes/internal-tool.md`

The question that settles most ambiguity: **does anyone log in and create something meant to be read by others?** No → marketing site. Yes, but private business data → SaaS. Yes and public → here.

## Default runtime track

**TypeScript / Node** — see `knowledge/runtime-tracks/ts-node.md`. Server-rendered content pages give
SEO and fast first paint; the same framework carries the authenticated surface without a second stack.
Alternatives: `knowledge/runtime-tracks/rails-laravel.md` when editorial CRUD and admin tooling outweigh
frontend interactivity · `knowledge/runtime-tracks/python.md` when recommendations or semantic search are
the differentiator · `knowledge/runtime-tracks/mobile-native.md` only as a companion client.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|
| Auth | Identity is the hinge — profiles, authorship, and gated tiers all hang off it | `knowledge/capabilities/auth.md` |
| Database | Content, social graph, and membership state are relational and query-heavy | `knowledge/capabilities/database.md` |
| Frontend architecture | Two surfaces: cached public reading pages and an authenticated app | `knowledge/capabilities/frontend-architecture.md` |
| API design | Feeds, cursor pagination, comment threads, moderation actions | `knowledge/capabilities/api-design.md` |
| Payments rails | Paid tiers, cohort purchases, annual memberships | `knowledge/capabilities/payments-rails.md` |
| Styling | Reading typography is the product; lock the type scale first | `knowledge/capabilities/styling.md` |
| Deployment | Cached public routes and dynamic member routes on one origin | `knowledge/capabilities/deployment.md` |
| Testing | Permission bugs leak gated content — the regression suite is not optional | `knowledge/capabilities/testing.md` |
| Accessibility | Long-form reading and threaded comments are where a11y failures hurt most | `knowledge/capabilities/accessibility.md` |
| Observability | Feed latency and moderation queue depth must be visible before users complain | `knowledge/capabilities/observability.md` |

Add on demand: `knowledge/capabilities/sync-and-collab.md` (live chat, presence, collaborative drafts) ·
`knowledge/capabilities/ai-llm-integration.md` (semantic search, auto-tagging, first-pass moderation) ·
`knowledge/capabilities/state-management.md` (rich editor with drafts) ·
`knowledge/capabilities/availability-engine.md` **whenever members book a seat in something scheduled** —
live classes, cohort sessions, office hours, community events, 1:1 coaching calls. Capacity, waitlists,
member timezones, and cancellation windows are a different problem from publishing, and a course
platform brief reaches it faster than anyone expects. Model the session as a Post with a schedule and
let that capability own the slot logic; do not invent a parallel calendar.

Also pull in **`knowledge/capabilities/china-mainland.md` when the members are inside mainland China**
— user-generated content there is moderated *before* publish rather than on report, which turns
"create post" from an insert into an asynchronous state machine and changes the data model, not just
the policy.

## Data model

| Entity | Key fields | Relationships |
|---|---|---|
| User | handle, display name, avatar, bio, role | authors Posts, writes Comments, holds Memberships |
| Post | title, slug, body, status, published_at, type | belongs to User, has many Comments, tagged by Topics |
| Topic | slug, name, description | many-to-many with Post |
| Comment | body, parent_id, status | belongs to Post and User; self-referencing for threads |
| Reaction | kind, target_type, target_id | polymorphic over Post and Comment |
| Follow | follower_id, followee_id | User ↔ User, and User ↔ Topic if you have topic feeds |
| Space | name, visibility, required_tier | contains Posts; scopes the feed |
| Membership | tier, status, started_at, current_period_end | User ↔ Plan, mirrored from the billing provider |
| Plan | name, price, interval, entitlements | gates Spaces and Post visibility |
| Notification | kind, actor_id, target, read_at | belongs to a recipient User |
| ModerationAction | action, reason, moderator_id, target | audit trail over Post, Comment, User |

Course variants add `Course → Module → Lesson` plus `Enrollment` and `LessonProgress` — keep progress in its
own table, not a JSON blob, or cohort completion queries become impossible. `status` on Post and Comment is
the workhorse field (`draft | in_review | published | hidden | removed`): every read path filters on it.

## Directory structure

Shown for the TypeScript track; the same boundaries hold elsewhere.

```
src/
  app/
    (public)/     # cached + indexable: home, [topic]/, p/[slug]/ (teaser if tier missing), u/[handle]/, search/
    (member)/     # session required, never CDN-cached: feed/, spaces/[space]/, write/, settings/, billing/
    (admin)/      # moderation/ (report queue, bulk actions), content/ (editorial pipeline)
    api/
  content/        # body renderer, embeds, TOC, share
  social/         # comments, reactions, follows, notifications
  membership/     # tiers, entitlement checks, upgrade prompts
  moderation/     # report intake, queue, actions, audit trail
  feed/           # query builders, ranking, pagination cursors
  lib/            # auth/ db/ search/ email/ + templates: digest, reply, welcome
tests/e2e/        # gating matrix, comment flow, report-and-remove
```

## Build order

1. **Scaffold and design system** — project init, formatting, type scale, color, reading measure, all locked before any page. · *Done when:* the runtime track's lint and typecheck commands exit 0 on a clean tree and a specimen page renders body, headings, code, and blockquote at final scale.
2. **Data model and migrations** — Users, Posts, Topics, Comments, Reactions, Follows. · *Done when:* migrations run forward and back on an empty database; a seed inserts 3 users, 12 posts, 30 comments, and one query returns a post with its threaded comments.
3. **Auth and profiles** — signup, login, session, public profile page. · *Done when:* WHEN a user completes signup THE SYSTEM SHALL create a User row with a unique handle and a profile at `/u/<handle>` reachable while logged out.
4. **Content rendering** — body renderer with headings, code, images, embeds, callouts, auto TOC. · *Done when:* a seeded post renders all six element types server-side and view-source contains the article text with JavaScript disabled.
5. **Editorial workflow** — draft → in_review → published, with author and editor roles. · *Done when:* WHEN a post is in `draft` THE SYSTEM SHALL return 404 on its public URL for everyone but its author and editors; publishing makes it 200 anonymously.
6. **Listings and feeds** — home, topic pages, profile pages, cursor pagination. · *Done when:* paging a 200-post seed by cursor yields every post exactly once, with no duplicate at any page boundary.
7. **Search** — indexed on publish, results page with topic filters. · *Done when:* a post published in the running app is findable by a phrase from its body within one indexing cycle, and a removed post disappears from results.
8. **Comments and reactions** — threaded replies, edit window, optimistic reaction toggles. · *Done when:* a reply nests under its parent after reload; double-clicking a reaction leaves exactly one Reaction row.
9. **Moderation** — reporting, queue, hide/remove/ban, audit trail. · *Done when:* WHEN a moderator removes a comment THE SYSTEM SHALL set its status to `removed`, hide it from all public reads, and write a ModerationAction row naming the moderator and reason.
10. **Memberships and gating** — plans, checkout, webhook-driven entitlement, teaser rendering. · *Done when:* a free account requesting a paid post gets the teaser plus an upgrade prompt with the full body absent from the HTML response — not hidden by CSS — and the same URL returns the full body after a test checkout.
11. **Notifications** — in-app inbox plus email for replies, mentions, and posts in followed spaces. · *Done when:* a reply creates one Notification row for the parent author and zero for the replier; the digest job composes an email carrying a working unsubscribe link.
12. **SEO and syndication** — per-page metadata, article structured data, sitemap, RSS, social images. · *Done when:* the sitemap lists every published post and no drafts, the RSS feed validates, and a post URL produces a correct preview card in a link-preview checker.
13. **Testing and hardening** — E2E for the gating matrix, comment flow, moderation; rate limits on writes. · *Done when:* the suite passes across anonymous/free/paid × public/gated, and posting 20 comments in a minute returns 429.
14. **Deploy and instrument** — production deploy, cache rules, feed-latency and queue-depth dashboards. · *Done when:* a public page serves a CDN cache hit on second request while a member feed request never does, and a production error appears in the dashboard within a minute.

## Pitfalls

- **Gating in the client.** A paywall overlay on a fully-rendered body is a three-second bypass. Gate at the query layer — the server never sends bytes the reader cannot see.
- **Offset pagination.** New posts shift every page, so infinite scroll shows duplicates. Keyset cursors from day one, load-tested against a 10k-post seed.
- **Hard deletes.** Moderation needs an audit trail and an appeal path: flip `status`, keep the row, log who acted, and never `DELETE`.
- **No moderation tooling until launch.** The first spam wave lands the week signups open. Ship the report button and the queue in the same release as public commenting.
- **Notification storms.** A popular post generates thousands. Batch into digests and give every notification type an off switch before shipping the second type.
- **Treating a course platform as a different shape.** Lessons are posts with ordering and progress — reuse the content pipeline instead of building a parallel one.
- **N+1 comment threads and naive text search.** Load a whole thread in one query; pick the search approach in `knowledge/capabilities/database.md` before step 7, not after — database `LIKE` is fine for hundreds of posts and a wall at thousands.

## Skills for the build phase

Install commands and fallbacks: `knowledge/skills-registry.md`. No slash means auto-activating — a slash form
is a silent no-op. If a skill is missing, fall back to the capability file plus `WebSearch` and keep building.

| Skill | When |
|---|---|
| `ui-ux-pro-max` | Step 1 — type scale, reading measure, palette, comment-thread density |
| `frontend-design` | Steps 4-6 — post pages, feeds, profile layouts |
| `emil-design-eng` | Step 8 — reaction and reply micro-interactions, optimistic states |
| `/claude-seo-ai:audit` `/claude-seo-ai:geo` | Step 12 — classic SEO plus answer-engine visibility |
| `/humanizalo` | Seed and launch content, so the platform does not debut sounding synthetic |
| `playwright-cli` | Step 13 — the gating matrix is the suite that matters |
| `/last30days` | Before locking search or moderation tooling choices |

## See also

- `knowledge/runtime-tracks/ts-node.md` — the default track and its pinned versions
- `knowledge/capabilities/auth.md` — roles, sessions, and the entitlement check that gates reads
- `knowledge/capabilities/payments-rails.md` — plans, webhooks, membership state sync
- `knowledge/capabilities/availability-engine.md` — seats, waitlists, timezones, and cancellation windows for live sessions, classes, and events
- `knowledge/shapes/marketing-site.md` — when nobody logs in and nothing is user-generated
- `knowledge/shapes/saas-webapp.md` — when the created object is private workspace data
- `knowledge/stack-compatibility.md` — check the chosen track against search and auth choices
