---
name: onelo-auth
description: Adds Onelo Auth (hosted sign-in/sign-up) to an app — detects the platform, inserts the official Onelo snippet shipped with this plugin, wires it in, and explains the moving parts. Use when a developer wants to add authentication with Onelo, or asks why their Onelo sign-in isn't working.
allowed-tools: Bash Glob Grep Read Edit Write
---

# Onelo Auth — starter

Get a developer from "nothing" to "a working, branded sign-in" without them
reading any docs. This skill does the wiring; the dashboard does the rest.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 0 · Prerequisites: app exists, publishable key + apiUrl in hand
- [ ] 1 · Detect the platform(s) in this project → report what you found
- [ ] 2 · READ the snippet from references/ (never write one from memory)
- [ ] 3 · Propose where it goes → WAIT for approval
- [ ] 4 · Insert + substitute placeholders
- [ ] 5 · Verify + tell them what to check in the dashboard
- [ ] 6 · Next steps (what this unlocks)
```

---

## What Onelo Auth actually is (read once)

Onelo hosts the whole sign-in surface. You do **not** build a login form, handle
passwords, verify emails, or run OAuth callbacks — you call one method and get a
session back.

The single most important idea: **`loadAuthView()` is a router, not a login
box.** Depending on how the app is configured in the dashboard, the same call
sends the user to sign-in, to the waitlist, or to the store — and it only
resolves with a session once the user is actually *entitled* to use the app.

That is why you gate your UI on **the result of the call**, never on "sign-in
happened":

```
const session = await onelo.loadAuthView()
if (!session) return   // they closed the store/waitlist without completing
// …only now render the paid app
```

A developer who renders their app right after sign-in has a paywall that anyone
can walk past. Say this out loud when you wire it in — it is the mistake that
costs them money.

### Two presentations

- **Embedded** (default) — an in-page iframe modal. Requires the site's origin in
  **Allowed Origins** in the dashboard. On a test key `localhost` already works.
- **Popup** — a top-level window, works from any origin, nothing to configure.
  `loadAuthView({ mode: 'popup' })`.

If the developer is on a framework/origin you can't verify, prefer popup and say
why — a misconfigured origin fails at runtime with a CORS-shaped error that is
annoying to diagnose.

---

## Phase 0 — Prerequisites

Before touching code, confirm the developer has:

1. **An app in the Onelo dashboard.** If not, they create one first — the
   publishable key belongs to an app.
2. **The publishable key** (dashboard → **SDK** tab, shown at the top,
   `onelo_pk_live_…` / `onelo_pk_test_…`).
3. **The API URL** — shown in the same snippet.

Never invent or guess a key. If they don't have one, stop and point at the SDK
tab; everything else here depends on it.

---

## Phase 1 — Detect the platform

Report a map before you touch anything. A repo often holds more than one target
(a web app *and* a Swift client).

```bash
# Web / Node
ls package.json && grep -E '"(next|react|vue|svelte|express|fastify)"' package.json
# Swift (iOS / macOS)
ls Package.swift *.xcodeproj 2>/dev/null
# React Native / Expo
grep -E '"(react-native|expo)"' package.json 2>/dev/null
# Flutter
ls pubspec.yaml 2>/dev/null
# Android
ls build.gradle.kts build.gradle 2>/dev/null
# Python / PHP backends
ls requirements.txt pyproject.toml composer.json 2>/dev/null
```

Map each target to its `lang` value for the snippet API:

| Target | `lang` |
|---|---|
| Web (bundler: Next/React/Vue/…) | `npm` |
| Swift — iOS | `swift` |
| Swift — macOS | `macos` |
| Electron | `electron` |
| React Native | `reactnative` |
| Android (Kotlin) | `android` |
| Flutter | `flutter` |
| Python backend | `python` |
| Node backend | `node` |
| PHP backend | `php` |

**Frontend and backend are different jobs.** The frontend snippet signs the user
in; the backend snippet *verifies* the token on your own API. An app with a
backend usually needs both — ask which they want, don't assume.

---

## Phase 2 — Read the snippet (never write one from memory)

The integration code ships **inside this plugin**, baked from `@onelo/snippets`
at publish time — the same source the dashboard **SDK** tab and **/docs** render
from. No network call, no API key, nothing to fetch.

Open ONLY the file for the platform you detected in Phase 1 — one file per
platform, so the rest costs you nothing.

<!-- snippet-index:start -->
| Platform | Snippet file |
|---|---|
| Android — Kotlin | [`references/snippets/android.md`](references/snippets/android.md) |
| Electron | [`references/snippets/electron.md`](references/snippets/electron.md) |
| Flutter — Dart | [`references/snippets/flutter.md`](references/snippets/flutter.md) |
| React Native | [`references/snippets/react-native.md`](references/snippets/react-native.md) |
| Swift — iOS and macOS | [`references/snippets/swift.md`](references/snippets/swift.md) |
| Web — JavaScript / TypeScript | [`references/snippets/web.md`](references/snippets/web.md) |
| Node.js — backend | [`references/snippets/node.md`](references/snippets/node.md) |
| PHP — backend | [`references/snippets/php.md`](references/snippets/php.md) |
| Python — backend | [`references/snippets/python.md`](references/snippets/python.md) |
<!-- snippet-index:end -->

Frontend and backend are different jobs: the client file signs the user in, the
backend file verifies the token on your own API. An app with a backend usually
needs both — ask which they want.

**Never author an Onelo SDK call yourself**, and never adapt one language's
snippet into another. If a platform has no section here, it has no supported
snippet — say so plainly and stop. An invented SDK call compiles, ships, and
fails in production.

---

## Phase 3 — Propose placement, then WAIT

Show the developer exactly what you intend to do before doing it:

```
Onelo Auth — proposed changes

#1  src/lib/onelo.ts (new)      create the client, module-level singleton
#2  src/app/layout.tsx:14       import it
#3  src/components/Gate.tsx:22  gate the app on the loadAuthView() result

Install: npm install github:onelo-tools/onelo-js
```

Placement rules that matter:

- **One instance, module level.** Not inside a component, hook or request
  handler. Re-creating it drops the session listener and the in-memory state.
- **SSR (Next.js, Remix): create it once at server startup**, never per request.
- The gate goes where paid UI is decided — not in the router, not in a layout
  that renders before the check resolves.

Wait for approval. Commands: `apply`, `skip #N`, `explain #N`, `cancel`.

---

## Phase 4 — Insert

- Substitute `onelo_pk_live_YOUR_KEY` and `https://api.onelo.tools` with the real values.
- Keep the snippet's comments. They carry the routing/gating warnings above and
  the developer will read them later, when you're not there.
- Change nothing else in the files you touch.

---

## Phase 5 — Verify

Tell the developer what to check, in this order — it matches how it fails:

1. **Origin.** Embedded mode from an unlisted origin fails silently-ish. Dashboard
   → app → **Auth → Allowed origins**. On a live key `localhost` is NOT allowed.
2. **The call resolves.** `loadAuthView()` returning `null` is not a bug — it
   means the user closed the store/waitlist. Their gate must handle it.
3. **Session persists across reload** — `onelo.auth.getSession()`.
4. **Sign-out returns to the login UI.** The SDK clears the session but cannot
   re-render their app: they must wire one sign-out handler
   (`onAuthStateChange` / `onSessionChange`).

---

## Phase 6 — What this unlocks

Say this at the end, briefly — it's the natural next step, not an upsell:

- **Selling access?** → the `onelo-store` skill (in-app store + website embed).
- **Letting subscribers manage their plan?** → `onelo-customer-portal`.
- **Pre-launch?** → `onelo-waitlist` — and note `loadAuthView()` already routes
  to it when Waitlist mode is on, with no code change.
- **Look and feel** is entirely in the dashboard → **Branding** (one theme, every
  hosted surface). Don't hand-style the hosted pages.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Modal never opens, console shows an origin/CORS error | origin not in Allowed origins → add it, or use `mode: 'popup'` |
| `loadAuthView()` resolves `null` every time | user isn't entitled — Waitlist mode or paywall is on and they didn't complete it. This is the gate working. |
| Session gone after reload | the client was created inside a component/handler — move it to module level |
| Signed in, but paid UI shows to everyone | gate is on "sign-in happened" instead of on the resolved session |
| Works locally, fails in production | live key + production origin not added; `localhost` never applies to a live key |
