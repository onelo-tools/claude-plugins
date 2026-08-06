# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=android field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=android field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-shown Composable/Fragment can sit
// gated in your code and still never reach the dashboard Registry on a fresh
// install. declare() registers the whole list immediately, independent of
// what happens to render.
onelo.features.declare(listOf("advanced-export"))
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=android field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```kotlin
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

// ── IDENTIFY (skip if you use Onelo Auth — automatic). identify() is suspend. ──
lifecycleScope.launch { onelo.identify(currentUser.id) }

// ⚡ FIRST READ: for a RETURNING user, identify() restores THAT user's cached
// snapshot SYNCHRONOUSLY — feature() below is already correct with ZERO wait.
// Do NOT reflexively suspend on `onelo.features.ready()` before every read
// "to be safe" — that blocks every launch on the network, cache or not, which
// is worse than the problem it solves. The flicker only exists for a
// genuinely NEW identity/device with no cached snapshot yet; if that matters
// for one specific Composable, track a separate loading flag scoped to it and
// render a skeleton — never fold "not yet known" into hidden, and never block
// app launch on it.

// Feature state is available synchronously via feature(name). Updates push in REAL TIME
// the moment an admin clicks Deploy (SSE); the SDK AUTO-refreshes when the app returns to
// the FOREGROUND and re-resolves when the signed-in user's plan changes — newly-unlocked
// features appear with NO restart. In Compose, collect onelo.features.updates to recompose.
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.

// ── CORRECT rendering pattern (Jetpack Compose) — copy this shape ────────────
// Two rules: (1) HIDDEN or DISABLED → render NOTHING; (2) otherwise render, and let
// f.badgeLabel show the padlock / hint. A GATED feature — GREYED (dashboard "Locked",
// shows 🔒) or UPSELL (shows "Available in <plan>") — becomes tappable → the upgrade flow
// ONLY when the dashboard's "Tapping the feature opens the upgrade flow" toggle is on.
// That toggle is f.upgradeCta; the backend sends it for greyed, upsell AND coming_soon
// (any plan-gated status with a requiredPlan) — so gate the tap on upgradeCta + requiredPlan.
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: ENABLED → plain · NEW/BETA → SAME plus badgeLabel
// ('New'/'Beta', already handled below — no extra branch needed) · GREYED →
// 🔒 tap-to-upgrade · UPSELL → 'Available in <plan>' · COMING_SOON → 'Coming
// Soon' · HIDDEN/DISABLED → nothing. No badge on new/beta in your build? Log
// `f` right before this Composable renders before assuming a rendering bug —
// an undeployed dashboard status looks identical to one from here.
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
