# Shape: Mobile App

> An app installed on a phone from the App Store or Play Store, shipped on a release train and gated by platform review.

Last verified: 2026-07-27

## Is this your project?

**Yes if:**
- The user says "app" and means an icon on a home screen — not a website you can add to the home screen
- It needs something only a device gives you: camera, GPS, biometrics, health data, step counting, Bluetooth, background location
- Push notifications are a core loop, not a nice-to-have
- It must do something useful with no signal (offline reads, queued writes)
- A health/fitness brief: workouts, sleep, nutrition, activity rings — that is this shape, with the health-data variant below

**No if:**
- It's really a responsive web product they'd *also* like installable → `knowledge/shapes/saas-webapp.md`
- Only the server is being designed and someone else owns the client → `knowledge/shapes/api-backend.md`
- It runs on a desktop OS with a menu bar, tray, or local filesystem access → `knowledge/shapes/desktop-app.md`
- It's a chat/assistant surface where the model *is* the product and the phone is incidental → `knowledge/shapes/agent-app.md`
- The app exists to sell a catalog — variants, stock, checkout, shipping — and the phone is one channel → `knowledge/shapes/ecommerce-storefront.md`
- The screens are a thin client over credit-metered image/video/audio generation, and the queue and asset pipeline are the real build → `knowledge/shapes/generative-media-app.md`

**Composition:** if the app also needs its own server (custom business logic, non-BaaS data, webhooks), design that as `knowledge/shapes/api-backend.md` and treat this shape as the client. Do not smear backend design into the mobile build order.

## Default runtime track

**mobile-native** — see `knowledge/runtime-tracks/mobile-native.md`. One codebase, both stores, and the managed build/OTA pipeline removes the two worst chores (signing, release plumbing).

Alternatives:
- **Fully native**, per the same track file, when the app is a thin shell over heavy platform APIs (deep HealthKit/Health Connect integration, CarPlay, widgets, watch app) or when a single platform ships first.
- `knowledge/runtime-tracks/ts-node.md` for the companion backend when you compose with `api-backend`.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|
| Auth | Sessions must survive app kill; social/biometric sign-in expected | `knowledge/capabilities/auth.md` |
| State management | Server cache vs. UI state split decides how offline feels | `knowledge/capabilities/state-management.md` |
| Sync & collab | Offline queue, conflict rules, background refresh | `knowledge/capabilities/sync-and-collab.md` |
| Styling | Design tokens must survive dark mode, safe areas, dynamic type | `knowledge/capabilities/styling.md` |
| Deployment | Build profiles, release channels, store submission | `knowledge/capabilities/deployment.md` |
| Observability | You cannot SSH into a user's phone — crash + event telemetry is the only debugger | `knowledge/capabilities/observability.md` |
| Testing | Device-matrix E2E; unit tests for logic that can't be tapped | `knowledge/capabilities/testing.md` |
| Accessibility | Screen readers and dynamic type are review-gated on both stores | `knowledge/capabilities/accessibility.md` |
| Payments rails | Digital goods must use store IAP; physical goods must not | `knowledge/capabilities/payments-rails.md` |
| Database | Only when composing with a backend of your own | `knowledge/capabilities/database.md` |
| Mainland China operation | No Play Store and no FCM — per-vendor stores and per-vendor push channels, each with its own review | `knowledge/capabilities/china-mainland.md` |

## Data model

| Entity | Notes |
|---|---|
| `user` | Server identity. Never the source of truth for local UI state. |
| `device` | One row per install: push token, platform, app version, locale, timezone. Push targeting reads this. |
| `session` | Refresh token in the OS secure enclave/keystore — never in plain local storage. |
| `<core entity>` | The thing the app is about. Needs `updated_at` and a client-generated ID so offline creates don't collide. |
| `outbox` | Local-only. Queued mutations awaiting connectivity, with attempt count and last error. |
| `notification_pref` | Per-category opt-in. One global toggle is not enough for review or for retention. |
| `entitlement` | If paid: product ID, store, expiry, original transaction ID. Server-verified, never trusted from the client. |

**Health/fitness variant adds:** `health_permission` (per data type, granted/denied/undetermined), `workout` (source: manual vs. platform-imported, with an external ID for dedupe), `daily_metric` (steps, sleep, HR — timestamped, one row per source per day). Never overwrite platform-sourced samples with your own aggregates.

## Directory structure

Shown for the cross-platform file-routed track; the native track's layout lives in the runtime track file.

```
app/                      # File-based routes = navigation
  (tabs)/                 # Main tab group + tab bar layout
  (auth)/                 # Sign-in flow, no tab bar
  (modals)/               # Detail + settings sheets
  _layout.tsx             # Root: providers, session gate, deep-link handling
components/
  ui/                     # Primitives: Button, Input, Sheet, EmptyState
  screens/                # Screen-local composition
lib/
  api/                    # Typed client for the backend
  storage/                # Secure store (tokens) vs. plain store (cache)
  sync/                   # Outbox, retry, conflict resolution
  push/                   # Token registration, category handlers
  health/                 # Platform health adapter (variant only)
hooks/                    # useSession, useNetworkState, use<Feature>
constants/                # Color, spacing, type tokens
assets/                   # Icons, splash, fonts
```

## Build order

1. **Scaffold and boot both simulators** — project init, design tokens (color, type scale, spacing, elevation), dark mode, safe-area handling. *Done when:* the build command produces a runnable iOS and Android artifact and both launch to the root screen in a headless simulator/emulator in CI, exiting 0; a snapshot test renders every shipped screen at the minimum and maximum supported dynamic-type scale and asserts zero truncated text nodes and zero overlapping layout rects; a test asserts the token set resolves to different values under light and dark appearance without a remount.
2. **Navigation skeleton** — tab group, auth group, modal group, 404 route. *Done when:* every planned screen is reachable by tapping from cold start, with placeholder content and no crash on back-navigation from any depth.
3. **Auth** — sign-in, sign-up, sign-out, secure token storage. *Done when:* signing in, force-quitting, and reopening lands on the authenticated home screen with no re-login; signing out clears the secure store (verified by re-launch).
4. **Core read path** — the primary list → detail flow against real data. *Done when:* the list renders live server data, pull-to-refresh refetches, and detail deep-links by ID.
5. **Core write path** — create/edit the main entity with optimistic UI. *Done when:* WHEN a write is submitted in airplane mode THE SYSTEM SHALL show it immediately in the list, persist it to the outbox, and flush it on reconnect without duplicating.
6. **Offline and network state** — cache persistence, offline banner, retry policy. *Done when:* killing the app offline and relaunching offline still shows the last-synced list; the banner appears within 2s of losing connectivity.
7. **Permissions flow** — camera/location/notifications/health requested in context, with a denied state. *Done when:* each permission has a pre-prompt explaining why, and a denied path that still leaves the app usable and offers a deep link to OS settings.
8. **Push notifications** — token registration, per-category prefs, tap-to-route. *Done when:* an integration test asserts that on permission grant the device token is persisted to that user's `device` row and that re-registration replaces it rather than duplicating it; a test against a stubbed push provider asserts the outbound send carries the correct category, deep-link target, and per-category opt-in check, and sends nothing for an opted-out category; and a unit test feeds a synthesized notification object of every category into the tap-to-route handler for all three app states — cold start, background, foregrounded — and asserts each resolves to that category's screen. Delivery through the real push service to hardware is a launch-checklist item, not a build gate.
9. **Deep links and universal links** — scheme + verified domain links. *Done when:* a script drives the simulator/emulator URL-open command once per entry in the route table and asserts the resolved screen and parsed params for each; a test asserts the platform association files are generated with the correct app identifier and signing fingerprint and are served at their required paths with the required content type; and a test asserts the web fallback route returns 200 for every deep-linkable path. Tapping a real link on real hardware, and live domain verification, are launch-checklist items.
10. **Health integration** *(variant only)* — platform health adapter, background delivery. *Done when:* a test injects a synthetic workout sample through the platform health API in the simulator/emulator, triggers a background refresh, and asserts exactly one `workout` row exists with the source marked platform-imported; re-injecting the identical sample and refreshing again asserts the row count is unchanged, deduped by external ID.
11. **Payments** *(if paid)* — store IAP products, restore purchases, server-side receipt validation. *Done when:* a sandbox purchase grants the entitlement, "Restore Purchases" re-grants it on a fresh install, and a tampered client receipt is rejected server-side.
12. **Telemetry and crash reporting** — release-tagged crash reporting, core funnel events. *Done when:* a test throws a deliberate crash from a release-configured build, then queries the crash reporter's **API** by event ID and asserts the event is retrievable, its release tag equals the build's version and build number, and the top frame of its stack resolves to a named symbol in project source rather than a bare hex address — that symbol resolution is what "readable" means here; a second test asserts each core funnel event fires once per trigger with its declared property set. Reading any of it in the vendor's dashboard UI is a launch-checklist item.
13. **Store-ready packaging** — icons at every required size, generated screenshots, privacy manifest / data-safety declaration, review notes with a demo account, all committed as files. *Done when:* the release command emits a signed iOS archive and a signed Android bundle, the platforms' own static validators (`altool`/`bundletool` equivalents, per `knowledge/runtime-tracks/mobile-native.md`) exit 0 on both in CI, and a manifest-lint script asserts every required metadata field — icons, screenshots, privacy strings, data-safety answers, demo credentials — is present and non-empty. Submission itself is a launch-checklist item, not a build gate.
14. **Release train setup** — build profiles (dev/preview/production), OTA channel per profile. *Done when:* an OTA update pushed to the preview channel reaches a preview build within one app restart, and production builds are unaffected.

## Post-build launch checklist

Not build steps — every item here waits on hardware you must hold or a queue you do not control, so none of them can gate an autonomous build. Write them into the blueprint as a launch checklist, assign each an owner, and start the slow ones early.

| Item | Why it cannot be a build gate | Start it |
|---|---|---|
| Install on one physical iOS and one physical Android device and walk the core flow | Simulators lie about push delivery, biometrics, camera, and health data | As soon as step 8 lands |
| Send a real push through the production push service to both physical devices and confirm tap-to-route from cold start, background, and foregrounded | Only the live path exercises certificates, entitlements, and carrier delivery — none of which exist in a simulator | With step 8, on the same hardware |
| Tap a universal/app link in Mail and Messages on one device with the app installed and one without | OS link handling and the without-app fallback are decided by the phone and the live domain, not by your code | With step 9, after domain verification is live |
| Log a real workout in the OS health app on hardware and confirm it lands in yours *(health variant)* | Real sensor sources, background delivery budgets, and the OS permission sheet only exist on device | With step 10 |
| Confirm the release's symbolicated crash is visible in the crash reporter's dashboard with symbols/mapping uploaded | Symbol upload and dashboard ingestion run on the vendor's infrastructure, on their schedule | With step 12 |
| TestFlight + Play internal/external testing distribution | Apple and Google decide when a build reaches testers | Immediately after step 13 |
| Store review submission and approval | An external review queue, days to weeks | Never schedule marketing against an unapproved build |
| Apple Developer / Play Console enrollment and identity verification | Procurement and identity checks, not code | Day one of the project |
| Deep-link domain verification live on the production domain | Requires DNS and the real hostname | With step 9 |

## Pitfalls

- **Treating OTA as a release valve for everything.** JS and assets ship over the air; anything touching native code — a new SDK, a new permission, a new capability, an icon change — requires a store build. Decide *per change* and write the rule into the project's CLAUDE.md.
- **Asking for permissions on first launch.** Cold-prompting for notifications or location tanks accept rates and reads as spammy to reviewers. Ask at the moment of value, and always after your own explainer screen.
- **Charging for digital goods outside store IAP.** Apple and Google both reject this and it is the single most common rejection in this shape. Physical goods and real-world services are the opposite — those must *not* use IAP.
- **Retrofitting deep links.** Universal/app links require domain verification files, entitlements, and a route table. Doing it at step 9 is fine; doing it after launch means broken shared URLs in the wild.
- **Assuming iOS and Android agree.** Safe areas, keyboard avoidance, back gesture, notification channels, and permission semantics all differ. Test both on hardware — simulators lie about push, biometrics, camera, and health data.
- **Shipping health data without the privacy paperwork.** Both stores require declaring health data collection; you may not use it for advertising, and misdeclaring is a rejection *and* a policy strike.
- **Blocking the first screen on the network.** A cold start that shows a spinner until the API responds feels broken on transit Wi-Fi. Render cached content first, revalidate behind it.
- **Version-drifted native modules.** Every added native dependency constrains the SDK upgrade path. Check support before installing, not after — see `knowledge/stack-compatibility.md`.

## Skills for the build phase

Install commands and fallbacks: `knowledge/skills-registry.md`. No leading slash means the skill auto-activates; writing it with a slash is a no-op.

| Skill | When |
|---|---|
| `ui-ux-pro-max` | Design tokens, palette, type scale, component style |
| `emil-design-eng` | Screen transitions, gestures, haptics, micro-interactions |
| `frontend-design` | Building the actual screens, build phase only |
| `/last30days` | Current opinion on store policy shifts and SDK changes before you commit |
| `find-skills` | Discover device/testing skills before starting |

## See also

- `knowledge/runtime-tracks/mobile-native.md` — the pins, scaffolding commands, and per-platform gotchas
- `knowledge/capabilities/sync-and-collab.md` — offline outbox and conflict resolution in depth
- `knowledge/capabilities/payments-rails.md` — store IAP vs. external payment rails
- `knowledge/capabilities/observability.md` — crash reporting and release tagging for shipped binaries
- `knowledge/shapes/api-backend.md` — compose with this when the app needs its own server
- `knowledge/shapes/saas-webapp.md` — when a responsive web app is the real ask
- `knowledge/shapes/desktop-app.md` — the same install-and-signing problems on a desktop OS, with a different store story
- `knowledge/shapes/generative-media-app.md` — when the app is a client over a credit-metered generation backend
- `knowledge/shapes/ecommerce-storefront.md` — when the app is a store and the catalog is the domain
- `knowledge/stack-compatibility.md` — native module and SDK combinations known to break
