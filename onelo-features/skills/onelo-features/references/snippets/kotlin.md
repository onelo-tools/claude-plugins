# Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## init
<!-- onelo:snippet sdk=features lang=kotlin field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

## usage
<!-- onelo:snippet sdk=features lang=kotlin field=usage -->
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
//    a) ENTRY POINTS — every route / endpoint / RPC / GraphQL resolver a client can
//       reach, plus every CLI command, scheduled job and queue consumer. These are
//       inherently the right unit; the atom filter does not apply to them.
//    b) CALLERS — everything that invokes an entry point: a frontend fetch, a
//       webhook sender, another service, a cron entry. A caller is NOT its own
//       feature — it INHERITS the entry point's name. One feature = one entry-point
//       row + its linked caller rows, never two names.
//    c) CAPABILITIES — a unit of work with end-to-end logic that is not a route:
//       generate a report, run an export, sync a third party, send a digest.
//       Name the ACTION, not the function that happens to hold it.
//
// 2. QUALIFY. A feature is a unit you could plausibly SELL or GATE — not every
//    function. Route handlers, jobs and consumers: KEEP. Drop the plumbing:
//    middleware, serializers, DB models, migrations, config loaders, health checks,
//    client factories. Internal / debug / admin-only endpoints: ASK before gating —
//    do not gate them by default.
//
//    ⚠ THE EXCEPTION THAT PAYS — a candidate with NO user-facing call site.
//    The rule is that a capability normally needs at least one call site from the
//    UI, so pure plumbing — init paths, persistence, background sync nobody
//    triggers — must NOT become a feature. BUT:
//    on a backend most sellable behaviour has no UI at all,
//    so this is the NORM here, not the edge case.
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

// onelo-kotlin is a BACKEND / JVM SDK — isEnabled() gates your server logic.
// It is true for ENABLED/NEW/BETA and false for GREYED/COMING_SOON/UPSELL/HIDDEN/
// DISABLED, so ONE check denies every non-usable state.
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.
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
