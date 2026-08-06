# Web — HTML embed and JavaScript / TypeScript

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

> One snippet covers Web — HTML embed and Web — JavaScript / TypeScript — the SDK code is identical.

## install
<!-- onelo:snippet sdk=features lang=web field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```html
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=web field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-rendered component (an early-return,
// an unselected item) can sit gated in your code and still never reach the
// dashboard Registry on a fresh load. declare() registers the whole list
// immediately, independent of what happens to render.
onelo.features.declare(['advanced-export'])
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=web field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```html
// ─── STEP 0 — INVENTORY BEFORE YOU GATE (hard precondition) ──────────────────
// Do NOT write a single
//     onelo.features.feature('advanced-export')
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
// The SETTLED name is what goes in the call: onelo.features.feature('advanced-export')
// — and the SAME string is used at every point of the thread, on every platform.

// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// ⚡ FIRST READ: for a RETURNING user, load() restores THAT user's cached
// snapshot from localStorage SYNCHRONOUSLY — feature() below is already
// correct with ZERO wait. Do NOT reflexively `await onelo.features.ready()`
// before every read "to be safe" — that blocks first paint on the network on
// EVERY load, cache or not, which is worse than the problem it solves.
//
// The flicker only exists for a genuinely NEW identity/device with no cached
// snapshot yet — there, feature() reads the configured default (fail-closed
// 'hidden' unless you changed it) until the network resolves. If that default
// flash matters for a specific gate, track a separate loading flag scoped to
// THAT component and render a skeleton — never fold "not yet known" into the
// same branch as .hidden, and never block the whole app on it.
//
// onelo.features.ready(timeoutMs) exists for the rare case you need a
// CONFIRMED (not just cached) status before one specific decision — e.g. a
// paywall check right before charging. It is not a first-paint gate.

// REACTIVITY — feature states are available immediately (no init() needed) and
// update in real-time the moment an admin clicks Deploy. subscribe() fires on every
// change (a Deploy over SSE, a plan change, an identity swap) — re-read feature()
// and re-render (React: wrap in useSyncExternalStore; Vue: bump a ref). Returns an
// unsubscribe fn; call it on teardown.
//
// ⚠️ THE SDK NEVER READS THE DRAFT. Setting a status in the dashboard Registry
// (Beta, New, plan-gate, …) does nothing here until an admin clicks Deploy —
// the client only ever sees the last DEPLOYED snapshot. "I changed it and
// nothing happened" is almost always a missing Deploy click, not a code bug —
// check that before debugging the read side.
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
