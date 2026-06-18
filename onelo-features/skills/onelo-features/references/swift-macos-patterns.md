# Swift macOS Patterns

## macOS-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Swift on macOS adds these signals on top:

- **`*ViewController` → screen.** UIKit-style screens (rare on macOS —
  Catalyst). Always keep.
- **`*WindowController` subclass of `NSWindowController` → screen.** The
  AppKit way to own a window. Always keep.
- **Symbol passed to `NSHostingView(rootView: X(...))` anywhere → X is a
  screen.** Keep X even if its name is not screen-shaped. This is THE
  macOS signal: AppKit apps host SwiftUI content in windows this way, so
  a plain `FooView` hosted in an `NSHostingView` is a real destination.
- **`MenuBarExtra` / `NSPopover` content view → screen.** Menu-bar apps
  present their main UI this way; the content view is a destination.
- **`*ViewModel` is NOT a destination if its View was already found.**
  One gate per presentation layer (SKILL.md Gating philosophy): the gate
  goes in the View's `body`, never in the ViewModel. Never insert a
  screen-gate into a ViewModel — a ViewModel guard leaves the UI rendered
  but inert. ViewModel guards are reserved for capability handlers.
- **Plain `*View` is ambiguous.** SwiftUI overuses `View`. Disambiguate
  by path (`Screens/`, `Tabs/`, `Windows/`, `Flows/`,
  `Features/<X>/<X>View.swift` → screen) or by the `NSHostingView` signal
  above; otherwise mark `ambiguous` for Phase 3.

### Extra Swift-only atom suffixes to drop

On top of the generic Phase 2.5 list:
`Style`, `Modifier`, `Shape`, `Path` — these are SwiftUI extension
points (`ButtonStyle`, `ViewModifier`), never user-visible features.

## Grep patterns (destinations)

```bash
# SwiftUI Views
grep -rn --include="*.swift" -E "^struct ([A-Z][A-Za-z0-9]+): (some )?View" . 2>/dev/null
# AppKit window controllers
grep -rn --include="*.swift" -E "^(final )?class ([A-Z][A-Za-z0-9]+WindowController)" . 2>/dev/null
# UIKit-style VCs hosted on macOS (rare, Catalyst)
grep -rn --include="*.swift" -E "^class ([A-Z][A-Za-z0-9]+ViewController)" . 2>/dev/null
# Views hosted in AppKit windows (screen signal even without screen-shaped name)
grep -rn --include="*.swift" -E "NSHostingView\(rootView: ?[A-Z][A-Za-z0-9]+\(" . 2>/dev/null
# SwiftUI macOS scenes
grep -rn --include="*.swift" -E "(WindowGroup|Window\(|MenuBarExtra)\(" . 2>/dev/null
```

**Skip:** `Tests/`, `*Tests.swift`, `*Spec.swift`, `Preview*.swift`, `Generated/`

## Trigger detection patterns

After destinations are identified, scan for code that opens or presents
them. Triggers inherit the destination's feature name and get an
`isVisible` gate (not `isEnabled`).

```bash
# AppKit menu items (target/action)
grep -rn --include="*.swift" -E "NSMenuItem\((title|image)?:?.*action: ?#selector" . 2>/dev/null
# menuItem SDK helper (already-gated triggers — record as gated)
grep -rn --include="*.swift" -E "\.menuItem\(" . 2>/dev/null
# (no `title:` in the regex — multi-line calls put `title:` on the next line)
# Window presentation
grep -rn --include="*.swift" -E "(makeKeyAndOrderFront|showWindow|orderFront)" . 2>/dev/null
# SwiftUI openWindow
grep -rn --include="*.swift" -E "(openWindow\(id:|@Environment\(\\\\.openWindow\))" . 2>/dev/null
# Status bar / toolbar
grep -rn --include="*.swift" -E "(NSStatusItem|NSToolbarItem|statusItem\()" . 2>/dev/null
# Sheets & popovers (SwiftUI on macOS)
grep -rn --include="*.swift" -E "\.(sheet|popover)\(" . 2>/dev/null
# Notification-decoupled triggers (Button posts a notification, AppDelegate
# observes it and opens the window — common SwiftUI↔AppKit bridge idiom)
grep -rn --include="*.swift" -E "NotificationCenter\.default\.post\(name:" . 2>/dev/null
```

**Notification-decoupled linking (three-hop):** for each `post(name: .someRequest)`
hit inside a Button/menu action, find the matching observer
(`addObserver(... name: .someRequest` or `.onReceive(... .someRequest)`), then
resolve the observer's handler body to a destination symbol the same way as the
selector heuristic below. The Button that posts is the trigger; it inherits the
destination's feature name.

**Already-gated triggers:** any hit on the `.menuItem(` pattern is a
menu item built through the Onelo SDK helper — it is ALREADY gated by the
feature it was created from. Record it as `gated` in the trigger inventory
(it counts toward Phase 6 coverage) and do NOT re-propose an insertion
for it.

### Two-hop linking heuristic

An AppKit menu item does not name the window it opens —
`NSMenuItem(action: #selector(openSettings))` only names a selector. Link
the trigger to its destination in two hops:

1. **Selector → function.** Find `func openSettings` (or
   `@objc … func openSettings`) in the same file or its target class.
2. **Function body → destination symbol.** Grep the function's BODY (the
   whole body, not just one line) for a known destination symbol:
   `SettingsWindowView`, `SettingsWindowController`,
   `NSHostingView(rootView: Settings…)`.
3. **Link.** The menu item inherits that destination's proposed_name.

**Worked example (Settings):**

```
AppDelegate.swift:1259
    NSMenuItem(title: "Settings...", action: #selector(openSettings), keyEquivalent: ",")
```

Hop 1: grep `func openSettings` → `AppDelegate.swift:1432
@objc func openSettings()`. Hop 2: its body contains
`SettingsWindowController` (which hosts
`NSHostingView(rootView: SettingsWindowView(...))`). Hop 3: destination
`SettingsWindowView` has proposed_name `settings-window` → this menu item
is a trigger for `settings-window` and must be gated with the same name.

### Trigger snippets

Prefer the SDK helper for menu items:

```swift
if let item = onelo.features.feature("$NAME").menuItem(
    title: "…", action: #selector(existingAction)
) {
    menu.addItem(item)
}
```

The helper returns `nil` when status is `hidden` and renders greyed/badged
variants for the other statuses — one call replaces the whole
`isVisible`/`isGreyed` branching.

For non-menu triggers (toolbar items, status items, buttons), fall back to
the generic wrap:

```swift
let feat = onelo.features.feature("$NAME")
if feat.isVisible {
    // existing trigger UI
}
```

## Capability detection patterns

A capability is a user-invoked action with end-to-end logic that is NOT a
screen — export a file, start a recording, run a sync.

```bash
# @objc action handlers (menu/toolbar targets)
grep -rn --include="*.swift" -E "@objc[a-zA-Z@ ]* func [a-z][A-Za-z0-9]*(Action|Tapped|Clicked|Pressed)?\(" . 2>/dev/null
# (attributes may precede/follow @objc: `@MainActor @objc private func fooAction(` must match)
# IBActions (xib/storyboard projects)
grep -rn --include="*.swift" -E "@IBAction func" . 2>/dev/null
```

**Filter (mandatory, anti-noise):**
- Keep only handlers whose body does NOT open a window/sheet — handlers
  that present UI are triggers, handled above.
- Keep only handlers that ARE referenced from a UI element:
  `#selector(<name>)` appears in an `NSMenuItem`/`NSButton`/
  `NSToolbarItem`/keyboard-shortcut line. Pure internal functions (init
  paths, persistence, background sync) are exactly the "internal
  processes" that must NOT become features — if there is no UI call
  site, drop the candidate.

Name the action, not the widget: `export-recording`, never
`export-button`. Each capability gets TWO insertions sharing one feature
name — the handler guard plus an `isVisible` gate on the invoking UI
element (same pattern as destination + trigger).

**Capability snippet** (first line of the handler):

```swift
guard onelo.features.feature("$NAME").isEnabled else { return }
```

## Symbol extraction

- `struct AnalyticsView: View` → symbol: `AnalyticsView`
- `class ProfileViewModel` → symbol: `ProfileViewModel`
- `class SettingsViewController` → symbol: `SettingsViewController`

## Insertion point

**SwiftUI View:**
- Find `var body: some View {` or `@ViewBuilder var body: some View {`
- Insert as first line inside the body block

**ViewModel:**
- ViewModels are not screen gates (see SKILL.md Gating philosophy). Only
  capability handlers inside a ViewModel get a `guard … isEnabled else { return }`.

**ViewController:**
- Find `override func viewDidLoad()` or `override func viewWillAppear`
- Insert as first line after `super.viewDidLoad()` / `super.viewWillAppear(animated)`

## Snippets

> **API note:** Onelo's Swift SDK exposes `isVisible`, `isEnabled`, `isGreyed`
> etc. as **stored properties**, not methods. Access without parentheses
> (`feat.isEnabled`, never `feat.isEnabled()`).
>
> The snippets assume an `onelo` instance is reachable — typically a
> `@StateObject`/`@EnvironmentObject` declared in your App entry point.
> See Phase 6 of SKILL.md for the init pattern.

### Choosing the right check

Default to **`isEnabled`** for "show this content or don't" gates. It matches
the dashboard's primary on/off semantics and the Swift snippet generated by
`app.onelo.tools`, so docs and codegen stay aligned.

| Check | True when status is | Use for |
|---|---|---|
| `isEnabled` | `.enabled`, `.new`, `.beta` | **Default.** Binary gate: show content vs. hide. Paid features, A/B tests, gradual rollouts. |
| `isVisible` | anything except `.hidden` | When you render a greyed-out / upsell / "coming soon" variant in place of the real content. The status itself is the signal for *which* placeholder to show. |
| `isGreyed`, `isUpsell`, `isNew`, `isBeta`, `isComingSoon` | the matching status | Branching on a specific promotional state — pairs naturally with the `OneloFeatureBadge` view. |

**SwiftUI View:**
```swift
let feat = onelo.features.feature("$NAME")
if feat.isEnabled {
    // existing body content
}
```

`if` rather than `guard return` because SwiftUI bodies often have complex inferred
return types (multi-branch, modifiers, generics). `guard ... else { return }` only
compiles when the body's return type is `EmptyView`/`Never`-equivalent. The `if`
form works in every shape and produces an `EmptyView` automatically when disabled.

**ViewModel:**
```swift
let feat = onelo.features.feature("$NAME")
guard feat.isEnabled else { return }
```

**ViewController (UIKit):**
```swift
let feat = onelo.features.feature("$NAME")
guard feat.isEnabled else { return }
```

### Dev-mode default

A freshly instrumented `feature("$NAME")` returns `.hidden` by default until
the gate is enabled in the dashboard — fail-closed in production. During
development you can flip the default so new gates render until you toggle
them in the dashboard:

```swift
#if DEBUG
let onelo = Onelo(
    publishableKey: "...",
    baseURL: URL(string: "...")!,
    featureDefaultStatus: .enabled
)
#else
let onelo = Onelo(
    publishableKey: "...",
    baseURL: URL(string: "...")!
)
#endif
```

Available in onelo-swift ≥ 3.19.0.

### Upfront declaration & registry codegen

Lazy discovery via `feature("...")` only registers a name once the code
path actually runs. SwiftUI views behind navigation, debug-only views in
`#if DEBUG`, settings sheets opened once a month — none of them appear
in the dashboard until someone happens to navigate there.

Call `declare(...)` once at startup with every name you instrument so
the dashboard reflects the full registry from the first run:

```swift
let onelo = Onelo(/* … */)
onelo.features.declare(FeatureRegistry.all)
```

Maintain `FeatureRegistry.all` with a build-time script that greps your
sources for `feature("...")` calls — zero manual sync, zero risk of the
list drifting from reality.

#### Generated file

Path: `Sources/<Module>/Generated/FeatureRegistry.swift` (Swift Package
convention; for Xcode-only projects use the equivalent group path).

```swift
// AUTO-GENERATED. Do not edit manually.
// Regenerated on every build by scripts/generate-feature-registry.sh

enum FeatureRegistry {
    static let all: [String] = [
        "analytics-dashboard",
        "export-button",
        "settings-window",
        // ...
    ]
}
```

#### Generator script

Drop into `scripts/generate-feature-registry.sh` and `chmod +x`. Reads
two arguments (or env vars): sources directory and output file. Both
have sensible defaults so the typical case is a one-line invocation.

```bash
#!/bin/bash
set -euo pipefail

# Usage: generate-feature-registry.sh [<sources_dir>] [<output_file>]
SOURCES_DIR="${1:-${SOURCES_DIR:-Sources}}"
OUT_FILE="${2:-${OUT_FILE:-$SOURCES_DIR/Generated/FeatureRegistry.swift}}"

mkdir -p "$(dirname "$OUT_FILE")"

NAMES=$(grep -rh 'features\.feature("' "$SOURCES_DIR" \
  --include="*.swift" \
  --exclude-dir=Generated \
  | grep -oE '"[a-z][a-z0-9-]*"' \
  | sort -u)

{
  echo "// AUTO-GENERATED. Do not edit manually."
  echo "// Regenerated on every build by scripts/generate-feature-registry.sh"
  echo ""
  echo "enum FeatureRegistry {"
  echo "    static let all: [String] = ["
  while IFS= read -r name; do
    [ -n "$name" ] && echo "        $name,"
  done <<< "$NAMES"
  echo "    ]"
  echo "}"
} > "$OUT_FILE"

COUNT=$(echo "$NAMES" | grep -c '^"' || true)
echo "Generated $OUT_FILE with $COUNT features"
```

The `--exclude-dir=Generated` flag prevents the script from feeding on
its own output. The atom regex `[a-z][a-z0-9-]*` matches the SDK's
`feature()` name validation, so anything that wouldn't be a valid name
is silently ignored.

#### Build hook integration

**Xcode project (.xcodeproj or .xcworkspace):** Add a "Run Script" build
phase BEFORE "Compile Sources":

```bash
"${SRCROOT}/scripts/generate-feature-registry.sh" \
  "${SRCROOT}/Sources/MyApp" \
  "${SRCROOT}/Sources/MyApp/Generated/FeatureRegistry.swift"
```

Uncheck "Based on dependency analysis" so it runs every build, not just
when inputs change — feature names appear in arbitrary `.swift` files
and the dependency graph won't catch new instrumentations.

**Swift Package (no Xcode project):** Wrap `swift build` in a tiny
`build.sh`:

```bash
#!/bin/bash
set -euo pipefail
./scripts/generate-feature-registry.sh Sources/MyApp
swift build "$@"
```

Use `./build.sh` instead of bare `swift build` in CI and local
dev. Editor builds (Xcode opening Package.swift) won't run the hook —
either commit the registry file and accept manual sync via `./build.sh`
before every commit, or switch to an Xcode project for tighter
integration.

**SwiftPM `Package.swift` plugin** is technically possible (build tool
plugins) but adds two manifest dependencies and runs in a sandbox that
forbids `grep` outside its inputs. For most projects the wrapper script
is simpler and equally reliable.
