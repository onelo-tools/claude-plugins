# Swift iOS Patterns

## Contents
- iOS-specific classification rules
- Grep patterns (destinations)
- Trigger detection patterns
- Capability detection patterns
- Symbol extraction
- Insertion point
- Snippets

## iOS-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Swift on iOS adds three signals on top:

- **`*ViewController` → screen.** UIKit screens. Always keep.
- **`*ViewModel` → screen** (one ViewModel per screen is the convention;
  if you have row-level VMs, their suffix would be `*RowViewModel`/
  `*CellViewModel` — those still drop via the atom-suffix rule).
- **Plain `*View` is ambiguous.** SwiftUI overuses `View`. Disambiguate
  by path:
  - Inside `Screens/`, `Tabs/`, `Windows/`, `Flows/`, or any
    `Features/<X>/<X>View.swift` → treat as `screen`.
  - Anywhere else → mark `ambiguous` so the developer can decide in
    Phase 3.

### Extra Swift-only atom suffixes to drop

On top of the generic Phase 2.5 list:
`Style`, `Modifier`, `Shape`, `Path` — these are SwiftUI extension
points (`ButtonStyle`, `ViewModifier`), never user-visible features.

## Grep patterns (destinations)

```bash
# SwiftUI Views
grep -rn --include="*.swift" \
  -E "^struct ([A-Z][A-Za-z0-9]+): (some )?View" \
  . 2>/dev/null

# ViewModels
grep -rn --include="*.swift" \
  -E "^(class|struct) ([A-Z][A-Za-z0-9]+ViewModel)" \
  . 2>/dev/null

# UIKit ViewControllers
grep -rn --include="*.swift" \
  -E "^class ([A-Z][A-Za-z0-9]+ViewController)" \
  . 2>/dev/null

# Tab roots & full-screen content
grep -rn --include="*.swift" -E "\.(tabItem|fullScreenCover)\(" . 2>/dev/null
# UIKit tab/nav containers
grep -rn --include="*.swift" -E "^class ([A-Z][A-Za-z0-9]+(TabBarController|NavigationController))" . 2>/dev/null
```

**Skip:** `Tests/`, `*Tests.swift`, `*Spec.swift`, `Preview*.swift`, `Generated/`

## Trigger detection patterns

After destinations are identified, scan for code that opens or
navigates to them. Triggers inherit the destination's feature name and
get an `isVisible` snippet (not `isEnabled`).

### Grep patterns

```bash
# SwiftUI navigation
grep -rn --include="*.swift" \
  -E '(NavigationLink|fullScreenCover|sheet|popover)' \
  . 2>/dev/null

# UIKit imperative
grep -rn --include="*.swift" \
  -E '(pushViewController|present\(|presentAsModal|show\()' \
  . 2>/dev/null

# Multi-window apps (macOS / iPad)
grep -rn --include="*.swift" \
  -E '(WindowGroup\(id:|openWindow|OpenWindowAction|dismissWindow)' \
  . 2>/dev/null

# Deep link / URL scheme handlers
grep -rn --include="*.swift" \
  -E '(\.onOpenURL|application.*open url|UIApplicationDelegate.*open)' \
  . 2>/dev/null

# Buttons whose closure navigates (Button { path.append(...) } / Button { showX = true })
grep -rn --include="*.swift" -E "Button\((action:)? ?\{" . 2>/dev/null
# Tap gestures that navigate
grep -rn --include="*.swift" -E "\.onTapGesture" . 2>/dev/null
# UIKit bar buttons
grep -rn --include="*.swift" -E "UIBarButtonItem\(" . 2>/dev/null
```

**Note:** Button/onTapGesture matches are HIGH-NOISE — link only those
whose closure body (same statement or following 3 lines) references a
known destination symbol or toggles a state var that a
`.sheet(isPresented:)/.fullScreenCover` of a known destination reads.
Unlinked matches are discarded, not proposed.

### Linking heuristic

For a destination symbol `SettingsWindow` (proposed name `settings-window`),
match triggers where:

1. Symbol literal appears in the line: `NavigationLink(destination: SettingsWindow())`
2. URL path matches feature name: `case "settings": present(SettingsWindow())`
3. Window id matches: `openWindow(id: "settings")`

Substring matches are fine — Phase 3 shows them to the dev to confirm.

### Trigger snippet

```swift
let feat = onelo.features.feature("$NAME")
if feat.isVisible {
    // existing trigger UI: Button, NavigationLink, etc.
}
```

`isVisible` is true for every status except `hidden`. So:
- Admin sets `hidden` → trigger disappears.
- Admin sets `greyed`/`coming_soon`/`beta`/`new` → trigger stays visible; admin's intended UX renders.
- Admin sets `targeted` → SDK resolves rules per user.

For ViewModels / imperative code paths (e.g. URL handlers), use
`guard feat.isVisible else { return }` instead of `if`.

### Rich destination states (opt-in)

For features that support paid upsell or coming-soon teasers, the
destination can branch on multiple statuses instead of the binary
`isEnabled` default:

```swift
let feat = onelo.features.feature("$NAME")
if feat.isGreyed {
    UpsellView(feature: feat)
} else if feat.isComingSoon {
    ComingSoonView()
} else if feat.isEnabled {
    // existing real content
}
```

This is opt-in — write it once per feature where rich states matter.

## Capability detection patterns

A capability is a user-invoked action with end-to-end logic that is NOT a
screen — export a file, start a recording, run a sync.

```bash
# IBActions (xib/storyboard projects)
grep -rn --include="*.swift" -E "@IBAction func" . 2>/dev/null
# @objc action handlers (bar button / control targets)
grep -rn --include="*.swift" -E "@objc[a-zA-Z@ ]* func [a-z][A-Za-z0-9]*(Action|Tapped|Pressed)\(" . 2>/dev/null
# (attributes may precede/follow @objc: `@MainActor @objc private func fooAction(` must match)
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
