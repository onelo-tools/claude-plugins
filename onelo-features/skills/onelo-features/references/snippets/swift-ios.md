# Swift — iOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## init
<!-- onelo:snippet sdk=features lang=swift field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift

import OneloSwift

// ─────────────────────────────────────────────────────────────────────
// Onelo Features SDK — Swift (iOS / macOS)
//
// ⚙️ SETUP — feature ENVIRONMENT (test vs live), one-time per project.
//    Onelo Features keeps a SEPARATE Test and Live snapshot. In dev you want
//    TEST (your in-progress features show + auto-discover into the registry);
//    production reads LIVE. You pick it with OneloFeatureEnvironment — NO
//    special dev/test key. Your normal publishable key is used everywhere;
//    registry growth is bound to this app + device (instance id), not to a key.
//
// 1. Create file: Config/Onelo.local.xcconfig with one line:
//      ONELO_FEATURE_ENVIRONMENT = test     ← dev/staging (prod: live, or omit)
// 2. Add to .gitignore:  Config/Onelo.local.xcconfig
// 3. Xcode → Project → Info → Configurations:
//      Debug   → set xcconfig to Config/Onelo.local.xcconfig
//      Release → leave xcconfig EMPTY (no file) → resolves to LIVE, safe default
// 4. Edit Info.plist, add:
//      <key>OneloFeatureEnvironment</key>
//      <string>$(ONELO_FEATURE_ENVIRONMENT)</string>
//    Set the SAME value in your backend (e.g. Python's ONELO_FEATURE_ENVIRONMENT)
//    so app + server resolve the same snapshot.
// ─────────────────────────────────────────────────────────────────────

// Initialize once (e.g. in your App or ViewModel). No featureEnvironment arg
// needed — the SDK auto-reads the OneloFeatureEnvironment Info.plist key above
// (falling back to the ONELO_FEATURE_ENVIRONMENT process env var). Pass
// Onelo(..., featureEnvironment: "test") only if you compute it in code.
let onelo = Onelo(
  publishableKey: "onelo_pk_live_YOUR_KEY",
  callbackScheme: "myapp",
  baseURL: URL(string: "https://api.onelo.tools")!
)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=swift field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// ── IDENTIFY ──────────────────────────────────────────────
await onelo.identify(currentUser.id)

// Check features synchronously — no init() needed.
// Updates push in real-time the moment an admin clicks Deploy in the Onelo dashboard.
// The SDK also auto-refreshes when your app comes to foreground or the system
// wakes from sleep (macOS App Nap aware), and when the signed-in user's plan
// changes (purchase / upgrade / downgrade) — so newly-unlocked features appear
// immediately, no app restart. Background apps stay in sync.
//
// ⚠️ Render by STATUS — do NOT gate UI with `if feature.isEnabled { show }`.
// isEnabled is true only for enabled/new/beta, so that pattern HIDES greyed
// (locked), coming_soon and upsell features instead of showing their padlock /
// badge. Use menuItem() on macOS, or the helpers below, so locked features stay
// visible.

// macOS menu (NSMenu): menuItem() maps every status for you —
//   greyed → 🔒 padlock · coming_soon → "Coming Soon" badge · upsell → "Available in <plan>".
//   For greyed / upsell / coming_soon, when the dev enabled tap-to-upgrade (upgradeCta +
//   requiredPlan) the item is CLICKABLE and opens the upgrade flow; otherwise disabled.
//   new/beta → badge · enabled → runs your action · hidden → omitted (returns nil).
#if canImport(AppKit)
let menu = NSMenu()
if let item = onelo.features.feature("advanced-export")
        .menuItem(title: "Advanced Export…", action: #selector(openAdvancedExport)) {
    menu.addItem(item)   // greyed features appear here — disabled, with a padlock
}
#endif

// SwiftUI / UIKit: show unless hidden/disabled, then style from the status.
let f = onelo.features.feature("advanced-export")
if f.isVisible && !f.isDisabled {      // .isVisible is false ONLY for .hidden
    // A blocked tile is tappable-to-UPGRADE only when the backend says so: f.upgradeCta
    // (dashboard "Tapping the feature opens the upgrade flow") + f.requiredPlan — covers
    // greyed, upsell AND coming_soon. f.badgeLabel is the ready-made label (🔒 / "Available
    // in Pro" / …) — render it, NOT the raw requiredPlan slug. e.g.:
    //   let canUpgrade = f.upgradeCta && f.requiredPlan != nil
    //   Button(f.badgeLabel.map { "Export \($0)" } ?? "Export") {
    //       if f.isEnabled { runExport() }
    //       else if canUpgrade { Task { await onelo.openUpgrade(forPlan: f.requiredPlan!) } }
    //   }
    //   .disabled(!f.isEnabled && !canUpgrade)   // blocked + no upgrade CTA → inert
}

// Run gated code only when the feature is actually usable (enabled/new/beta):
if onelo.features.feature("advanced-export").isEnabled {
    // …
}

// Escape hatch — force a snapshot reconcile via REST. Debounced internally
// to one network call per second; safe to call from anywhere.
// You should rarely need this; deploys reach the SDK automatically.
await onelo.features.refresh()

// Helpers + HOW EACH STATUS DRIVES YOUR UI — gate INTERACTIVITY on isEnabled; a blocked
// tile is tappable-to-UPGRADE only when f.upgradeCta && f.requiredPlan != nil (the backend's
// own signal — covers greyed, upsell AND coming_soon). NEVER gate on isUpsell/isGreyed alone.
//   if !f.isVisible || f.isDisabled { EmptyView() }   // .hidden / .disabled → render nothing
//   let canUpgrade = f.upgradeCta && f.requiredPlan != nil
//   Button(f.badgeLabel.map { "Export \($0)" } ?? "Export") {
//       if f.isEnabled { runExport() }
//       else if canUpgrade { Task { await onelo.openUpgrade(forPlan: f.requiredPlan!) } }
//   }
//   .disabled(!f.isEnabled && !canUpgrade)      // blocked + no upgrade CTA → inert
// .isEnabled    → .enabled/.new/.beta — USABLE; the ONE interactivity gate.
// .isVisible    → false ONLY for .hidden → when false, render nothing. Also skip .disabled.
// .isDisabled   → .disabled — a killed feature; render nothing.
// .isNew / .isBeta   → usable; cosmetic badge only.
// .isGreyed     → .greyed      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → .coming_soon — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .isUpsell     → .upsell      — VISIBLE with "Available in <plan>"; blocked. Tappable → upgrade when upgradeCta.
// .upgradeCta   → dashboard "tap opens upgrade" toggle; PAIR WITH .requiredPlan → the tappable gate.
// .badgeLabel   → ready-made: 🔒 (greyed) · "Available in Pro" (upsell) · New/Beta/Coming Soon · nil.
// .requiredPlan / .requiredPlanLabel → plan that unlocks it (machine slug / human label — render the LABEL).
// .status       → .enabled | .new | .beta | .coming_soon | .greyed | .hidden | .upsell | .disabled
```
<!-- /onelo:snippet -->
