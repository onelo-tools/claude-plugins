# Swift — macOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## init
<!-- onelo:snippet sdk=features lang=macos field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-shown view can sit gated in your code
// and still never reach the dashboard Registry on a fresh launch. declare()
// registers the whole list immediately, independent of what happens to render.
onelo.features.declare(["advanced-export"])
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=macos field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// ─── STEP 0 — INVENTORY BEFORE YOU GATE (hard precondition) ──────────────────
// Do NOT write a single
//     onelo.features.feature("advanced-export")
// call until the table in step 4 exists and the developer has SEEN it.
// In the examples below the feature name is already chosen
// for you — choosing it is the actual job, and it is the step integrations skip.
// Gating whatever you happened to read produces a registry that LOOKS complete
// while most of the product stays ungated, unsellable and invisible in the
// dashboard.
//
// 1. ENUMERATE — three passes, in this order. Skip these paths entirely:
//       node_modules/  dist/  build/  out/  .next/  vendor/  __pycache__/
//       .build/  DerivedData/  xcuserdata/  Generated/
//       *.test.*  *.spec.*  *_test.*  test/  tests/
//    Skip anything already wrapped in a feature check a few lines above.
//    a) DESTINATIONS — every screen, window, tab, route, sheet or modal a user can
//       reach. This is the pass most integrations stop at.
//    b) TRIGGERS — everything that navigates to a destination: buttons, menu items,
//       links, deep links, keyboard shortcuts, command-palette entries. A trigger is
//       NOT its own feature — it INHERITS its destination's name. One feature = one
//       destination row + its linked trigger rows, never two names.
//    c) CAPABILITIES — a user-invoked action with end-to-end logic that is NOT a
//       screen: export a file, start a recording, run a sync. Name the ACTION, not
//       the widget that fires it.
//
// 2. QUALIFY — the atom filter. A feature is a unit you could plausibly SELL or
//    GATE, not every widget. DROP atoms: names ending Button, Cell, Row, Item,
//    Tile, Chip, Tag, Badge, Banner, Bar, Header, Footer, Icon, Field, Input,
//    Label, Spinner, Loader, Toast, Tooltip — and anything under /ui/, /atoms/,
//    /primitives/, /components/ui/, /design-system/. KEEP screen-shaped things:
//    names ending Screen, Page, Tab, Window, Sheet, Modal, Wizard, Flow, Activity,
//    Fragment, ViewController — and files under /screens/, /pages/, /routes/,
//    /flows/, /windows/, /onboarding/. Debug / Dev / Internal / Sandbox surfaces:
//    ASK before gating — do not gate them by default.
//
//    ⚠ THE EXCEPTION THAT PAYS — a candidate with NO user-facing call site.
//    The rule is that a capability normally needs at least one call site from the
//    UI, so pure plumbing — init paths, persistence, background sync nobody
//    triggers — must NOT become a feature. BUT:
//    some of the most sellable things in a product have no
//    screen and no button at all.
//    If a candidate changes product BEHAVIOUR — a config boolean, a branch on plan
//    or tier, a per-tenant toggle, a switchable integration (email notifications,
//    webhooks, remove-branding, custom domain, data export) — KEEP it, mark it
//    needs-confirmation, and ASK the developer. Silently filtering one out is the
//    expensive outcome: the developer never learns it was missed. Plumbing with no
//    behavioural effect still gets dropped — that distinction IS the qualify step.
//
// 3. THREAD IT ACROSS THE STACK, THEN SETTLE ONE NAME. A feature rarely lives in
//    one place: a trigger, the destination it opens, and the backend handler where
//    it actually ends are ONE feature. Group them into one thread, e.g.
//        feedback
//          ├─ client   ReportBugButton   (trigger)
//          ├─ client   FeedbackSheet     (destination)
//          └─ backend  routes/feedback   (handler — where it ends)
//    A scan sees an HTTP call, not which endpoint answers it — ASK when the link is
//    not obvious, and show the thread you believe in so it can be corrected.
//
//    ⚠ Onelo keys the registry BY NAME. The same name declared from two platforms
//    becomes ONE entry tagged with both; two DIFFERENT names silently create two
//    features and the tagging never happens. So: one feature = one name, at every
//    point of the thread, on every platform. Never rename between platforms to
//    "make it clearer".
//
//    Then, on every PAID feature, ask: "is there a part of this you want to sell
//    separately?" A sub-feature takes the parent's name plus a hyphen
//    (feedback → feedback-bug). Developers rarely volunteer these, and that is
//    exactly where the upsell lives.
//
// 4. WRITE THE TABLE — before you write any code. One row per feature:
//      feature | file:line | proposed name | destination/trigger/capability | gated / skipped + why
//    Show it to the developer and WAIT for approval. Group by feature, not by file,
//    with sub-features nested under their parent and every needs-confirmation
//    candidate called out.
//
// 5. THEN implement — one THREAD at a time, never one file at a time, and keep the
//    table in the PR description. Gating half a thread is worse than not gating it:
//    the UI hides while the handler stays open, so the feature is off for honest
//    users and on for anyone who knows the URL.
//
// ⚠️ REGISTRATION IS NOT OPTIONAL — call declare() with every name in the table,
//    once, at startup (see "Upfront declaration" below for the exact call). A
//    feature() call only registers a name the FIRST TIME that code path actually
//    RUNS — a feature gated inside a conditionally-rendered component (an
//    early-return, an unselected item, a rarely-hit branch) can sit in your table
//    as "gated" and still never reach the dashboard Registry on a fresh install,
//    because nothing ever called feature() for it. declare() registers the whole
//    table immediately, independent of what happens to render. Do this for EVERY
//    thread you implement in this pass, not just ones you notice are conditional
//    — you cannot always tell from the table alone which destinations are
//    reachable on first load.
//
// COVERAGE RULE: every candidate you enumerated is either gated, or present in the
// table with a stated reason for the skip. "I did not get to it" is not a reason.
// A deliberate skip and an unexamined miss must never look the same in your report.
//
// ─── NAMING CONVENTION ───────────────────────────────────────────────────────
// kebab-case, lowercase, [a-z0-9-] only, max 48 chars. Name the ACTION or the
// DESTINATION — never the widget that happens to open it.
//   Good:  advanced-export · analytics-dashboard · export-recording · settings-window
//   Bad:   export-button · ExportButton · exportBtn · advanced_export · screen2
// Convert PascalCase / camelCase to kebab-case, then strip a trailing suffix:
//   View, Screen, Page, Activity, Fragment, ViewController, ViewModel, Handler,
//   Controller, Service, Widget.
//   AnalyticsDashboardView → analytics-dashboard · ExportHandler → export
// Collision? append -2, -3. Sub-feature? parent name + hyphen: feedback-bug.
// The SETTLED name is what goes in the call: onelo.features.feature("advanced-export")
// — and the SAME string is used at every point of the thread, on every platform.

// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// ── IDENTIFY ──────────────────────────────────────────────
await onelo.identify(currentUser.id)

// ⚡ FIRST READ: for a RETURNING user, identify() restores THAT user's cached
// snapshot SYNCHRONOUSLY — feature() below is already correct with ZERO wait.
// Do NOT reflexively `await onelo.features.ready()` before every read "to be
// safe" — that blocks every launch on the network, cache or not, which is
// worse than the problem it solves. The flicker only exists for a genuinely
// NEW identity/device with no cached snapshot yet; if that matters for one
// specific view, track a separate `@State private var ready = false` scoped
// to it and render a skeleton — never fold "not yet known" into .hidden, and
// never block app launch on it.

// Check features synchronously — no init() needed.
// Updates push in real-time the moment an admin clicks Deploy in the Onelo dashboard.
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.
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
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: .enabled → plain · .new/.beta → SAME plus
// badgeLabel ('New'/'Beta', already handled below — no extra branch needed) ·
// .greyed → 🔒 tap-to-upgrade · .upsell → 'Available in <plan>' · .comingSoon
// → 'Coming Soon' · .hidden/.disabled → nothing. No badge on new/beta in your
// build? Print `f` right before this block before assuming a rendering bug —
// an undeployed dashboard status looks identical to one from here (see "THE
// SDK NEVER READS THE DRAFT" above).
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
