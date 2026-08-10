# Capability: Authentication & Authorization

> Proving who the caller is, and deciding what that caller is allowed to touch — the two jobs that get conflated and then leak data.

Last verified: 2026-07-27

## When a project needs this

- The brief contains "sign up", "log in", "my account", "invite a teammate", "roles", or "admin panel".
- Any row in the database is owned by somebody and must not be visible to everybody.
- A third party calls your endpoints programmatically (that is auth too — API keys or client credentials, not a login form).
- Enterprise words appear: SSO, SAML, SCIM, directory sync → also read `knowledge/capabilities/enterprise-readiness.md`.
- The users are inside mainland China → read `knowledge/capabilities/china-mainland.md` **first**. Identity is phone-first there, email-first signup converts badly, and the cross-surface key is a WeChat `unionid` rather than anything in this file.

Skip it entirely for a brochure site or a public read-only API. Adding auth "just in case" costs you a schema, a mirror-sync problem, and a whole test surface.

## Decision one: session or token

Not a vendor question. Answer it before picking anything.

| | Cookie session | Bearer token |
|---|---|---|
| Carrier | httpOnly + Secure + SameSite cookie, set by the server | `Authorization` header, held by the client |
| Revocation | Immediate — delete the session row | Hard — the token is valid until it expires |
| Use for | Anything with a browser, including the app's own fetches | Native mobile, CLIs, third-party API clients, service-to-service |
| Main risk | CSRF on state-changing requests | Token theft from client storage |

Rules: browsers get cookies, never tokens in `localStorage`. Native and machine clients get a short-lived access token plus a rotating refresh token with reuse detection — if a refresh token is presented twice, kill the whole family. Keep sessions server-referenceable (opaque id, or a JWT whose id you can blocklist) so "log out all devices" and "we fired that employee" are one write, not a wait for expiry.

## Decision two: hosted or self-hosted

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **Clerk** | SaaS that wants login working today | Drop-in UI, orgs/invitations/roles built in, MFA, bot defense, mobile SDKs | Users live in their system; per-MAU pricing; mirror-sync required |
| **Supabase Auth** | Anything already on Supabase | Users land in your own Postgres, so row-level policies can reference them; magic links, social, MFA | Weaker org/team primitives; you build the account UI |
| **Better Auth** | Self-hosted TypeScript with real features | Users and sessions in your database, plugin-based orgs/MFA/passkeys/API keys, no per-user cost | You own deliverability, rate limiting, and the abuse surface |
| **Auth.js** | Provider-heavy OAuth login, minimal extras | Huge provider list, adapters for common ORMs, free | Thin above sign-in — orgs, roles, MFA, and admin flows are yours to build |
| **WorkOS** | Selling to enterprises | SSO, SAML, directory sync, audit logs as a product | Overkill until a buyer demands SSO; priced per connection |
| **Firebase Auth** | Mobile-first, Google ecosystem | Best-in-class phone/anonymous auth, strong native SDKs | Pulls the rest of the app toward Google's data layer |
| **API keys** | B2B and machine callers | Trivial to issue and revoke; no session at all | Not user identity — store hashed, scope them, support rotation |

**Lucia is no longer a library.** Its author stopped shipping it and turned the project into implementation guides for writing session auth yourself. Treat it as reading material, not a dependency. Older guidance points at Auth.js as the replacement; the closer successor for library-shaped self-hosted auth is Better Auth, which covers the same ground Lucia did plus orgs, MFA, and passkeys.

## Recommendation

**Default to Clerk for a multi-tenant SaaS, and to Supabase Auth when the project is already on Supabase.** The work in auth is never the login form; it is the long tail — password reset, email change, account linking, MFA enrollment and recovery, bot signups, session revocation, breached-password checks. Hosted providers ship that tail on day one, and it is worth real money not to build it.

Deviate when:
- **Per-MAU cost breaks the model** (huge free tier, consumer scale) → Better Auth, self-hosted.
- **Data residency or "users cannot leave our database"** is a stated constraint → Better Auth or Supabase Auth.
- **Mobile is the primary surface** and phone/anonymous auth matters → Firebase Auth.
- **The first customer is an enterprise demanding SAML** → WorkOS, or a hosted provider whose SSO tier you have priced.
- **There are no humans**, only services → API keys or client-credentials tokens. No session layer at all.

## The auth-to-database pairing question

Ask it in Phase 2, because it decides whether your `user` table is real or a shadow.

**Provider owns the users** (Clerk, WorkOS, Firebase): your database holds a *mirror* row. Consequences — the primary key is the provider's string id, not your own UUID; signup is not transactional; and a dropped webhook means a foreign key pointing at nothing.

Make it safe with two mechanisms, not one:
1. A webhook that upserts on user created/updated/deleted.
2. **Just-in-time provisioning:** on every authenticated request, if no local row exists for this provider id, create it inside the same transaction as the work. This is the mechanism that actually saves you; the webhook is just the fast path.

**Your database owns the users** (Better Auth, Auth.js, Supabase Auth): real foreign keys, transactional signup, one source of truth, and joins that work. You inherit the security surface in exchange. Supabase Auth is the pragmatic middle — a managed service writing into your own Postgres, so database-level policies can reference the current user directly.

Never let both sides own profile fields. Pick one home for name, avatar, and locale; the other side reads it.

## Organizations, teams, and RBAC

- Roles belong on the **membership**, not the user. A person can be an owner of one workspace and a viewer of another.
- Climb this ladder only as far as the brief requires:
  1. **Role column** (`owner` / `admin` / `member`) checked in one guard. Covers ~90% of products.
  2. **Role → permission map in code** — handlers ask `can(actor, 'invoice.delete')`, never `role === 'admin'`. Adds capability without a schema change.
  3. **Permission rows in the database** — only when customers define their own roles. This is a product feature with a UI, not a refactor.
- One authorization helper, called by every read and write. Scattered `if` checks are how tenants see each other's data.
- Return **404, not 403**, for a resource in another tenant. 403 confirms the id exists.
- The last owner of an organization cannot be removed or demoted. Enforce in the database or the guard, not the UI.

## Social providers

Pick by audience, not by count. Every extra provider is another account-linking edge case.

| Audience | Ship |
|---|---|
| Developer tools | Google + GitHub |
| Consumer | Google + Apple — Apple is required by App Store review if you offer any third-party login on iOS |
| B2B / enterprise | Google + Microsoft, then SSO when a buyer asks |

Decide the collision policy up front: when a social login returns an email that already has a password account, either link automatically **only if the provider marked the email verified**, or force a login-and-confirm. Auto-linking on an unverified email is account takeover.

## Data model additions

| Table | Fields | Notes |
|---|---|---|
| `user` | id, external_auth_id, email, name, avatar_url, timestamps | Mirror when the provider owns identity; source of truth when you do. Unique index on `external_auth_id` |
| `session` | id, user_id, expires_at, ip, user_agent, revoked_at | Self-hosted only. Enables device list and remote sign-out |
| `organization` | id, name, slug, timestamps | Only if multi-tenant. Slug is immutable once public |
| `membership` | user_id, org_id, role, invited_by, accepted_at | Roles live here. Unique on (user_id, org_id) |
| `invitation` | org_id, email, role, token_hash, expires_at, accepted_at | Store the hash, not the token. Single-use and expiring |
| `api_key` | id, org_id, name, key_hash, prefix, scopes, last_used_at, revoked_at | Hash the secret; show it once. `prefix` makes keys identifiable in logs without exposing them |

## Build steps this adds

1. **Provider wiring + protected-route guard** — install, configure callbacks, define the public-route allowlist and protect the rest. *Done when:* **WHEN** an anonymous request hits a protected route **THE SYSTEM SHALL** redirect to sign-in and return to the originally requested URL after a successful login.
2. **User mirror or user table** — schema plus webhook handler plus just-in-time provisioning. *Done when:* deleting the local row and issuing one authenticated request recreates it; replaying the same provider webhook twice leaves exactly one row.
3. **Session accessor** — one server-side function returning the current actor, used everywhere. *Done when:* a repository-wide search finds no handler reading auth cookies or headers directly.
4. **Authorization guard** — a single `authorize(actor, action, resource)` helper wired into every read and write. *Done when:* a test signed in as tenant A gets 404 for a tenant B record across list, detail, update, and delete.
5. **Account flows** — password reset, email change with confirmation, logout everywhere. *Done when:* a reset link works once, is rejected after expiry, and "log out everywhere" invalidates a second browser's session on its next request.
6. **Abuse limits** — rate limit login, signup, reset, and invite acceptance by IP and by account. *Done when:* the eleventh login attempt within a minute from one IP returns 429 and the counter resets after the window.
7. **Roles and invitations** *(multi-tenant only)* — invite, accept, change role, remove member. *Done when:* a `member` receives 403 on an owner-only action and the last owner cannot be demoted.

## Pitfalls

- **Confusing authentication with authorization.** Knowing who someone is says nothing about what they may touch. A logged-in user hitting `/api/invoices/:id` is authenticated on every request and unauthorized on most of them.
- **Trusting the client's copy of the role.** The token or the local state can say `admin`. Recheck server-side on every mutation.
- **Webhook-only user sync.** Webhooks are dropped, delayed, and delivered out of order. Without just-in-time provisioning your first user in production is an orphan.
- **JWTs you cannot revoke.** A stateless token with a long expiry means a compromised account stays compromised until it lapses. Keep expiry short and keep a revocation list.
- **Storing invitation tokens or API keys in plaintext.** A database leak becomes an access leak. Hash on write; compare on read.
- **403 that confirms existence.** Enumeration across tenants. Return 404.
- **Middleware as the only guard.** Route matchers miss cases — a new path, a nested handler, a background job. Middleware is a convenience layer; the real check lives next to the data access.
- **Building enterprise SSO before a buyer asks.** Weeks of work for a hypothesis. Design the schema so it can slot in, then wait for the contract.

## See also

- `knowledge/capabilities/database.md` — where the user mirror, memberships, and tenancy scope columns actually live
- `knowledge/capabilities/api-design.md` — validating and authorizing at the request boundary; API keys for machine callers
- `knowledge/capabilities/enterprise-readiness.md` — SSO, SAML, SCIM, audit logs when a company is the buyer
- `knowledge/capabilities/testing.md` — tenancy-isolation tests, the regression that matters most here
- `knowledge/capabilities/china-mainland.md` — phone-and-SMS identity, WeChat authorization, and the `unionid` data-model trap
- `knowledge/shapes/saas-webapp.md` — the shape that pairs this with billing and multi-tenancy
