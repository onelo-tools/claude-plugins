# Node.js — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=node field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=node field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// route that's rarely hit can sit gated in your code and still never reach
// the dashboard Registry on a fresh deploy. declare() registers the whole
// list immediately, independent of traffic.
onelo.features.declare(['advanced-export', 'face-stream'])

// Recommended: block until the first snapshot lands so the very first request
// sees real state (not the 'hidden' fail-closed default). Returns a Promise<boolean>.
await onelo.features.ready(2000)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=node field=usage -->
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
// The SETTLED name is what goes in the call: onelo.features.feature('advanced-export')
// — and the SAME string is used at every point of the thread, on every platform.

// Global flag (no per-user targeting) — the common case. Synchronous read:
app.get('/export', (req, res) => {
  if (!onelo.features.feature('advanced-export').isEnabled) return res.status(404).end()
  // ...run the feature
})

// Per-user / per-plan gating — resolve for THIS request's user. STATELESS and
// multi-user-safe (no global identity, no race). Cached per-user ~30s; fail-closed
// (a network error resolves every feature to hidden, never throws):
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: enabled/new/beta → isEnabled true, buildReports() ·
// greyed/upsell/coming_soon → isEnabled false, upgradeHint set → locked JSON ·
// hidden → isEnabled false, upgradeHint null → 404. No upgradeHint where you
// expect one? THE SDK NEVER READS THE DRAFT — a status set in the dashboard
// Registry does nothing until Deploy is clicked. Log `feat` before assuming a
// bug; an undeployed status looks identical to one.
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
