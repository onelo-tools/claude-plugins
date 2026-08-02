---
name: onelo-feedback
description: Adds Onelo Feedback to an app — wires a "Report a bug" or "Send feedback" entry point that opens Onelo's hosted form, with the reporter's identity and active features attached automatically. Use when a developer wants in-app bug reports or feature requests, or asks where feedback from their users should go.
allowed-tools: Bash Glob Grep Read Edit Write
---

# Onelo Feedback — starter

Onelo hosts the report form. You wire an entry point and pass context; the
report lands in your dashboard with everything you need to act on it.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 0 · Prerequisites: app exists, Onelo client already created
- [ ] 1 · Detect the platform(s) → report what you found
- [ ] 2 · Read the snippet file for that platform
- [ ] 3 · Decide anonymous vs identified, and where the entry points go → WAIT
- [ ] 4 · Insert + substitute placeholders
- [ ] 5 · Verify — the report arrives WITH its context
```

---

## What it is (read once)

One call opens a hosted form in a modal:

```
await onelo.feedback.open()
```

Three optional pieces of context change how useful the report is:

| Option | What it does |
|---|---|
| `type` | `'bug'` · `'feature_request'` · `'general'` — pre-selects the form so the user doesn't classify their own report |
| `area` | free-form label for **where** they were (`'checkout'`, `'settings'`) — this is what makes reports triageable |
| `userId` | the signed-in user, so you know who reported it |

**The active features from the session are attached automatically.** You don't
pass them — the SDK reads them from the feature state and sends them along, so a
report arrives knowing which flags that user actually had on. That is usually
the difference between "cannot reproduce" and a one-minute fix.

`userId` travels as a header, never a query parameter, so it never lands in an
access log. Keep it that way — don't invent a URL-based variant.

---

## Phase 1 — Detect the platform

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

There is **no backend snippet** — feedback is a user-facing surface only.

---

## Phase 3 — Anonymous or identified? Then placement.

**Anonymous** (`open()` with no `userId`) — for public-facing surfaces: a
marketing site, a signed-out screen, a public docs page. The report isn't tied
to a person.

**Identified** (`userId: currentUser.id`) — inside the app, behind sign-in. Pass
it. A bug report you can't trace to an account is a bug report you often can't
resolve.

Placement that works:

- a **"Report a bug"** item in the help or account menu → `type: 'bug'`
- a **"Send feedback"** item next to it → `type: 'general'`
- an **empty state or error screen** → `type: 'bug'` plus the `area` of that
  screen. This is the highest-signal placement in most apps.

Always set `area` from the screen you're on. Reports without it pile up as
"something is broken somewhere".

```
Onelo Feedback — proposed changes

#1  src/components/HelpMenu.tsx:38    "Report a bug"  → open({type:'bug', area:'help', userId})
#2  src/screens/CheckoutError.tsx:71  "Report this"   → open({type:'bug', area:'checkout', userId})
```

Wait for approval. Commands: `apply`, `skip #N`, `explain #N`, `cancel`.

---

## Phase 4 — Insert

- Reuse the existing Onelo client — one instance at module level. Don't create a
  second one just for feedback.
- Substitute the publishable key and API URL with real values.
- Pass `userId` only where a user is actually signed in; passing `undefined` is
  correct on public surfaces.
- Change nothing else.

---

## Phase 5 — Verify

1. The modal opens and closes cleanly — one ✕, no stacked buttons.
2. A submitted report **appears in the dashboard**.
3. It carries the `type` and `area` you passed, and the reporter when identified.
4. The active features are listed on the report — this is the part worth
   checking, because it is what you'll rely on when triaging.

---

## Phase 6 — What this connects to

- **Turning reports into public commitments?** → `onelo-roadmap`. Items you mark
  Public show on a hosted page you can link from your changelog.
- **Knowing which flags a reporter had?** → `onelo-features`; the feature state
  is what gets attached to reports.
- **Errors nobody reports** → `onelo-monitor`. Feedback catches what users
  notice; Monitor catches what they don't.
- **Branding** — the form's colours and logo come from the dashboard.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Modal opens then immediately closes | two Onelo clients, or the entry point re-renders and remounts — create the client once at module level. |
| Reports arrive with no reporter | `userId` not passed, or passed on a screen where the user isn't signed in yet. |
| Reports are impossible to triage | no `area` — add it per screen. |
| Active features missing from the report | the feature state hasn't loaded yet; on web call `identify()` (own auth) or sign in first. |
| Nothing opens, network shows a non-200 on initiate | wrong publishable key or API URL. |
