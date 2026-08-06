---
name: onelo-features
description: Scans a codebase for instrumentable features and inserts onelo-features SDK calls. Use when a developer wants to add feature flagging to their app with the Onelo Features SDK.
disable-model-invocation: true
allowed-tools: Bash Glob Grep Read Edit
---

# onelo-features Instrumentation

Instruments your codebase with the `onelo-features` SDK in phases 0–6 — follow the
checklist below in order.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

Mandatory. Copy the list below into your reply and check items off as you go. Do
NOT scan or instrument before Phase 0 is done.

```
- [ ] 0a · SDK installed? if not → install (references/sdk-setup.md)
- [ ] 0b · Installed version vs LATEST tag — if behind OR unsure, UPDATE first
- [ ] 0c · Smoke-test: project still builds/imports after install/update (native build/import cmd)
- [ ] 1   · Detect the STACK (frontend AND backend) → report the map
- [ ] 1.5 · OFFER the guided walkthrough — always, before scanning
- [ ] 2   · Scan (destinations + triggers + capabilities) — material, not verdict
- [ ] 2.5 · Classify (atom filter)
- [ ] 2.7 · Model: threads across the stack + sub-features + names
- [ ] 3   · Propose → 4 · WAIT for approval
- [ ] 5   · Implement (+ 5b generate registry)
- [ ] 6   · Report (+ wire SDK: references/sdk-setup.md)
```

## Snippets — the code that gets inserted

One file per platform. Open ONLY the one you detected.

<!-- snippet-index:start -->
| Platform | Snippet file |
|---|---|
| Android — Kotlin | [`references/snippets/android.md`](references/snippets/android.md) |
| Electron | [`references/snippets/electron.md`](references/snippets/electron.md) |
| Flutter — Dart | [`references/snippets/flutter.md`](references/snippets/flutter.md) |
| Kotlin | [`references/snippets/kotlin.md`](references/snippets/kotlin.md) |
| React | [`references/snippets/react.md`](references/snippets/react.md) |
| React Native | [`references/snippets/react-native.md`](references/snippets/react-native.md) |
| Swift — iOS | [`references/snippets/swift-ios.md`](references/snippets/swift-ios.md) |
| Swift — macOS | [`references/snippets/swift-macos.md`](references/snippets/swift-macos.md) |
| Web — HTML embed and JavaScript / TypeScript | [`references/snippets/html-embed.md`](references/snippets/html-embed.md) |
| Node.js — backend | [`references/snippets/node.md`](references/snippets/node.md) |
| PHP — backend | [`references/snippets/php.md`](references/snippets/php.md) |
| Python — backend | [`references/snippets/python.md`](references/snippets/python.md) |
<!-- snippet-index:end -->

## Reference files

- JS/TS/React/Next.js patterns: [references/js-patterns.md](references/js-patterns.md)
- Swift macOS patterns: [references/swift-macos-patterns.md](references/swift-macos-patterns.md)
- Swift iOS patterns: [references/swift-ios-patterns.md](references/swift-ios-patterns.md)
- Kotlin/Android patterns: [references/kotlin-patterns.md](references/kotlin-patterns.md)
- Flutter/Dart patterns: [references/flutter-patterns.md](references/flutter-patterns.md)
- Python patterns: [references/python-patterns.md](references/python-patterns.md)
- **Feature modelling — the walkthrough, sub-features, naming, cross-stack threads: [references/feature-modelling.md](references/feature-modelling.md)**
- **Classification rules (Phase 2.5 atom filter): [references/classification-rules.md](references/classification-rules.md)**
- SDK setup (install, per-language init, declare, build hook): [references/sdk-setup.md](references/sdk-setup.md)
- Troubleshooting: [references/troubleshooting.md](references/troubleshooting.md)

---

## Gating philosophy — in one paragraph

Gate whole pieces of functionality the user perceives as "a thing the app can
do" — screens, flows, capabilities — never the atoms they are built from. And
gate a feature's **full surface**: a destination whose trigger stays visible
means users click into nothing and conclude the app is broken. One gate per
presentation layer, in the screen itself, never in its ViewModel.

The reasoning, the failure modes and the per-language nuances live in
[references/feature-modelling.md](references/feature-modelling.md).

---

## Phase 0 — Ensure the SDK is present & current (always first)

Before scanning or instrumenting:

1. **Is the SDK installed?** Detect it per the language signals in Phase 1. If it
   is NOT present, install it first — see
   [references/sdk-setup.md](references/sdk-setup.md) ("Install the SDK").
2. **Is it current?** The snippets this skill inserts may use APIs added in newer
   SDK versions. **Find the latest:** `git ls-remote --tags https://github.com/onelo-tools/<repo>`
   (e.g. `onelo-js`, `onelo-swift`, `onelo-python`) → highest `*-staging` tag;
   compare to the installed version (see sdk-setup.md "Keep the SDK current").
   If installed < latest **OR you can't tell → UPDATE first**.
3. **Smoke-test after ANY install/update.** An SDK upgrade can REMOVE or tighten
   things existing code relies on (not just add APIs) — that's how a removed
   `discovery_key` / blank `feature_environment` upgrade crash-loops a backend.
   Verify the project still builds/imports BEFORE instrumenting, with its native
   command: JS/TS `npm run build` (or `tsc --noEmit`); Swift `swift build`; Kotlin
   `./gradlew assemble`; Flutter `flutter analyze`; Python import the entry
   (`python -c "import main"`). If it FAILS, the update broke
   EXISTING code → fix that first; do not instrument.
4. Only then proceed to Phase 1. Never instrument against an absent, stale, or
   non-building SDK — you'd insert code the installed version can't run.

---

## Keys

Frontend → **publishable** `onelo_pk_live_…` (dashboard → **SDK** tab).
Backend → **secret** `onelo_sk_live_…` (dashboard → **API Keys → Secret keys**),
read from `ONELO_SECRET_KEY`; never hard-coded, never in client code.
A feature threaded across the stack needs BOTH — ask for each up front.

---

## Phase 1 — Detect platforms

Check for these files in the project root (use Bash `ls` or Glob):

```bash
ls package.json next.config.* Package.swift build.gradle build.gradle.kts pubspec.yaml pyproject.toml requirements.txt setup.py Pipfile poetry.lock 2>/dev/null
```

Map findings to platforms and load the corresponding reference files:

| Signal | Platform | Reference to load |
|---|---|---|
| `next.config.*` + `package.json` | Next.js | js-patterns.md |
| `package.json` (+ .tsx/.jsx found) | React | js-patterns.md |
| `package.json` (no .tsx/.jsx) | Node.js | js-patterns.md |
| `Package.swift` / `*.xcodeproj` + AppKit signals (see below) | Swift macOS | swift-macos-patterns.md |
| `Package.swift` / `*.xcodeproj` + UIKit/SwiftUI without AppKit | Swift iOS | swift-ios-patterns.md |
| `build.gradle` or `build.gradle.kts` | Kotlin Android | kotlin-patterns.md |
| `pubspec.yaml` | Flutter | flutter-patterns.md |
| `pyproject.toml` / `requirements.txt` / `setup.py` / `Pipfile` / `poetry.lock` | Python backend | python-patterns.md |

**Swift platform disambiguation** — a language is not a platform. Decide by framework imports:

```bash
# AppKit (macOS desktop) signals:
grep -rln --include="*.swift" -E "import AppKit|NSApplicationDelegate|NSWindowController|NSStatusItem|MenuBarExtra" . 2>/dev/null | head -5
# UIKit (iOS) signals:
grep -rln --include="*.swift" -E "import UIKit|UIApplicationDelegate|UIViewController" . 2>/dev/null | head -5
```

- AppKit hits only → load `swift-macos-patterns.md`
- UIKit hits only (or pure SwiftUI with no AppKit) → load `swift-ios-patterns.md`
- Both (multiplatform target) → load BOTH and run each platform's patterns

Multiple platforms are possible (e.g. Next.js frontend + native mobile). Load all relevant references.

### Report the stack map — out loud, before anything else

A feature ends where it actually ends. If the app has a backend, a feature that
starts in the UI often finishes in a request handler, and gating only the UI
leaves the endpoint open. You cannot reason about that without knowing the shape
of the stack, so state it:

```
Stack detected
  frontend  Swift (iOS)     ./App
  backend   Python FastAPI  ./server
```

If you find a frontend and no backend, say so — it means every feature ends in
the app, which is simpler and worth confirming rather than assuming.

Read the reference file(s) now before proceeding.

---

## Phase 1.5 — Offer the walkthrough (ALWAYS offer; never force)

Before scanning, put this choice to the developer verbatim:

```
Two ways to do this:

[A] Walk me through it  — I ask about your product feature by feature, map each
    one to your code, and we name them together. Best result, ~10 minutes.
[B] Scan and propose    — I read the codebase and hand you a list to correct.
    Faster; I will miss features that have no screen or button.

Which one?
```

**A is the recommended default** — say so. A scan finds code shapes; only the
developer knows which of them their users would call a feature, and which part
of one they intend to sell separately.

If they pick **A**, run [references/feature-modelling.md](references/feature-modelling.md)
and use the scan below as raw material for its step 4.
If they pick **B**, continue straight to Phase 2 — but you still owe them the
modelling pass in Phase 2.7, just without the interview.

---

## Phase 2 — Scan

Run the Grep patterns from each loaded reference file. For every match:

1. Record: `file`, `line`, `symbol`, `lang`
2. Generate `proposed_name` using these rules:
   - Convert PascalCase/camelCase to kebab-case: `ExportButton` → `export-button`
   - Strip trailing suffixes: Fragment, Activity, ViewController, ViewModel, View, Screen, Page, Handler, Controller, Service, Widget
   - Lowercase, max 48 chars, only `[a-z0-9-]`
   - If duplicate proposed_name exists in the list, append `-2`, `-3`, etc.
3. Skip if `features.feature(` already appears within 5 lines of the insertion point in that file
4. Skip paths: `node_modules/`, `.next/`, `dist/`, `build/`, `out/`, `vendor/`, `__pycache__/`, `.build/`, `DerivedData/`, `xcuserdata/`, `Generated/`, `*.test.*`, `*.spec.*`, `*_test.*`, `test/`, `tests/`

Build the full candidate list before Phase 2.5.

### Trigger discovery (after destination scan)

For every destination candidate found above, run the language reference's
"Trigger detection patterns" to find code that navigates to it.
A trigger is anything that opens, presents, pushes, or navigates to
the destination — buttons in toolbars, menu items, links, deep-link
handlers, keyboard shortcut handlers.

For each trigger match, record:
- `file`, `line`, `lang`
- `linked_to`: the destination's `proposed_name` (e.g. `settings-window`)
- `kind`: one of `button`, `link`, `menu-item`, `deep-link`, `cmd-action`

Triggers inherit the `proposed_name` from their destination — never
generate a separate name for them. The whole feature is "settings-window";
its destination is one row in the proposal, its triggers are linked rows.

Match destination ↔ trigger by symbol/route name with substring fuzzing:
a `Navigator.push(SettingsScreen())` links to destination `settings-screen`,
a `<Link href="/settings/account">` links to destination `settings-account`
even if the path encodes hierarchy. Imperfect matches are fine — Phase 3
shows them to the dev for confirmation.

### Capability discovery (third pass, optional per platform)

If the loaded reference file defines a **"Capability detection patterns"**
section, run it. A capability is a user-invoked action with end-to-end logic
that is NOT a screen — export a file, start a recording, run a sync.

For each capability match, record: `file`, `line`, `symbol`, `lang`,
`kind: capability`, and `invoked_from` (the UI element that calls it —
button, menu item, shortcut).

**Anti-noise filter:** a capability candidate normally needs at least one UI
call site. Pure internal functions — init paths, persistence, background sync
nobody triggers — are the "internal processes" that must NOT become features.

**But do not drop a no-UI candidate silently.** Some genuine product features
have no screen and no button: email notifications, webhooks, remove-branding.
If a candidate changes product BEHAVIOUR (a config boolean, a branch on plan or
tier, a per-tenant toggle, a switchable integration) but has no UI trigger, keep
it and mark it `needs-confirmation` — then ask in Phase 2.7. Silently filtering
it out is the expensive outcome, because the developer never learns it was
missed. Plumbing with no behavioural effect still gets dropped.

Naming: name the action, not the widget — `export-recording`, never
`export-button`. The capability gets TWO insertions sharing one feature
name: `isEnabled` guard at the top of the handler, `isVisible` gate on the
invoking UI element (same pattern as destination + trigger).

Reference files without a "Capability detection patterns" section are
skipped — this pass is additive and backward-compatible.

---

## Phase 2.5 — Classify (atom filter)

Evaluate every Phase 2 candidate against the ordered rule set in
[references/classification-rules.md](references/classification-rules.md) —
first match wins. It covers linked triggers, internal surfaces, backend routes,
screen-shaped vs atom-shaped names and paths, per-language overrides, and what
to do with the ones that stay ambiguous.

Drop every `atom-likely` candidate before Phase 3 and report how many you
dropped in the proposal header.

---

## Phase 2.7 — Model: threads, sub-features, names

The scan gave you locations. This turns them into features. Do it whether or not
the developer took the walkthrough — full rules in
[references/feature-modelling.md](references/feature-modelling.md).

**1. Thread each feature across the stack.** Group the candidates that belong to
ONE feature, including the backend end of it:

```
feedback
  ├─ Swift   ReportBugButton.swift:42   (trigger)
  ├─ Swift   FeedbackSheet.swift:18     (destination)
  └─ Python  routes/feedback.py:87      (handler — where it ends)
```

A scan sees an HTTP call, not which endpoint answers it — so **ask** when the
link isn't obvious, and show the thread you believe in for correction.

**2. One feature = ONE name, on every point of the thread.** Onelo keys the
registry by name: `feedback` declared from Swift and `feedback` declared from
Python becomes a single entry tagged with both platforms. Two different names
silently create two features and the tagging never happens.

**3. Ask for sub-features on every paid feature:** *"is there a part of this you
want to sell separately?"* If yes, it takes the parent's name plus a hyphen —
`feedback` → `feedback-bug`. Developers rarely volunteer these, and they are
where the upsell lives.

**4. Never drop a no-UI candidate silently.** Product capabilities like email
notifications, webhooks or remove-branding have no screen and no button. Mark
them `needs-confirmation` and ask, rather than filtering them out.

---

## Phase 3 — Propose

Group the table **by feature**, not by file — one block per feature showing
every point of its thread, sub-features nested under their parent, and anything
still unconfirmed called out:

```
feedback                                    (Swift + Python)
  trigger      ReportBugButton.swift:42
  destination  FeedbackSheet.swift:18
  handler      routes/feedback.py:87
  └ feedback-bug        sub-feature — paid
      handler  routes/feedback.py:104

email-notifications                         needs-confirmation
  no UI trigger found — is this a feature you sell, or plumbing?
```

Then display this grouped table:

```
Found N features (M destinations + K triggers):

  Feature: settings-window
    Destination:
      #1   src/Windows/SettingsWindow.swift:23     SettingsWindow         screen
    Triggers (3):
      #1a  src/Toolbars/MainToolbar.swift:67       SettingsLauncherButton button → settings-window
      #1b  src/Menus/AppMenu.swift:41              openSettings()         menu   → settings-window
      #1c  src/URLHandler.swift:104                case "settings"        link   → settings-window

  Feature: analytics-dashboard
    Destination:
      #2   src/app/analytics/page.tsx              AnalyticsPage          screen
    Triggers (2):
      #2a  src/Sidebar.tsx:88                      AnalyticsLink          link   → analytics-dashboard
      #2b  src/Cmd/CmdPalette.tsx:202              case "analytics"       cmd    → analytics-dashboard

  Internal / dev-only (4) — default: SKIP (say "keep #5" to gate one anyway):
      #5   src/DebugWindowView.swift:29            DebugWindowView        internal
      #6   src/DebugMoodWindowView.swift:262       DebugMoodWindowView    internal

  ⚠ Destinations with NO detected entry points:
      #3   src/OnboardingView.swift:14             onboarding
      → Find how the user reaches this screen and add it:
        "add trigger for onboarding in <file> at line <N>"
      → Or confirm it's reachable only programmatically:
        "confirm no-trigger #3"

Review the list. You can:
  • Rename:    "rename feature #2 to analytics-page"
  • Remove a destination + its triggers:  "remove #2"
  • Remove a trigger only (false positive):  "remove #1c"
  • Add a trigger:  "add trigger for settings-window in src/Cmd.swift at line 12"
  • Keep an internal feature:  "keep #5"
  • Confirm a screen has no UI entry point:  "confirm no-trigger #3"
  • Accept all:  "ok" or "looks good"
```

---

## Phase 4 — Wait for approval

**Do NOT proceed to Phase 5 until the developer explicitly says "ok", "looks good", "go ahead", "yes", or equivalent.**

After each edit command, update the internal list and re-display the updated table. Keep waiting.

---

## Phase 5 — Implement

**Implement a feature THREAD at a time, not a file at a time.** All points of one
feature — trigger, destination and the backend handler where it ends — go in
together, with the SAME feature name. A half-wrapped feature is worse than an
unwrapped one: the UI hides while the endpoint stays open, so the feature is off
for honest users and on for anyone with the URL.

Do not rename between platforms to "make it clearer". Identical names on both
sides are what make Onelo record ONE registry entry tagged with both platforms.

For each approved candidate (in order):

1. Take the snippet for this language from
   the snippet file for this language (table at the top of this skill). It ships inside this plugin,
   baked from `@onelo/snippets` at publish time — the same source that powers the
   in-app SDK page and /docs, so plugin output cannot drift from what the
   developer sees in the dashboard. No network call, no fallback, no env var.

   Sections available: `npm`, `react`, `web`, `swift`, `macos`, `electron`,
   `reactnative`, `android`, `kotlin`, `flutter`, `python`, `node`, `php`.
   A language with no section has no supported snippet — say so and skip it
   rather than adapting another language's code.

2. Use the `usage` block. Replace `$NAME` with the `proposed_name` for this
   candidate.

   For trigger candidates, wrap with `isVisible` instead of the default
   `isEnabled` check — see the reference file's "Trigger snippet" section.

3. Read the target file.

4. Find the insertion line using the rules from the reference file for that language.

5. Insert the snippet, replacing `$NAME` with the `proposed_name`.

6. Write the file with Edit tool — insert only the snippet lines, do not modify anything else.

If a file cannot be found or the insertion point cannot be determined, skip and add to the "errors" list.

### 5b — Generate the feature registry

After all instrumentation is done, write a registry file plus the
generator script for ongoing maintenance. This is what fixes the
"dashboard only shows features whose code path ran" problem — the
registry is declared upfront so every name appears regardless of
runtime traversal.

For each language you instrumented:

1. Write the generator script (path & language as documented in the
   reference file's "Generator script" section). Mark it executable
   (`chmod +x`) for shell scripts. The Dart generator runs via
   `dart run` and doesn't need an executable bit.

2. Invoke the generator once via Bash to produce the initial registry
   file. The script scans the sources you just instrumented and emits
   the registry constant.

3. Add the registry path and the generator location to the Phase 6
   report so the developer knows which files were created and how to
   wire the build hook.

If the developer's project structure deviates from the defaults
(non-standard sources directory, custom output path), pass explicit
arguments to the script — both the shell and Dart generators accept
positional args for sources dir and output file.

---

## Phase 6 — Report

Display the summary, then walk the developer through wiring the SDK in their app.

```
✓ Instrumented N locations:

  src/components/ExportButton.tsx        → export-button
  src/app/analytics/page.tsx             → analytics-dashboard
  ...

⚠ Skipped M locations:
  src/api/reports/route.ts               → could not determine insertion point
```

### Entry-point coverage

```
Coverage: 6/8 destinations have ALL detected entry points gated.

  ⚠ settings-window — 1 of 2 entry points NOT gated:
      src/AppDelegate.swift:1259   NSMenuItem("Settings...")  ← not instrumented
  ⚠ onboarding — no entry points found (confirmed programmatic-only by developer)
```

A destination whose triggers are not all gated is a half-gated feature:
hiding it in the dashboard leaves dead buttons in the app. Never report
success without this section.

## Phase 6.5 — Wire up the SDK

APPLY with the Edit tool, like Phase 5 — "told the developer to add declare()"
isn't done, the call must be IN the file. Insert the init sample from
[references/sdk-setup.md](references/sdk-setup.md), wiring `declare(...)` to
the Phase 5b registry import (not a hand-typed list — goes stale; run 5b
first if skipped). Confirm in the Phase 6 report declare() is in the file.

⚠️ On CLIENT platforms, a gated component read BEFORE `identify()`'s network
resolve settles can flash-then-disappear-then-reappear — looks like a caching
bug, isn't. See "Avoid the first-paint flicker" in each per-language reference
file, and [references/troubleshooting.md](references/troubleshooting.md).

## Troubleshooting

Common first-integration pitfalls (fail-closed default, first-paint flicker, App
Attest hang, Keychain duplicates, WebView crashes, TCC re-prompts, SwiftUI render
loop): [references/troubleshooting.md](references/troubleshooting.md).
