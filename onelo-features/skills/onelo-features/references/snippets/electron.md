# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=electron field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=electron field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-rendered window/menu can sit gated in
// your code and still never reach the dashboard Registry on a fresh launch.
// declare() registers the whole list immediately, independent of what happens
// to render.
onelo.features.declare(['advanced-export'])
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
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
// snapshot from disk SYNCHRONOUSLY — feature() below is already correct with
// ZERO wait. Do NOT reflexively `await onelo.features.ready()` before every
// read "to be safe" — that blocks every app launch on the network, cache or
// not, which is worse than the problem it solves. The flicker only exists for
// a genuinely NEW identity/device with no cached snapshot yet; if that matters
// for one specific IPC push, forward a separate loading flag to the renderer
// instead of blocking main-process startup on it.

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
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.
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
