# Onelo Features — snippets

The official Onelo Features code for every supported platform, **baked into this
file when the plugin was published** — straight from `@onelo/snippets`, the same
source the dashboard **SDK** tab and **/docs** render from.

Each section has **install** (adding the package), **init** (creating the client
once) and **usage** (the flag check itself). Swift and macOS have no install
step — the package is added in Xcode via SPM; see
[sdk-setup.md](sdk-setup.md).

Replace `$NAME` in the usage snippet with the feature's slug, and
`onelo_pk_live_YOUR_KEY` / `https://api.onelo.tools` with the developer's real values.

Use these verbatim. The `*-patterns.md` files explain *which* code is worth a
flag and how to name it — the API shape comes from here, never from memory and
never adapted from another language.

---

## NPM

### install
<!-- onelo:snippet sdk=features lang=npm field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=npm field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  // Feature environment: 'test' | 'live' — the features discovery/targeting env,
  // INDEPENDENT of monitor's `environment` field and NOT decided by your key.
  // Discovery (features auto-registering in your dashboard Registry) works ONLY
  // in 'test'; on 'live' the registry is read-only. Browser has no env var, so
  // this config field is the switch. 'test' while developing; remove / 'live'
  // for production.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=npm field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// REACTIVITY — feature states are available immediately (no init() needed) and
// update in real-time the moment an admin clicks Deploy. subscribe() fires on every
// change (a Deploy over SSE, a plan change, an identity swap) — re-read feature()
// and re-render (React: wrap in useSyncExternalStore; Vue: bump a ref). Returns an
// unsubscribe fn; call it on teardown.
const unsubscribe = onelo.features.subscribe(() => { /* re-read + re-render */ })
const feature = onelo.features.feature('advanced-export')

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell and coming-soon states. And a GATED tile is tappable-to-UPGRADE
// only when the backend says so: feature.upgradeCta (the dashboard "Tapping the
// feature opens the upgrade flow" toggle) + feature.requiredPlan — covers greyed,
// upsell AND coming_soon. onelo.openUpgrade() routes to the working web surface:
// subscriber → Customer Portal (Change plan); otherwise → Store. Signed-in user.
if (!feature.isVisible || feature.isDisabled) {
  // 'hidden' or 'disabled' → render NOTHING
} else {
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null
  if (feature.isEnabled) {
    // enabled / new / beta → usable; run the feature on click
  } else if (canUpgrade) {
    // greyed / upsell / coming_soon + upgrade CTA → on click, open the upgrade flow:
    //   await onelo.openUpgrade()   // subscriber → Customer Portal · else → Store
    // Render feature.badgeLabel: '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  } else {
    // gated but informational (no upgrade CTA) → render feature.badgeLabel, disable the click.
  }
}

// Escape hatch — clears the local cache. You should rarely need this;
// deploys reach the SDK automatically.
onelo.features.invalidateCache()

// State helpers on feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing.
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .isUpsell     → 'upsell'      — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .upgradeCta   → the dashboard "tap opens the upgrade flow" toggle; gate onelo.openUpgrade on it.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', …) or null
// .requiredPlan / .requiredPlanLabel / .upgradeHint → plan-gate CTA data
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show + BLOCKED · hidden·disabled = don't render
```
<!-- /onelo:snippet -->

---

## REACT

### install
<!-- onelo:snippet sdk=features lang=react field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=react field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'
import { createContext, useContext, useEffect, useState } from 'react'

// Create the Onelo instance once (outside the component tree)
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  // Feature environment: 'test' | 'live' — the features discovery/targeting env,
  // INDEPENDENT of monitor's `environment` field and NOT decided by your key.
  // Discovery (features auto-registering in your dashboard Registry) works ONLY
  // in 'test'; on 'live' the registry is read-only. Browser has no env var, so
  // this config field is the switch. 'test' while developing; remove / 'live'
  // for production.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
})

export function useFeature(name: string) {
  const [state, setState] = useState(() => onelo.features.feature(name))

  useEffect(() => {
    setState(onelo.features.feature(name))
    // Live-update: subscribe() fires on every Deploy (SSE), plan change or identity
    // swap. Re-read feature() so the component re-renders. Returns the unsubscribe.
    return onelo.features.subscribe(() => setState(onelo.features.feature(name)))
  }, [name])

  return state
}
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=react field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// Usage in a component. Gate INTERACTIVITY on isEnabled; a blocked tile is
// tappable-to-UPGRADE only when feature.upgradeCta (dashboard "Tapping the feature
// opens the upgrade flow") + feature.requiredPlan — covers greyed, upsell AND
// coming_soon. onelo.openUpgrade() routes to the working web surface: subscriber →
// Customer Portal (Change plan); otherwise → Store. Needs a signed-in Onelo Auth user.
function ExportButton() {
  const feature = useFeature('advanced-export')
  if (!feature.isVisible || feature.isDisabled) return null   // hidden / disabled → nothing
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null
  return (
    <button
      disabled={!feature.isEnabled && !canUpgrade}            // blocked + no upgrade → inert
      onClick={feature.isEnabled ? runExport : () => onelo.openUpgrade()}
    >
      {feature.badgeLabel ? `Export • ${feature.badgeLabel}` : 'Export'}
    </button>
  )
}
```
<!-- /onelo:snippet -->

---

## WEB

### install
<!-- onelo:snippet sdk=features lang=web field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=web field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  // Feature environment: 'test' | 'live' — the features discovery/targeting env,
  // INDEPENDENT of monitor's `environment` field and NOT decided by your key.
  // Discovery (features auto-registering in your dashboard Registry) works ONLY
  // in 'test'; on 'live' the registry is read-only. Browser has no env var, so
  // this config field is the switch. 'test' while developing; remove / 'live'
  // for production.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=web field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// REACTIVITY — feature states are available immediately (no init() needed) and
// update in real-time the moment an admin clicks Deploy. subscribe() fires on every
// change (a Deploy over SSE, a plan change, an identity swap) — re-read feature()
// and re-render (React: wrap in useSyncExternalStore; Vue: bump a ref). Returns an
// unsubscribe fn; call it on teardown.
const unsubscribe = onelo.features.subscribe(() => { /* re-read + re-render */ })
const feature = onelo.features.feature('advanced-export')

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell and coming-soon states. And a GATED tile is tappable-to-UPGRADE
// only when the backend says so: feature.upgradeCta (the dashboard "Tapping the
// feature opens the upgrade flow" toggle) + feature.requiredPlan — covers greyed,
// upsell AND coming_soon. onelo.openUpgrade() routes to the working web surface:
// subscriber → Customer Portal (Change plan); otherwise → Store. Signed-in user.
if (!feature.isVisible || feature.isDisabled) {
  // 'hidden' or 'disabled' → render NOTHING
} else {
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null
  if (feature.isEnabled) {
    // enabled / new / beta → usable; run the feature on click
  } else if (canUpgrade) {
    // greyed / upsell / coming_soon + upgrade CTA → on click, open the upgrade flow:
    //   await onelo.openUpgrade()   // subscriber → Customer Portal · else → Store
    // Render feature.badgeLabel: '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  } else {
    // gated but informational (no upgrade CTA) → render feature.badgeLabel, disable the click.
  }
}

// Escape hatch — clears the local cache. You should rarely need this;
// deploys reach the SDK automatically.
onelo.features.invalidateCache()

// State helpers on feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing.
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .isUpsell     → 'upsell'      — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .upgradeCta   → the dashboard "tap opens the upgrade flow" toggle; gate onelo.openUpgrade on it.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', …) or null
// .requiredPlan / .requiredPlanLabel / .upgradeHint → plan-gate CTA data
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show + BLOCKED · hidden·disabled = don't render
```
<!-- /onelo:snippet -->

---

## SWIFT

### init
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

### usage
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

---

## MACOS

### init
<!-- onelo:snippet sdk=features lang=macos field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift

import OneloSwift

// ─────────────────────────────────────────────────────────────────────
// Onelo Features SDK — Swift (iOS / macOS, SwiftPM or Xcode project)
//
// ⚙️ SETUP — feature ENVIRONMENT (test vs live), one-time per developer.
//    Onelo Features keeps a SEPARATE Test and Live snapshot. In dev you want
//    TEST (your in-progress features show + auto-discover into the registry);
//    production reads LIVE. You pick it with ONELO_FEATURE_ENVIRONMENT — NO
//    special dev/test key. Your normal publishable key is used everywhere;
//    registry growth is bound to this app + device (instance id), not a key.
//    Set the SAME value in your backend (e.g. Python ONELO_FEATURE_ENVIRONMENT)
//    so app + server resolve the same snapshot. Pick ONE method:
//
//    • Method A — Xcode scheme env var (SwiftPM AND classic Xcode):
//        Product → Scheme → Edit Scheme → Run → Arguments → Environment
//        Variables → +   Name: ONELO_FEATURE_ENVIRONMENT   Value: test
//        Keep it a USER scheme (xcuserdata/, gitignored) so it only applies
//        in dev — Archive/Release builds don't see it → resolves LIVE.
//
//    • Method B — xcconfig + Info.plist (classic Xcode project):
//        Config/Onelo.local.xcconfig (add to .gitignore):
//          ONELO_FEATURE_ENVIRONMENT = test
//        Xcode → Project → Info → Configurations: Debug → link that xcconfig;
//        Release → leave empty (→ LIVE). Info.plist:
//          <key>OneloFeatureEnvironment</key>
//          <string>$(ONELO_FEATURE_ENVIRONMENT)</string>
//
//    Anything other than "test"/"live" is ignored; empty in Release → LIVE (safe).
// ─────────────────────────────────────────────────────────────────────

// No featureEnvironment arg needed — the SDK auto-reads ONELO_FEATURE_ENVIRONMENT
// (scheme env var) or the OneloFeatureEnvironment Info.plist key. Pass
// Onelo(..., featureEnvironment: "test") only if you compute it in code.
let onelo = Onelo(
  publishableKey: "onelo_pk_live_YOUR_KEY",
  callbackScheme: "myapp",
  baseURL: URL(string: "https://api.onelo.tools")!
)
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=macos field=usage -->
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

---

## ELECTRON

### install
<!-- onelo:snippet sdk=features lang=electron field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=electron field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/electron'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // your app id — REQUIRED for codesign attestation (a signed build 403s without it)
  // Feature environment: 'test' | 'live' — which features env this app READS
  // (drafts/QA vs production), INDEPENDENT of your key. Electron runs in Node,
  // so the SDK AUTO-READS process.env.ONELO_FEATURE_ENVIRONMENT: set it to
  // 'test' while developing (enables discovery — features auto-register in your
  // dashboard Registry) and 'live' (or leave unset) in production (registry is
  // read-only). Set the SAME value in your backend so both read one environment.
  // To pass it explicitly instead of the env var:
  //   featureEnvironment: process.env.ONELO_FEATURE_ENVIRONMENT,
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
  // Re-sync on OS resume / app re-activate (on by default). false = poll + SSE only.
  // autoLifecycleRefresh: true,
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// REACTIVITY — the SDK runs in the MAIN process. subscribe() fires on every cache
// change (an admin's Deploy over SSE, a plan change, a lifecycle resync); forward it
// to your window so the renderer re-renders. Returns an unsubscribe fn.
const unsubscribe = onelo.features.subscribe(() => {
  mainWindow.webContents.send('onelo:features-changed')
})
// Expose feature state to the renderer over IPC. ⚠️ Send featureSnapshot(name) —
// a PLAIN object with every getter (isEnabled/isVisible/badgeLabel/upgradeHint…)
// materialized as data. A raw feature() instance loses its getters over IPC's
// structured-clone, so the renderer would see them all as undefined and hide
// every tile. In the renderer: ipcRenderer.on('onelo:features-changed', refetch)
// and ipcRenderer.invoke('onelo:feature', name) → this handler.
ipcMain.handle('onelo:feature', (_e, name) => onelo.features.featureSnapshot(name))

// Feature states are available immediately — no init() needed. In the MAIN process
// read them directly (getters work here); to the renderer send featureSnapshot(name).
const f = onelo.features.feature('advanced-export')

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell and coming-soon states. And a GATED tile is tappable-to-UPGRADE
// only when the backend says so: f.upgradeCta (the dashboard "Tapping the feature
// opens the upgrade flow" toggle) + f.requiredPlan — covers greyed, upsell AND
// coming_soon. onelo.openUpgrade(plan) opens the hosted upgrade in the system
// browser (subscriber → "Change plan" pinned; non-subscriber → store).
if (!f.isVisible || f.isDisabled) {
  // 'hidden' or 'disabled' → render NOTHING
} else {
  const canUpgrade = f.upgradeCta && f.requiredPlan != null
  if (f.isEnabled) {
    // enabled / new / beta → usable; run the feature on click
  } else if (canUpgrade) {
    // greyed / upsell / coming_soon + upgrade CTA → on click, open the upgrade flow:
    //   onelo.openUpgrade(f.requiredPlan)
    // Render f.badgeLabel: '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  } else {
    // gated but informational (no upgrade CTA) → render f.badgeLabel, disable the click.
  }
}

// Escape hatch for edge cases — clears local cache (and notifies subscribers).
// You should rarely need this; deploys reach the SDK automatically.
onelo.features.invalidateCache()
// On teardown: unsubscribe()

// State helpers on feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing (isVisible is still true for it).
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta (a
//                 plan-gated coming_soon CAN carry an upgrade CTA — don't assume it's inert).
// .isUpsell     → 'upsell'      — VISIBLE with "Available in <plan>"; blocked. Tap → onelo.openUpgrade.
// .upgradeCta   → dashboard "tap opens upgrade" toggle. PAIR WITH .requiredPlan → the tappable gate.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', 'Coming Soon', …) or null
// .requiredPlan / .requiredPlanLabel → plan that unlocks it (slug / human label).
// .upgradeHint → { requiredPlan, currentStatus } when plan-gated — for upgrade-prompt UI.
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show +
//   (tap→upgrade when upgradeCta, else blocked) · hidden·disabled = don't render
```
<!-- /onelo:snippet -->

---

## REACTNATIVE

### install
<!-- onelo:snippet sdk=features lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/react-native'
import { useEffect, useState } from 'react'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // app id / package name for X-Bundle-Id — iOS auto-derives, ANDROID needs it (else bundle_id_mismatch 403 under enforcement)
  // Feature environment: 'test' | 'live' — which features snapshot this app READS
  // (drafts/QA vs production) and whether DISCOVERY is active, INDEPENDENT of your
  // key. React Native has no process.env, so this config field is the ONLY switch
  // (there is no env-var fallback). Set 'test' while developing — your in-progress
  // features show and auto-register in the dashboard Registry; on 'live' (or unset)
  // the registry is read-only. Set the SAME value in your backend.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
  // Foreground force-refreshes the snapshot (on by default); false = poll + SSE only.
  // autoLifecycleRefresh: true,
})

// Read a feature's state and STAY IN SYNC. FeatureState is a synchronous snapshot,
// and this hook re-renders automatically the instant the SDK cache changes — an
// admin's Deploy (pushed over the shared SSE connection), a foreground/identity
// resync, or invalidateCache(). subscribe() is purely local (no extra network, DB,
// or SSE) and returns an unsubscribe fn, which useEffect runs on unmount.
export function useFeature(name: string) {
  const [state, setState] = useState(() => onelo.features.feature(name))

  useEffect(() => {
    setState(onelo.features.feature(name))
    return onelo.features.subscribe(() => setState(onelo.features.feature(name)))
  }, [name])

  return state
}
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call identify() after login so per-user /
// per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id)

// Feature states are available immediately — no init() needed. Updates push in
// real-time the moment an admin clicks Deploy in the dashboard; the SDK also
// resyncs on app-foreground and when the signed-in user's plan changes.

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell (upgrade) and coming-soon states you may have configured.
function ExportButton() {
  const feature = useFeature('advanced-export')
  // 'hidden' or 'disabled' → render NOTHING.
  if (!feature.isVisible || feature.isDisabled) return null

  // A plan-gated tile (greyed / upsell / coming_soon) is tappable-to-UPGRADE ONLY
  // when the backend says so: upgradeCta is the dashboard "Tapping the feature opens
  // the upgrade flow" toggle, and requiredPlan is the plan to open. This ONE gate
  // covers all three gated states — do NOT special-case isGreyed()/isUpsell().
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null

  // enabled / new / beta → usable.
  if (feature.isEnabled) return <Button onPress={runExport}>Export</Button>

  // gated + upgrade CTA → tap opens the hosted upgrade flow (subscriber → "Change
  // plan" with the plan pinned; non-subscriber → store). badgeLabel is ready-made:
  // '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  if (canUpgrade) {
    return (
      <Button onPress={() => onelo.openUpgrade(feature.requiredPlan!)}>
        Export {feature.badgeLabel}
      </Button>
    )
  }

  // gated but informational (no upgrade CTA) → visible, tap DISABLED.
  return <Button disabled>Export {feature.badgeLabel}</Button>
}

// Escape hatch — clears the local cache. You should rarely need this;
// deploys reach the SDK automatically.
onelo.features.invalidateCache()

// OPEN THE UPGRADE FLOW — onelo.openUpgrade(plan) opens the hosted upgrade in the
// system browser: an active subscriber gets "Change plan" (target pinned), a
// non-subscriber gets the store. The backend decides — you just pass the plan.
// The tappable-to-upgrade gate is feature.upgradeCta && feature.requiredPlan != null
// (covers greyed, upsell AND coming_soon). Never gate on isGreyed()/isUpsell() alone.
//
// State helpers on onelo.features.feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing (isVisible is still true for it).
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta (a plan-gated
//                 coming_soon CAN carry an upgrade CTA — don't assume it's inert).
// .isUpsell     → 'upsell'      — VISIBLE with "Available in <plan>"; blocked. Tap → onelo.openUpgrade.
// .upgradeCta   → dashboard "tap opens upgrade" toggle. PAIR WITH .requiredPlan → the tappable gate.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', 'Coming Soon', …) or null.
// .requiredPlan / .requiredPlanLabel → plan that unlocks it (slug / human label).
// .upgradeHint → { requiredPlan, currentStatus } when plan-gated — for upgrade-prompt UI.
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show +
//   (tap→upgrade when upgradeCta, else blocked) · hidden·disabled = don't render
```
<!-- /onelo:snippet -->

---

## ANDROID

### install
<!-- onelo:snippet sdk=features lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val config = OneloConfig(
    publishableKey = "onelo_pk_live_YOUR_KEY",
    apiUrl = "https://api.onelo.tools",
    // Features environment: "test" | "live" — a SEPARATE Test and Live snapshot. In dev
    // use "test" (your in-progress features show + auto-discover into the dashboard
    // Registry); production reads "live" (Registry is read-only). NOT decided by your key.
    // Set the SAME value on your backend so app + server resolve the same snapshot. Omit /
    // "live" for release builds.
    featureEnvironment = "test",
)
val onelo = Onelo(config, this)
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// ── IDENTIFY (skip if you use Onelo Auth — automatic). identify() is suspend. ──
lifecycleScope.launch { onelo.identify(currentUser.id) }

// Feature state is available synchronously via feature(name). Updates push in REAL TIME
// the moment an admin clicks Deploy (SSE); the SDK AUTO-refreshes when the app returns to
// the FOREGROUND and re-resolves when the signed-in user's plan changes — newly-unlocked
// features appear with NO restart. In Compose, collect onelo.features.updates to recompose.

// ── CORRECT rendering pattern (Jetpack Compose) — copy this shape ────────────
// Two rules: (1) HIDDEN or DISABLED → render NOTHING; (2) otherwise render, and let
// f.badgeLabel show the padlock / hint. A GATED feature — GREYED (dashboard "Locked",
// shows 🔒) or UPSELL (shows "Available in <plan>") — becomes tappable → the upgrade flow
// ONLY when the dashboard's "Tapping the feature opens the upgrade flow" toggle is on.
// That toggle is f.upgradeCta; the backend sends it for greyed, upsell AND coming_soon
// (any plan-gated status with a requiredPlan) — so gate the tap on upgradeCta + requiredPlan.
@Composable
fun AdvancedExport(onelo: Onelo) {
    // RECOMPOSE when features change (SSE / poll / foreground). This is the Compose
    // equivalent of Swift's @Observable OneloFeatures — WITHOUT collecting updates the UI
    // shows a stale one-time snapshot and a dashboard Deploy won't appear until the next
    // recomposition. Re-read feature() keyed on the tick.
    val tick by onelo.features.updates.collectAsState()
    val f = remember(tick) { onelo.features.feature("advanced-export") }
    val scope = rememberCoroutineScope()
    when {
        !f.isVisible() || f.isDisabled() -> Unit          // HIDDEN or DISABLED → render NOTHING
        else -> {
            // Backend's own signal: upgradeCta (the "tapping opens upgrade" toggle) is set
            // ONLY when a requiredPlan exists — for greyed, upsell AND coming_soon. Gate on
            // both (NOT on isGreyed()/isUpsell(), which would miss a coming_soon upgrade tile).
            val gatedUpgrade = f.upgradeCta && f.requiredPlan != null
            Button(
                // onelo.openUpgrade(plan) opens the hosted upgrade flow (suspend → coroutine);
                // or route to your own paywall. Non-gated taps run the real feature.
                onClick = {
                    if (gatedUpgrade) scope.launch { onelo.openUpgrade(f.requiredPlan ?: "") }
                    else openExport()
                },
                enabled = f.isEnabled() || gatedUpgrade,  // gated status WITHOUT an upgrade
                                                          // CTA (informational) stays DISABLED
            ) {
                Text(if (f.isComingSoon()) "Export (coming soon)" else "Export")
                // badgeLabel renders it all: 🔒 (greyed) · "Available in <plan>" (upsell)
                // · New / Beta / Coming Soon. Null → render nothing.
                f.badgeLabel?.let { Text("  $it") }
            }
        }
    }
}

// Escape hatch — force a REST reconcile (debounced 1/sec, suspend). Rarely needed: the SDK
// auto-refreshes on foreground and pushes over SSE.
lifecycleScope.launch { onelo.features.refresh() }

// ── Status → UI ──────────────────────────────────────────────────────────────
// Interactivity = isEnabled() (ENABLED/NEW/BETA) OR a gated feature WITH an upgrade CTA.
// The tappable-to-upgrade gate is f.upgradeCta && f.requiredPlan != null (covers greyed,
// upsell AND coming_soon). Never gate on isGreyed()/isUpsell() alone — you'd miss coming_soon.
// .isEnabled()    → ENABLED, NEW, BETA — USABLE (NEW/BETA also carry a cosmetic badge).
// .isVisible()    → false ONLY for HIDDEN. ALSO hide DISABLED via isDisabled().
// .isGreyed()     → GREYED (dashboard "Locked") — VISIBLE with 🔒; blocked. Tappable →
//                   upgrade ONLY when f.upgradeCta.
// .isUpsell()     → UPSELL — VISIBLE with "Available in <plan>"; blocked. Tappable →
//                   upgrade ONLY when f.upgradeCta.
// .isComingSoon() → COMING_SOON — VISIBLE, blocked. Tappable → upgrade when f.upgradeCta
//                   (a plan-gated coming_soon CAN carry an upgrade CTA — don't assume inert).
// .badgeLabel     → ready-made: 🔒 (greyed) · "Available in <plan>" (upsell) · New/Beta/Coming Soon · null.
// .upgradeHint    → UpgradeHint(requiredPlan, currentStatus) when plan-gated — for upgrade-prompt UI.
// .upgradeCta     → dashboard "tap opens upgrade" toggle; applies to greyed + upsell.
// .requiredPlan / .planLabel → the plan that unlocks it (slug / human label).
// .updates        → StateFlow — collectAsState() in Compose to recompose on live changes.
// .status         → ENABLED | NEW | BETA | COMING_SOON | GREYED | UPSELL | HIDDEN | DISABLED
```
<!-- /onelo:snippet -->

---

## KOTLIN

### init
<!-- onelo:snippet sdk=features lang=kotlin field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts: implementation("tools.onelo:onelo-kotlin:1.+")  (from mavenCentral)

import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val config = OneloConfig(
    publishableKey = "onelo_pk_live_YOUR_KEY",
    apiUrl = "https://api.onelo.tools",
)
val onelo = Onelo(config, this)
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=kotlin field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// onelo-kotlin is a BACKEND / JVM SDK — isEnabled() gates your server logic.
// It is true for ENABLED/NEW/BETA and false for GREYED/COMING_SOON/UPSELL/HIDDEN/
// DISABLED, so ONE check denies every non-usable state.
val f = onelo.features.feature("advanced-export")
if (!f.isEnabled()) {
    // not available for this user → deny / 403
    return
}
// serve the feature

// If you render UI from Kotlin (e.g. Compose Desktop), gate interactivity on
// isEnabled() too — NOT on isGreyed() alone (that leaves coming_soon clickable):
//   button.enabled = f.isEnabled() || f.isUpsell()   // greyed + coming_soon stay disabled

// Status → meaning (the rule that prevents the coming_soon trap):
// .isEnabled()    → ENABLED, NEW, BETA — USABLE (the gate).
// .isVisible()    → false ONLY for HIDDEN → don't surface the item at all.
// .isNew() / .isBeta()  → usable; cosmetic badge only.
// .isGreyed()     → GREYED      — NOT usable (show locked / deny).
// .isComingSoon() → COMING_SOON — NOT usable, EXACTLY like GREYED. NOT a badge.
// .isUpsell()     → UPSELL      — NOT usable; route to upgrade.
// .status         → ENABLED | NEW | BETA | COMING_SOON | GREYED | UPSELL | HIDDEN | DISABLED
```
<!-- /onelo:snippet -->

---

## FLUTTER

### install
<!-- onelo:snippet sdk=features lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
# pubspec.yaml
dependencies:
  onelo:
    git:
      url: https://github.com/onelo-tools/onelo-flutter
      ref: main
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

// Onelo Features keeps SEPARATE Test and Live snapshots. In dev use 'test'
// (your in-progress features show + auto-discover into the registry); production
// reads 'live'. Flutter has no process.env, so pass featureEnvironment
// explicitly — the SAME value your backend uses (ONELO_FEATURE_ENVIRONMENT) so
// app + server resolve the same snapshot.
final onelo = Onelo(
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp',
  featureEnvironment: 'test', // 'test' in dev/staging · 'live' (or omit) in prod
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod;
  // pass FeatureStatus.enabled in dev to preview new gates before toggling them.
  // featureDefaultStatus: FeatureStatus.hidden,
  // Foreground/wake forces a REST resync (on by default); set false for poll+SSE only.
  // autoLifecycleRefresh: true,
);
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// If you use your own auth system, call onelo.identify(userId) after login so
// per-user/per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id);

// SECURE MODE (recommended for paid apps): your backend signs the userId with
// the app secret_key — hash = HMAC-SHA256(secret_key, "user:" + userId) — so
// per-user targeting can't be spoofed with a raw id.
//   final hash = await myBackend.fetchOneloIdentity(currentUser.id);
//   await onelo.identify(currentUser.id, userIdHash: hash);

// Optionally await the first resolve so UI renders without a "hidden" flash:
await onelo.features.ready();

// Read features synchronously anywhere. Updates arrive automatically in REAL
// TIME over SSE (Deploy / kill-switch / plan change), with a 60s poll +
// foreground/wake resync as fallback.
//
// ⚠️ Render by STATUS — do NOT gate visibility with isEnabled: it's true only
// for enabled/new/beta, so it HIDES greyed (locked) / coming_soon / upsell.
final f = onelo.features.feature('advanced-export');
if (f.isVisible) {                    // false ONLY when status is hidden
  //   f.isGreyed   → 🔒 locked
  //   f.isUpsell   → "Available in ${f.planLabel}"  // Pro / Business / …
  //   f.badgeLabel → 'New' | 'Beta' | 'Coming Soon' | '🔒' (greyed) |
  //                  'Available in <plan>' (upsell) | null — render as-is
}

// Run gated code only when the feature is actually usable (enabled/new/beta):
if (onelo.features.isEnabled('advanced-export')) {
  // …
}

// REACT to changes — OneloFeatures is a ChangeNotifier, so widgets can rebuild
// the moment a flag flips (Deploy, plan change, resync):
//   ListenableBuilder(
//     listenable: onelo.features,
//     builder: (context, _) =>
//       onelo.features.isEnabled('advanced-export') ? EnabledView() : LockedView(),
//   )

// Escape hatch — force a REST reconcile (debounced to 1/sec). Rarely needed;
// SSE + poll reach the SDK automatically.
await onelo.features.refresh();

// GATED MODULE DOWNLOADS (optional) — mint a short-lived, per-user token so your
// CDN serves gated module code only to entitled users. Requires an identified
// user; throws OneloFeaturesException (notEntitled / notAuthenticated) otherwise.
//   final token = await onelo.features.moduleToken('advanced-export');
//   // send the token to your CDN; it verifies it server-side before serving.

// OPEN THE UPGRADE FLOW — onelo.openUpgrade(plan) opens the hosted upgrade in the
// system browser: an active subscriber gets "Change plan" (target pinned); a
// non-subscriber gets the store. The backend decides — you just pass the plan.
//
// A gated tile is TAPPABLE-to-upgrade exactly when the backend says so:
//   f.upgradeCta   → the dashboard "Tapping the feature opens the upgrade flow" toggle
//   f.requiredPlan → the plan slug to upgrade to (pass it to openUpgrade)
// This is the ONE gate — it works for BOTH an "Available in <plan>" upsell AND a
// LOCKED (padlock) tile. A locked tile is NOT inert: when upgradeCta is on it must
// open the upgrade on tap.
//
//   if (!f.isVisible) return const SizedBox.shrink();          // hidden → render nothing
//   final canUpgrade = f.upgradeCta && f.requiredPlan != null; // backend: this tap opens upgrade
//   ElevatedButton(
//     onPressed: f.isEnabled ? runExport                        // usable → run it
//         : canUpgrade ? () => onelo.openUpgrade(f.requiredPlan!)  // gated + tappable → upgrade
//         : null,                                               // gated, informational only → disabled
//     child: Text(f.isEnabled ? 'Export'
//         : f.isComingSoon ? 'Export (soon)'
//         : canUpgrade ? 'Upgrade to ${f.planLabel}'           // e.g. "Upgrade to Pro"
//         : (f.badgeLabel ?? 'Export')),                        // 🔒 / New / Beta / Available in <plan>
//   )
// .isEnabled → enabled/new/beta — USABLE; the ONE interactivity gate.
// .isVisible → false only for hidden → when false, render nothing.
// .isNew / .isBeta → usable; cosmetic badge only.   .badgeLabel → ready label or null
// .isGreyed / .isComingSoon → VISIBLE but BLOCKED; a locked tile is TAPPABLE→upgrade when upgradeCta.
// .isUpsell → VISIBLE but BLOCKED; tap → onelo.openUpgrade.   .reason .requiredPlan .requiredPlanLabel .planLabel
// .upgradeCta → true when the dashboard says a gated tap should open the upgrade flow. PAIR WITH
//   .requiredPlan (the target) → the correct tappable-to-upgrade gate for greyed AND upsell.
// .upgradeHint → UpgradeHint(requiredPlan, currentStatus)? — convenience that bundles the
//   plan-path check (non-null when reason=='plan' + a requiredPlan + a gated status). For the
//   tap gate prefer .upgradeCta + .requiredPlan, which also covers user-override upsells.
// .status → enabled | disabled | greyed | hidden | upsell | newFeature | beta | comingSoon
```
<!-- /onelo:snippet -->

---

## PYTHON

### install
<!-- onelo:snippet sdk=features lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install git+https://github.com/onelo-tools/onelo-python.git
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=python field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# ─────────────────────────────────────────────────────────────────
# Onelo Features SDK — Python (server-side / backend)
#
# 🔑 KEY — your live secret key (onelo_sk_live_*). Acts like a classic
#    server API key: SSE stream, identify(), monitor, auth verification.
#    KEEP IT THE SAME in dev/staging/prod. Get it from: dashboard →
#    API Keys → Secret keys.
#
# 🌐 FEATURE ENVIRONMENT — "test" or "live". THIS selects which feature
#    snapshot the server reads AND whether feature DISCOVERY is active —
#    NOT the key (no separate test/discovery key needed anymore). In
#    dev/staging set "test": the SDK reads the Test snapshot, registers
#    the slugs your code checks into the dashboard registry, and shows
#    online under the Test tab. In prod set "live" (or omit → live).
#    Set the SAME value in your client app (Swift OneloFeatureEnvironment)
#    so app + backend resolve the same snapshot.
#    Registry growth is bound to this app + INSTANCE — a stable per-process
#    id (set ONELO_INSTANCE_ID for containers; otherwise auto-generated and
#    persisted). The dashboard authorizes/unpins that instance under
#    Features → Registry → Deploy access. No key type is involved.
#    SETUP — just an env var per deployment target:
#      .env.staging    →  ONELO_FEATURE_ENVIRONMENT=test
#      .env.production →  ONELO_FEATURE_ENVIRONMENT=live   (or omit)
#
# 🔒 Keys: keep OUT of git (.env in .gitignore). Load prod secrets via
#    Vault / AWS Secrets Manager / K8s Secrets / etc.
# ─────────────────────────────────────────────────────────────────

# ── Module-level setup (runs once per process) ──────────────────
import os
from onelo import Onelo

onelo = Onelo(
    secret_key=os.environ["ONELO_SECRET_KEY"],  # onelo_sk_live_*
    feature_environment=os.environ.get("ONELO_FEATURE_ENVIRONMENT"),  # "test" in staging, "live"/unset in prod
    api_url="https://api.onelo.tools",
    # Optional: ONELO_INSTANCE_ID for a stable instance identity in
    # containers (else a per-process id is generated + persisted).
    # Optional knobs (sensible defaults — uncomment to override):
    # app_version="1.0.0",     # surfaced in SDK telemetry
    # strategy="auto",         # "auto" | "sse" | "polling" (SDK-side SSE fallback)
)

# Optional: register all known features upfront so they appear in
# the dashboard registry without waiting for code paths to execute.
onelo.features.declare([
    # add your feature names here, e.g. "chat-stream", "voice-stream"
])

# Recommended: block until first refresh so the very first request
# already sees real feature state (avoids 'hidden' fail-closed defaults).
# Both forms are equivalent — pick whichever reads better in your code.
onelo.features.ready(timeout=2.0)   # parity with Swift's onelo.features.ready()
# onelo.ready(timeout=2.0)          # alias on the client
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=python field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# ── Usage (inside your route handler) ───────────────────────────
# ⚠️ identify() sets ONE process-global identity and the SDK swaps its
# whole cache to that user's targeted snapshot (full SSE reconnect on
# each switch). It is NOT per-request state. Use it only when the
# entire process acts as a single user — CLI tools, worker jobs,
# single-tenant services. In a multi-user backend, calling
# identify(user.id) per request would race: concurrent requests share
# the one identity, so user A can be evaluated with user B's targeting.
#
# Multi-user backend rule of thumb:
#   • Global flags (no per-user / per-plan targeting): just call
#     feature(name) — no identify() at all. This is the common case.
#   • Per-user / per-plan targeting server-side: use for_user(user_id)
#     (below). It is STATELESS and multi-user-safe — resolves THIS
#     user's plan-gated features without touching the global identity,
#     so concurrent requests can't race.
#
# Heads-up: if a plan-gated feature (status upsell/greyed) is read through
# the global feature() path, the SDK logs a one-time warning pointing you to
# for_user() — it's evaluating the gate against the shared identity, not the
# request's user. Silence with ONELO_SUPPRESS_GATING_WARNING=1 if intentional
# (e.g. a single-user CLI showing a teaser).

# Global flag (no targeting) — the common case:
@app.get("/export")
async def export(user = Depends(current_user)):
    if not onelo.features.feature("advanced-export").is_enabled:
        raise HTTPException(404)
    # ... run the feature

# Per-user / per-plan gating — resolve for THIS request's user. Cached
# per-user for ~30s so a busy backend (HTTP handlers, WebSocket servers)
# doesn't re-hit the network every request. Fail-closed: a network error
# resolves every feature to hidden, never raises.
@app.websocket("/ws/face")
async def face(ws, user = Depends(current_user)):
    uf = await onelo.features.for_user(user.id)
    if not uf.feature("face-stream").is_enabled:
        await ws.close()
        return
    # ... stream

# Server-rendered upsell — tell the user WHICH plan unlocks a locked feature.
# The backend resolves the plan; you just render its label. Works on both
# feature() and for_user(). upgrade_hint is None when there's nothing to upsell.
@app.get("/reports")
async def reports(user = Depends(current_user)):
    feat = (await onelo.features.for_user(user.id)).feature("advanced-reports")
    if feat.is_enabled:
        return render_reports()
    if feat.upgrade_hint:                       # e.g. "Pro"
        return render_locked(f"Available in {feat.upgrade_hint}", cta=feat.upgrade_cta)
    return render_hidden()

# After YOUR backend processes a plan change (e.g. your own Stripe webhook,
# or a grant/revoke), drop the cached snapshot so the NEXT for_user() is fresh
# immediately instead of waiting out the ~30s TTL:
# onelo.features.invalidate_user(user_id)   # pass nothing to clear every user

# Single-user processes (worker / CLI) may pin an identity once:
# onelo.identify(job_user_id)   # ...and onelo.identify(None) to clear

# Other property checks on the Feature returned by feature(name):
# .is_visible          → True for any visible status; False for "hidden" AND for
#                        any status this SDK build doesn't recognise (fail-closed)
# .is_greyed           → True when status is "greyed"
# .is_new              → True when status is "new"
# .is_beta             → True when status is "beta"
# .is_coming_soon      → True when status is "coming_soon"
# .is_upsell           → True when status is "upsell"
# .is_known            → False if the backend sent a status newer than this SDK
#                        build — a hint to bump the SDK (still fail-closed as hidden)
# .status              → wire status ("enabled" | "new" | "beta" | "coming_soon" | "greyed" | "upsell" | "hidden")
#
# Upsell metadata (attached to plan-gated features — for server-rendered CTAs):
# .upgrade_hint        → human plan label to render, e.g. "Pro"; None when nothing to upsell
# .required_plan_label → same human label as a raw field (.required_plan = machine key)
# .upgrade_cta         → True if you enabled a tap-to-upgrade action for this feature
# .reason              → why this status resolved ("plan" | "user_override" | "paywall_off" | ...)
```
<!-- /onelo:snippet -->

---

## NODE

### install
<!-- onelo:snippet sdk=features lang=node field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=node field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Onelo Features SDK — Node.js (server-side / backend)
//
// 🔑 KEY — your live SECRET key (onelo_sk_live_*). Server credential for the
//    SSE stream, features, auth verification, monitor. NOT a publishable key.
//    KEEP IT THE SAME across dev/staging/prod. Dashboard → API Keys → Secret keys.
//
// 🌐 FEATURE ENVIRONMENT — "test" or "live". Selects which feature snapshot the
//    server reads AND whether feature DISCOVERY is active. In dev/staging set
//    "test" (reads the Test snapshot, registers the slugs your code checks into
//    the dashboard registry); in prod set "live" (or omit → live). Set the SAME
//    value in your client app so app + backend resolve the same snapshot.
//      ONELO_FEATURE_ENVIRONMENT=test   (staging)   /   =live or unset (prod)
import { Onelo } from '@onelo/node'

const onelo = new Onelo({
  secretKey: process.env.ONELO_SECRET_KEY!,   // onelo_sk_live_*
  apiUrl: 'https://api.onelo.tools',
  featureEnvironment: process.env.ONELO_FEATURE_ENVIRONMENT as 'test' | 'live' | undefined,
  // Optional: strategy 'auto' | 'sse' | 'polling'; appVersion; pollInterval.
})

// Optional: register known features so they appear in the dashboard registry
// without waiting for code paths to run.
onelo.features.declare(['advanced-export', 'face-stream'])

// Recommended: block until the first snapshot lands so the very first request
// sees real state (not the 'hidden' fail-closed default). Returns a Promise<boolean>.
await onelo.features.ready(2000)
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=node field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Global flag (no per-user targeting) — the common case. Synchronous read:
app.get('/export', (req, res) => {
  if (!onelo.features.feature('advanced-export').isEnabled) return res.status(404).end()
  // ...run the feature
})

// Per-user / per-plan gating — resolve for THIS request's user. STATELESS and
// multi-user-safe (no global identity, no race). Cached per-user ~30s; fail-closed
// (a network error resolves every feature to hidden, never throws):
app.get('/reports', async (req, res) => {
  const uf = await onelo.features.forUser(req.oneloUser!.id)
  const feat = uf.feature('advanced-reports')
  if (feat.isEnabled) return res.json(await buildReports())
  // Server-rendered upsell — tell the user WHICH plan unlocks it:
  if (feat.upgradeHint) return res.json({ locked: `Available in ${feat.upgradeHint}`, cta: feat.upgradeCta })
  return res.status(404).end()
})

// After YOUR backend processes a plan change (your Stripe webhook / a grant),
// drop the cached snapshot so the NEXT forUser() is fresh immediately:
// onelo.features.invalidateUser(userId)   // pass nothing to clear every user

// ⚠️ onelo.identify(userId) sets ONE process-global identity (full SSE reconnect
// per switch) — only for single-user processes (worker / CLI). In a multi-user
// backend use forUser() instead; the SDK warns if a plan-gated feature (upsell/
// greyed) is read through the global feature() path.

// Feature property checks: .isEnabled · .isVisible (fail-closed for unknown status)
// · .isGreyed · .isNew · .isBeta · .isComingSoon · .isUpsell · .isKnown · .status
// Upsell metadata: .upgradeHint ("Pro") · .requiredPlan · .requiredPlanLabel · .upgradeCta · .reason
```
<!-- /onelo:snippet -->

---

## PHP

### install
<!-- onelo:snippet sdk=features lang=php field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=features lang=php field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
<?php
// ─────────────────────────────────────────────────────────────────
// Onelo Features SDK — PHP (server-side / backend)
//
// 🔑 KEY — your live SECRET key (onelo_sk_live_*). Server credential for the
//    feature stream/poll, forUser resolve, monitor, auth verification. NOT a
//    publishable key. KEEP IT THE SAME across dev/staging/prod. Dashboard →
//    API Keys → Secret keys. Load from env — never hard-code a secret.
//
// 🌐 FEATURE ENVIRONMENT — "test" or "live". Selects which snapshot the server
//    reads AND whether feature DISCOVERY is active. In dev/staging set "test"
//    (reads the Test snapshot + registers the slugs your code checks); in prod
//    set "live" (or omit). Registry growth is bound to app + INSTANCE (set
//    ONELO_INSTANCE_ID in containers; else auto-generated + persisted under
//    ~/.onelo). Authorize/unpin the instance under Features → Registry → Deploy
//    access. No key type involved.
//      ONELO_FEATURE_ENVIRONMENT=test   (staging)  /  =live or unset (prod)
//
// ⚙️ HOW FEATURES STAY FRESH IN PHP — pick one:
//   A) Zero-infra (default): each request lazily polls once. Just construct the
//      client; the FIRST feature() read does one GET /poll. Feature changes land
//      on the next request (no sub-second push).
//   B) Live streaming (recommended for prod): run the listener daemon
//         php artisan onelo:features:listen
//      as a supervised process (systemd / supervisor). It holds the SSE stream
//      and writes a shared snapshot file your FPM workers read — live
//      kill-switches, exactly like the Swift/Python/Node SDKs. Point BOTH the
//      daemon and the web client at the SAME store file and turn auto-refresh off
//      (Mode B below). Under Laravel Octane/Swoole run the Listener in-process.
// ─────────────────────────────────────────────────────────────────

use Onelo\Onelo;
use Onelo\Features\FileStore;

// ── Mode A — zero-infra (per-request poll) ──────────────────────
$onelo = new Onelo(
    secretKey: getenv('ONELO_SECRET_KEY') ?: throw new \RuntimeException('ONELO_SECRET_KEY is required'),
    apiUrl: 'https://api.onelo.tools',
    featureEnvironment: getenv('ONELO_FEATURE_ENVIRONMENT') ?: null,  // "test" staging, "live"/unset prod
);

// ── Mode B — live streaming: share the daemon's snapshot file ────
// $store = new FileStore(getenv('ONELO_FEATURES_STORE_PATH') ?: sys_get_temp_dir() . '/onelo/features.json');
// $onelo = new Onelo(
//     secretKey: getenv('ONELO_SECRET_KEY') ?: throw new \RuntimeException('ONELO_SECRET_KEY is required'),
//     apiUrl: 'https://api.onelo.tools',
//     featureEnvironment: getenv('ONELO_FEATURE_ENVIRONMENT') ?: null,
//     featureStore: $store,          // the SAME file the daemon writes
//     featureAutoRefresh: false,     // the daemon keeps it fresh
// );
// ...then run:  php artisan onelo:features:listen   (supervised)

// Optional: register known features upfront so they appear in the dashboard
// registry without waiting for code paths to execute.
$onelo->features->declare([
    // 'chat-stream', 'voice-stream',
]);

// Recommended: block until the first snapshot lands so the very first request
// sees real state (not the 'hidden' fail-closed default).
$onelo->features->ready(2.0);
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=features lang=php field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
<?php
// ── Usage (inside your controller / route handler) ──────────────
// Global flag (no per-user targeting) — the common case:
if (!$onelo->features->isEnabled('advanced-export')) {
    abort(404);
}
// ... run the feature

// Per-user / per-plan gating — resolve for THIS request's user. STATELESS and
// multi-user-safe (no global identity, no cross-request race). Cached per user
// ~30s. Fail-closed: a network error resolves every feature to hidden.
$uf = $onelo->features->forUser($user->id);
if (!$uf->feature('face-stream')->isEnabled()) {
    abort(403);
}

// Server-rendered upsell — tell the user WHICH plan unlocks a locked feature.
$feat = $onelo->features->forUser($user->id)->feature('advanced-reports');
if ($feat->isEnabled()) {
    return view('reports');
}
if ($feat->upgradeHint() !== null) {              // e.g. "Pro"
    return view('locked', ['plan' => $feat->upgradeHint(), 'cta' => $feat->upgradeCta]);
}
return view('hidden');

// After YOUR backend processes a plan change (your own Stripe webhook, a
// grant/revoke), drop the cached snapshot so the NEXT forUser() is fresh
// immediately instead of waiting out the ~30s TTL:
// $onelo->features->invalidateUser($user->id);   // pass nothing to clear every user

// Single-user processes (worker / CLI) may pin an identity once:
// $onelo->features->identify($jobUserId);        // identify(null) to clear

// ⚠️ Reading a plan-gated feature (upsell/greyed) through the GLOBAL path logs a
// one-time warning pointing you to forUser() — it evaluates the gate against the
// shared identity, not the request's user. Silence with
// ONELO_SUPPRESS_GATING_WARNING=1 if intentional (e.g. a single-user CLI teaser).

// Feature property checks: ->isEnabled() · ->isVisible() (fail-closed for unknown
// status) · ->isGreyed() · ->isNew() · ->isBeta() · ->isComingSoon() · ->isUpsell()
// · ->isKnown() · ->status
// Upsell metadata: ->upgradeHint() ("Pro") · ->requiredPlan · ->requiredPlanLabel
// · ->upgradeCta · ->reason
```
<!-- /onelo:snippet -->
