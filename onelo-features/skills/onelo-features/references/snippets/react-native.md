# React Native

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-rendered screen can sit gated in your
// code and still never reach the dashboard Registry on a fresh install.
// declare() registers the whole list immediately, independent of what
// happens to render.
onelo.features.declare(['advanced-export'])

// Read a feature's state and STAY IN SYNC. FeatureState is a synchronous snapshot,
// and this hook re-renders automatically the instant the SDK cache changes — an
// admin's Deploy (pushed over the shared SSE connection), a foreground/identity
// resync, or invalidateCache(). subscribe() is purely local (no extra network, DB,
// or SSE) and returns an unsubscribe fn, which useEffect runs on unmount.
export function useFeature(name: string) {
  // ⚡ FIRST READ: for a RETURNING user, this already reads THAT user's
  // cached snapshot from storage SYNCHRONOUSLY, right here — no wait, no
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
    const unsubscribe = onelo.features.subscribe(() => setState(onelo.features.feature(name)))
    return () => { cancelled = true; unsubscribe() }
  }, [name])

  return { ...state, ready }
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=reactnative field=usage -->
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

// If you use your own auth system, call identify() after login so per-user /
// per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id)

// Feature states are available immediately — no init() needed. Updates push in
// real-time the moment an admin clicks Deploy in the dashboard; the SDK also
// resyncs on app-foreground and when the signed-in user's plan changes.
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell (upgrade) and coming-soon states you may have configured.
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: enabled → plain · new/beta → SAME plus badgeLabel
// ('New'/'Beta', already handled below — no extra branch needed) · greyed →
// 🔒 tap-to-upgrade · upsell → 'Available in <plan>' · coming_soon → 'Coming
// Soon' · hidden/disabled → nothing. No badge on new/beta in your build? Log
// `feature` right before the return before assuming a rendering bug — an
// undeployed dashboard status looks identical to one from here.
function ExportButton() {
  const feature = useFeature('advanced-export')
  // Only if a BRAND-NEW device/identity's brief default-status flash actually
  // matters for this one gate — most don't need this:
  //   if (!feature.ready) return <ExportButtonSkeleton />
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
