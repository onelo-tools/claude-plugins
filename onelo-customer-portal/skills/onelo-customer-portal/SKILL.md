---
name: onelo-customer-portal
description: Adds Onelo's hosted Customer Portal to an app — wires a "Manage subscription" entry point where users cancel, change plan, request a refund and download invoices. Use when a developer wants subscribers to manage their own billing, asks about cancellation or invoices, or reports that opening the portal fails.
allowed-tools: Bash Glob Grep Read Edit Write
---

# Onelo Customer Portal — starter

Onelo hosts the entire portal — cancel, change plan, refunds within the window
you set, payment method, past invoices — branded with your app's colours and
logo from the dashboard. **You build one button.**

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 0 · Prerequisites: Onelo Auth wired, users can sign in
- [ ] 1 · Detect the platform(s) → report what you found
- [ ] 2 · Read the snippet file for that platform
- [ ] 3 · Propose where the entry point goes → WAIT for approval
- [ ] 4 · Insert + substitute placeholders
- [ ] 5 · Verify — including the sign-out path
```

---

## What it is (read once)

The portal is a hosted page opened by the SDK — an iframe overlay on web, a
WebView / WebAuthSession on native. There is no portal UI to build and no
billing endpoint to call.

Three facts shape every integration:

**1. It requires a signed-in user.** Opening it without a session throws
(`not_authenticated` / `OneloError.notAuthenticated`). So the entry point
belongs behind your sign-in, and the call needs a catch that routes back to
sign-in rather than showing a raw error.

**2. It resolves when the user closes it.** Not when they change something. Do
not treat "resolved" as "they upgraded" — plan changes arrive over SSE and your
feature gates update themselves.

**3. It can end the session.** If the user schedules account deletion, or a
refund revokes their access, the portal reports it and the SDK signs them out.
Your app must react by re-rendering to its signed-out state — wire that on your
auth-state listener, not around the portal call.

### The plan-picker shortcut

`open({ intent: 'change_plan' })` opens straight into the plan picker instead of
the management home. Use it when a subscriber taps a locked feature's upgrade
call-to-action — landing them on the plans, not "in front of the door". It falls
back to the management home when a plan change isn't available.

On web you usually don't call this directly: `onelo.openUpgrade()` already
routes a subscriber into the portal's Change-plan and a non-subscriber into the
store.

---

## Phase 1 — Detect the platform

A repo may hold more than one target. Report a map before touching anything.

```bash
ls package.json && grep -E '"(next|react|vue|svelte|electron)"' package.json
ls Package.swift *.xcodeproj 2>/dev/null          # Swift
grep -E '"(react-native|expo)"' package.json 2>/dev/null
ls pubspec.yaml 2>/dev/null                        # Flutter
ls build.gradle.kts build.gradle 2>/dev/null       # Android
```

---

## Phase 2 — Read the snippet (never write one from memory)

One file per platform. Open ONLY the one you detected.

<!-- snippet-index:start -->
| Platform | Snippet file |
|---|---|
| Android — Kotlin | [`references/snippets/android.md`](references/snippets/android.md) |
| Electron | [`references/snippets/electron.md`](references/snippets/electron.md) |
| Flutter — Dart | [`references/snippets/flutter.md`](references/snippets/flutter.md) |
| JavaScript / TypeScript | [`references/snippets/javascript.md`](references/snippets/javascript.md) |
| React Native | [`references/snippets/react-native.md`](references/snippets/react-native.md) |
| Swift — iOS and macOS | [`references/snippets/swift.md`](references/snippets/swift.md) |
<!-- snippet-index:end -->

The portal ships on all six client platforms. There is **no backend snippet** —
it is a user-facing surface only.

---

## Phase 3 — Propose placement, then WAIT

Put the entry point where a user looks for billing: account settings, a profile
menu, a "Subscription" row. Not in a nav bar, not on a landing screen.

```
Onelo Customer Portal — proposed changes

#1  src/settings/AccountSection.tsx:64   add "Manage subscription" button
#2  src/lib/onelo.ts                     (no change — reuse the existing client)
```

Rules that matter:

- **Reuse the existing Onelo client.** If the app already wired Auth, there is
  one instance at module level — do not create a second.
- **Show it only to signed-in users**, and ideally only to those with a plan.
  A free user opening the portal sees an empty management page.

Wait for approval. Commands: `apply`, `skip #N`, `explain #N`, `cancel`.

---

## Phase 4 — Insert

- Substitute the publishable key and API URL with the app's real values.
- Keep the error handling from the snippet — the `not_authenticated` branch is
  the one that actually fires in production.
- Change nothing else.

---

## Phase 5 — Verify

1. **Signed-in user opens the portal** and sees their plan.
2. **Closing it returns to the app** cleanly — no hung overlay, no double close
   button.
3. **Signed-out user** hitting the entry point is routed to sign-in, not shown
   an error.
4. **Sign-out path**: after scheduling account deletion in the portal, the app
   returns to its signed-out state. This is the step most integrations miss.
5. **After a plan change**, gated features update without a reload (SSE) — no
   need to act on the portal's return value.

---

## Phase 6 — What this connects to

- **No sign-in yet?** → the `onelo-auth` skill. The portal requires it.
- **Selling the plans in the first place?** → `onelo-store`.
- **Gating features by plan?** → `onelo-features`, and use `openUpgrade()` on a
  blocked feature rather than opening the portal directly.
- **Branding** — colours and logo come from the dashboard. The portal is
  cross-origin; you cannot style its contents.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Throws immediately, never opens | no session — the user isn't signed in, or the token expired. Route to sign-in. |
| Opens but shows nothing to manage | the user has no plan. Show the store instead (`onelo-store`). |
| Modal hangs open after "Change plan" | old SDK version — update it; the fix ships in current `@onelo/js`. |
| User deleted their account but the app still shows them signed in | the app doesn't react to the auth-state change — wire the signed-out re-render. |
| Portal doesn't match the app's branding | branding is per-app in the dashboard; check the right app is selected. |
| Native: user never comes back to the app | callback scheme / deep link not registered — see the platform snippet's setup notes. |
