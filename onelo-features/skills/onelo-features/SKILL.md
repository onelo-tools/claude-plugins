---
name: onelo-features
description: Scans a codebase for instrumentable features and inserts onelo-features SDK calls. Use when a developer wants to add feature flagging to their app with the Onelo Features SDK.
disable-model-invocation: true
allowed-tools: Bash Glob Grep Read Edit
---

# onelo-features Instrumentation

Instruments your codebase with the `onelo-features` SDK. Runs in 7 phases.

## Reference files

- JS/TS/React/Next.js patterns: [references/js-patterns.md](references/js-patterns.md)
- Swift macOS patterns: [references/swift-macos-patterns.md](references/swift-macos-patterns.md)
- Swift iOS patterns: [references/swift-ios-patterns.md](references/swift-ios-patterns.md)
- Kotlin/Android patterns: [references/kotlin-patterns.md](references/kotlin-patterns.md)
- Flutter/Dart patterns: [references/flutter-patterns.md](references/flutter-patterns.md)
- Python patterns: [references/python-patterns.md](references/python-patterns.md)
- Go patterns: [references/go-patterns.md](references/go-patterns.md)

---

## Gating philosophy — gate functionality, not atoms

A feature flag is a contract with the user: "this part of the product is
on/off for you right now." Therefore, gate **whole pieces of functionality**
the user perceives as "a thing the app can do" — screens, tabs, windows,
modals, route handlers, end-to-end flows. **Do not** gate the small UI
building blocks those screens are made of.

If you flag `ResultBanner` separately from `GameBoard`, you can produce
nonsensical states — the banner hidden but the board still running, leaving
a half-finished game with no result UI. Atoms are implementation details of
a screen; the screen is the unit users (and your analytics, and your billing)
actually reason about.

**The opposite failure mode — orphaned triggers — is just as nonsensical.**
If you gate `SettingsWindow` but leave the toolbar button that opens it
visible, users click it and the app silently fails — they conclude the app
is broken. A feature isn't a single component; it's the **destination plus
every entry point that reaches it**. Phase 2 grep patterns include trigger
detection per language; Phase 2.5's new Rule 0 keeps triggers linked to
their destination so a feature's full surface gates as one unit. Triggers
default to `isVisible` (hide only when the feature's status is explicitly
`hidden`) so admin-controlled states like `greyed` (paid upsell) or
`coming_soon` (teaser) keep the trigger reachable without any code change.

The full classification rules — what counts as a screen, what counts as an
atom, what to skip — live in **Phase 2.5** below. Each language's reference
file only adds the *language-specific* signals (e.g. SwiftUI's
`WindowGroup`, Flutter's `Scaffold` rule, Next.js's `app/<route>/page.tsx`
routing convention) on top of those generic rules.

**One gate per presentation layer.** Gate the screen in the screen (its
`body` / `viewDidLoad` / render function) — NOT in its ViewModel. A
ViewModel guard leaves the UI rendered but inert: the user sees the screen,
clicks, and nothing happens — indistinguishable from a bug. ViewModel/handler
guards are reserved for capabilities (action handlers), where the UI trigger
is gated separately with `isVisible`.

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
| `go.mod` / `*.go` files / `package main` | Go backend | go-patterns.md |

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

Read the reference file(s) now before proceeding to Phase 2.

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

**Anti-noise filter (mandatory):** a capability candidate MUST be invoked
from at least one UI element. Pure internal functions — init paths,
persistence, background sync not triggered by the user — are exactly the
"internal processes" that must NOT become features. If you cannot find a
UI call site, drop the candidate.

Naming: name the action, not the widget — `export-recording`, never
`export-button`. The capability gets TWO insertions sharing one feature
name: `isEnabled` guard at the top of the handler, `isVisible` gate on the
invoking UI element (same pattern as destination + trigger).

Reference files without a "Capability detection patterns" section are
skipped — this pass is additive and backward-compatible.

---

## Phase 2.5 — Classify (atom filter)

For every candidate from Phase 2, evaluate the rules below **in order**.
First match wins.

### Rule 0 — Linked trigger → keep (`trigger`)

If the candidate was matched by trigger discovery (Phase 2's trigger pass)
and links to an already-classified `screen` destination, classify as
`trigger` and keep — even if it would otherwise drop under Rule 3
(atom name like `*Button`) or Rule 4 (atom path like `/components/ui/`).

A trigger inherits its destination's feature name. The skill snippet
inserted in Phase 5 will use `isVisible` rather than `isEnabled` so
non-`hidden` statuses keep the trigger reachable.

This rule runs FIRST so that aggressive atom filtering (Rules 3-4)
doesn't drop legitimate triggers.

### Rule 0.5 — Internal/dev surface → `internal` (ask the developer)

If the symbol OR proposed_name contains one of these word-boundary segments
(case-insensitive): `Debug`, `Dev`, `Internal`, `Diagnostic`, `Test`,
`Playground`, `Sandbox` → classify as `internal`. Examples: `DebugWindowView`
→ internal; `debug-pipeline-window` → internal. A product feature that merely
contains the substring without word boundary (`Debugger` as a product) stays
a normal candidate — when unsure, classify `internal` and let Phase 3 ask.

`internal` candidates are NOT dropped and NOT silently kept: they go to a
separate section of the Phase 3 table with default action **skip**. The
developer can keep one with `keep #N` (legit use case: remotely toggling a
debug window). Every kept internal feature is renamed with the `internal-`
prefix (`internal-debug-pipeline`) so the dashboard's slug-prefix grouping
collects them in one group, clearly separated from product features.

### Rule 1 — Backend route handler → `screen` (keep)

If the language reference flagged the candidate as a route handler
(Express/Hono call, Next.js `route.ts`), classify as `screen` and keep.

Backend handlers are inherently the right unit; atom filtering does not
apply to them.

### Rule 2 — Screen-shaped name or path → `screen` (keep)

Symbol ends with one of these (case-insensitive) → keep:

```
Screen   Page    Tab     Window   Sheet    Modal
Wizard   Flow    FlowView         Onboarding
Activity Fragment ViewController
```

OR the file path contains one of these → keep:

```
/screens/    /pages/    /tabs/    /windows/    /flows/    /routes/
/wizards/    /onboarding/
```

### Rule 3 — Atom-shaped name → drop (`atom-likely`)

Symbol ends with one of these (case-insensitive) → drop:

```
Cell      Row       Item      Tile        Chip      Tag
Banner    Indicator Badge     Pill        Toast     Tooltip
Bar       Header    Footer    Divider     Skeleton  Placeholder
Avatar    Spinner   Loader    Icon        Thumbnail
Field     Input     Button    Label       Bubble
```

### Rule 4 — Atom-shaped path → drop (`atom-likely`)

File path contains one of these → drop:

```
/atoms/         /primitives/        /elements/
/components/atoms/    /components/primitives/    /components/ui/
/ui/atoms/      /ui/primitives/     /ui/
/design-system/    /designsystem/   /ds/
/widgets/atoms/    /shadcn/
```

### Rule 5 — Language override

The per-language reference file may add **its own** rules — extra atom
suffixes only meaningful in that ecosystem (e.g. `Tile` in Flutter,
`Style`/`Modifier` in SwiftUI), or extra disambiguators (e.g. Flutter's
"builds a `Scaffold` → screen" rule). Apply those after Rules 1–4.

### Rule 6 — Default → `ambiguous` (keep, flag)

Anything else stays in the proposal table but is marked `?` in the
classification column so the developer can drop it quickly.

### Output

Drop all `atom-likely` candidates from the list before Phase 3. Record
the dropped count and surface it in the proposal table header:

```
Found 12 instrumentable locations (skipped 47 atom-like; see Phase 2.5):
```

The developer can still re-add a dropped candidate manually in Phase 3
with `add feature <name> in <file> at line <N>`.

---

## Phase 3 — Propose

Display this grouped table:

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

For each approved candidate (in order):

1. Fetch the snippet for this language from the Onelo dashboard API. The
   dashboard exposes the same templates that power the in-app SDK page, so
   plugin output stays in lock-step with what developers see when they open
   `app.onelo.tools`. Try the override env var first (used during local
   plugin development), then the production URL:

   ```bash
   ONELO_API_BASE="${ONELO_API_BASE:-https://app.onelo.tools}"
   curl -sf "$ONELO_API_BASE/api/snippets?sdk=features&lang={lang}" 2>/dev/null
   ```

   Where `{lang}` is one of: `npm`, `react`, `script`, `swift`, `kotlin`, `reactnative`, `flutter`, `node`, `python`, `go`.

2. If the fetch succeeds, use the `usage` field from the JSON response as the
   snippet to insert. Replace `$NAME` with the `proposed_name` for this candidate.

   For trigger candidates, fetch the snippet using `lang={lang}-trigger`
   suffix when available; fall back to wrapping the existing snippet with
   `isVisible` instead of the default `isEnabled` check from the reference
   file's "Trigger snippet" section.

3. If the fetch fails (no internet, timeout, dashboard unreachable), fall back
   to the hardcoded snippet from the reference file for this language. The
   fallback may be slightly behind the live dashboard but is always shippable.

4. Read the target file.

5. Find the insertion line using the rules from the reference file for that language.

6. Insert the snippet, replacing `$NAME` with the `proposed_name`.

7. Write the file with Edit tool — insert only the snippet lines, do not modify anything else.

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

### 1. Install the SDK

Onelo SDKs ship with all modules — auth, features, paywall, monitor — in
one package, so a single init covers everything you instrumented.

| Stack | Install |
|---|---|
| JavaScript / TypeScript | `npm install @onelo/js` (staging: `npm install github:onelo-tools/onelo-js#staging`) |
| Swift (iOS / macOS) | Swift Package — `https://github.com/onelo-tools/onelo-swift` |
| Android (Kotlin) | JitPack — `com.github.onelo-tools:onelo-android` |
| Flutter | pub.dev — `onelo` |
| Python (FastAPI / Django / Flask / aiohttp) | `pip install git+https://github.com/onelo-tools/onelo-python.git` (staging: append `@staging`) |
| Go (net/http / Gin / Echo / Fiber / Chi / gRPC) | `go get github.com/onelo-tools/onelo-go` (staging: `@staging`) |

### 2. Initialize once at app startup

The instance is what every inserted `onelo.features.feature(...)` call reaches for.

> **`baseURL` matters.** Copy the URL from the dashboard SDK page
> (`app.onelo.tools` → your app → SDK), which renders the right one for
> production (`https://app.onelo.tools`) or staging
> (`https://st.backend.onelo.tools`). Hardcoding production while you're
> on a staging tenant (or vice versa) silently fails — requests land on
> the wrong backend with no useful error.

#### JavaScript / TypeScript
```ts
import { Onelo } from '@onelo/js'
import { FEATURE_REGISTRY } from './generated/feature-registry'

export const onelo = new Onelo({
  publishableKey: 'pk_live_...',
  baseURL: 'https://app.onelo.tools',
  // featureDefaultStatus: 'enabled',  // optional — see "Dev-mode default" below
})

// Register every instrumented feature upfront — see step 3 below for why.
onelo.features.declare([...FEATURE_REGISTRY])
```

#### Swift (SwiftUI App entry)
```swift
import OneloSwift

#if DEBUG
let onelo = Onelo(
    publishableKey: "pk_live_...",
    callbackScheme: "myapp",
    baseURL: URL(string: "https://app.onelo.tools")!,
    featureDefaultStatus: .enabled  // dev: new gates render until enabled in dashboard
)
#else
let onelo = Onelo(
    publishableKey: "pk_live_...",
    callbackScheme: "myapp",
    baseURL: URL(string: "https://app.onelo.tools")!
)
#endif

// Register every instrumented feature upfront — see step 3 below for why.
onelo.features.declare(FeatureRegistry.all)
```

For AppKit projects without a SwiftUI root, hold the instance on the app
delegate (`AppDelegate.shared.onelo`) and reach for it from views as
`AppDelegate.shared.onelo.features.feature(...)`. Onelo's modules are
not `ObservableObject`, so they don't slot naturally into `@StateObject`
or `@EnvironmentObject` — a singleton-on-delegate pattern is idiomatic
for mixed AppKit/SwiftUI codebases.

#### Kotlin (Android `Application` class)
```kotlin
import com.myapp.generated.FeatureRegistry

class MyApp : Application() {
    lateinit var onelo: Onelo
    override fun onCreate() {
        super.onCreate()
        onelo = Onelo(
            publishableKey = "pk_live_...",
            baseURL = "https://app.onelo.tools",
            // featureDefaultStatus = if (BuildConfig.DEBUG) FeatureStatus.ENABLED else FeatureStatus.HIDDEN,
        )
        // Register every instrumented feature upfront — see step 3 below.
        onelo.features.declare(FeatureRegistry.ALL)
    }
}
```
Reach via `(applicationContext as MyApp).onelo.features.feature("...")`,
or expose through Hilt/Koin if you use DI.

#### Flutter (`main.dart`)
```dart
import 'package:flutter/foundation.dart';
import 'package:onelo/onelo.dart';
import 'generated/feature_registry.dart';

final onelo = Onelo(
  publishableKey: 'pk_live_...',
  baseUrl: 'https://app.onelo.tools',
  // featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
);

void main() {
  // Register every instrumented feature upfront — see step 3 below.
  onelo.features.declare(featureRegistry);
  runApp(MyApp());
}
```
For Provider/Riverpod/BLoC projects, expose the instance through your DI of choice and read it from widgets accordingly.

### 3. Declare known features upfront (already wired above)

The init samples in step 2 each call `features.declare(...)` right
after construction. This step explains *why*, and how the registry
constant they reference is kept current.

**Why declare:** lazy discovery via `feature("...")` only registers a
name once that code path actually runs. For non-trivial apps this
leaves rarely-visited gates — debug screens, admin-only handlers,
monitoring endpoints, error boundaries, A/B variants no user is bucketed
into — **invisible in the dashboard** until someone happens to trigger
them. The dashboard also can't show a complete picture of "what
features exist in this codebase". `declare(...)` batch-pings every
known name on init, so they all appear from the first run regardless
of runtime traversal.

**Where the list comes from:** Phase 5b of this skill writes a generated
registry file (`FeatureRegistry.swift` / `feature-registry.ts` /
`FeatureRegistry.kt` / `feature_registry.dart`) plus a generator script.
The init samples above import the registry from that file — there's
nothing to type by hand.

Triggers (Phase 2.5 Rule 0) appear in the registry under their
destination's feature name — declaring the destination once covers
the whole feature surface.

### 4. Keeping the registry in sync (build hook)

The registry file is generated, not hand-maintained. To keep it current
as new `feature("...")` calls are added, wire the generator into your
build pipeline. Each language ecosystem has its own idiomatic hook —
Xcode Build Phases for Swift, `prebuild` script for npm, Gradle task
for Android, wrapper script for Flutter.

The exact snippet to drop into your build config lives in the language
reference file's "Build hook integration" subsection:

- Swift / Xcode (macOS) → `references/swift-macos-patterns.md`
- Swift / Xcode (iOS) → `references/swift-ios-patterns.md`
- JS / TS / Next.js / Vite → `references/js-patterns.md`
- Kotlin / Android Gradle → `references/kotlin-patterns.md`
- Flutter / Dart → `references/flutter-patterns.md`
- Python / FastAPI / Django / Flask / aiohttp → `references/python-patterns.md`
- Go / net/http / Gin / Echo / Fiber / Chi / gRPC → `references/go-patterns.md`

Wire it once, forget about it — every build refreshes the registry
from current source, so the dashboard list stays in lock-step with what
the codebase actually instruments.

### 5. Identify the user (only if you handle auth yourself)

If you use Auth0, Firebase, or your own backend, call
`onelo.identify(userId)` after your login completes. Without it,
per-user and per-plan targeting can't apply and the SDK logs a one-time
warning.

If you use **Onelo Auth**, identify happens automatically — skip this step.

### 6. Real-time delivery

When you flip a feature in the dashboard and click Deploy, every
connected SDK instance receives the update over a long-lived SSE
connection within milliseconds. There is nothing to poll, refresh, or
cache-bust manually.

### 7. What you'll see in the dashboard

Newly instrumented features appear in **Dashboard → Features → Registry**
on first run, tagged **"JUST ADDED"** for the first 24 hours. Configure
visibility (`active` / `hidden` / `greyed` / `upsell` / `targeted`) per
feature, then click Deploy to push the changes to live clients.

---

## Troubleshooting

Common pitfalls Onelo developers hit during their first integration. If
you (or the developer you're helping) are debugging unexpected behavior,
check these first.

### Fail-closed default is by design

A freshly instrumented `feature("...")` returns the SDK's default status
(usually `hidden` / `disabled`) until you enable it in the dashboard.
This is intentional — it prevents code that ships before the dashboard
is configured from leaking unfinished UI to real users.

If you want new gates to render their content while you build them
locally, pass `featureDefaultStatus: .enabled` (or the language
equivalent) in the SDK init, gated by your debug flag. Each language
reference file shows the exact syntax under "Dev-mode default".

### macOS App Attest hang on dev builds

Older SDK versions (< 3.18.0) could hang on
`DCAppAttestService.attestKey` for Developer ID-signed macOS apps that
ship without an embedded provisioning profile. The SDK now adds a 5-second
timeout and an entitlement check; bump to ≥ 3.18.0 if you see the
symptom (login UI never appears, attestation pegs CPU forever).

### Keychain `errSecDuplicateItem` after rebuilds

Ad-hoc-signed dev builds get a fresh `cdhash` on every `swift build`,
which invalidates the ACL on previously-stored keychain items. Older
SDKs (< 3.17.1) failed `SecItemAdd` because `SecItemDelete` couldn't
reach the existing entry. The SDK now falls back to `SecItemUpdate` on
duplicates; bump to ≥ 3.17.1.

### Hosted-auth WebView opens then disappears

If the WKWebView's content process crashes (sandbox kill, OOM, system
pressure), the window goes blank with no signal back to the user. The
SDK now recovers automatically — silent reload on first crash, error
message on the second; bump to ≥ 3.18.2.

### TCC permission re-prompts on every dev build

Each ad-hoc rebuild generates a new `cdhash`, and macOS treats that as a
distinct app for TCC purposes — Camera, Microphone, Files & Folders,
etc. permissions get re-prompted every launch. Workarounds: use a stable
Developer ID for daily dev builds, or deploy to TestFlight for testers.
This is a macOS / TCC concern, not specific to Onelo.

### `body` re-renders forever after instrumenting a SwiftUI view

Older SDKs (< 3.18.1) made `OneloFeatures` properties trigger view
invalidation when the SDK touched its internal discovery set, producing
an infinite render loop in any view that called `feature(...)` from
`body`. The SDK now marks all internal state as `@ObservationIgnored`;
bump to ≥ 3.18.1.
