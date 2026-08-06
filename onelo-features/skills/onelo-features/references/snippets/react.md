# React

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=react field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=react field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-rendered component (an early-return,
// an unselected item) can sit gated in your code and still never reach the
// dashboard Registry on a fresh load. declare() registers the whole list
// immediately, independent of what happens to render.
onelo.features.declare(['advanced-export'])

export function useFeature(name: string) {
  // ⚡ FIRST READ: for a RETURNING user, this already reads THAT user's
  // cached snapshot from localStorage SYNCHRONOUSLY, right here — no wait, no
  // skeleton needed for the common case. Do NOT gate every component behind
  // ready() "to be safe"; that shows a skeleton on every load, cache or not,
  // which is worse than the rare flicker it prevents.
  const [state, setState] = useState(() => onelo.features.feature(name))
  // `ready` is for the ONE case the synchronous read above can't cover: a
  // genuinely NEW identity/device with no cached snapshot yet, where state
  // starts at the configured default (fail-closed 'hidden' unless changed).
  // Most components don't need this — see ExportButton below for when to.
  const [ready, setReady] = useState(false)

  useEffect(() => {
    let cancelled = false
    onelo.features.ready(1500).then(() => {
      if (cancelled) return
      setState(onelo.features.feature(name))
      setReady(true)
    })
    // Live-update: subscribe() fires on every Deploy (SSE), plan change or identity
    // swap. Re-read feature() so the component re-renders. Returns the unsubscribe.
    // ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
    // nothing until Deploy is clicked; subscribe() only fires for a DEPLOYED change.
    // "I changed it and nothing happened" is almost always a missing Deploy click.
    const unsubscribe = onelo.features.subscribe(() => setState(onelo.features.feature(name)))
    return () => { cancelled = true; unsubscribe() }
  }, [name])

  return { ...state, ready }
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=react field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// ─── STEP 0 — INVENTORY BEFORE YOU GATE (hard precondition) ──────────────────
// Do NOT write a single
//     useFeature('advanced-export')
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
// The SETTLED name is what goes in the call: useFeature('advanced-export')
// — and the SAME string is used at every point of the thread, on every platform.

// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// Usage in a component — render `feature` directly. For a returning user the
// initial read is already correct and synchronous; do NOT gate this on
// feature.ready by default (see the note below for the one case that does).
// Gate INTERACTIVITY on isEnabled; a blocked tile is tappable-to-UPGRADE only
// when feature.upgradeCta (dashboard "Tapping the feature opens the upgrade
// flow") + feature.requiredPlan — covers greyed, upsell AND coming_soon.
// onelo.openUpgrade() routes to the working web surface: subscriber →
// Customer Portal (Change plan); otherwise → Store. Needs a signed-in Onelo Auth user.
//
// SELF-CHECK — before you call this done, verify EVERY status renders what's
// expected (write it as a test against stubbed statuses, don't eyeball it):
//   enabled       → plain button, no badge
//   new / beta    → SAME as enabled but badgeLabel is 'New' / 'Beta' — this
//                   line already handles it: {feature.badgeLabel ? …}. Do NOT
//                   special-case isNew/isBeta with extra branches; badgeLabel
//                   is the SDK's ready-made label for both.
//   greyed        → 🔒, disabled unless upgradeCta, click → openUpgrade()
//   upsell        → 'Available in <plan>', same click behavior as greyed
//   coming_soon   → 'Coming Soon', same click behavior as greyed
//   hidden        → nothing (null, above)
//   disabled      → nothing (null, above)
// If new/beta showed no badge in your build: confirm badgeLabel is actually
// reaching this component (log `feature` right before the return) before
// assuming a rendering bug — a stale (undeployed) dashboard status looks
// IDENTICAL to a rendering bug from here. THE SDK NEVER READS THE DRAFT: a
// status set in the dashboard Registry does nothing until Deploy is clicked.
function ExportButton() {
  const feature = useFeature('advanced-export')
  // Only if a BRAND-NEW device/identity's brief default-status flash actually
  // matters for this one gate — most don't need this:
  //   if (!feature.ready) return <ExportButtonSkeleton />
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
