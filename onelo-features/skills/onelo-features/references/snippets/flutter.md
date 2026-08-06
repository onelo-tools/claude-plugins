# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```dart
# pubspec.yaml
dependencies:
  onelo:
    git:
      url: https://github.com/onelo-tools/onelo-flutter
      ref: main
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// feature gated inside a conditionally-shown widget can sit gated in your
// code and still never reach the dashboard Registry on a fresh install.
// declare() registers the whole list immediately, independent of what
// happens to render.
onelo.features.declare(['advanced-export']);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```dart
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

// If you use your own auth system, call onelo.identify(userId) after login so
// per-user/per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id);

// SECURE MODE (recommended for paid apps): your backend signs the userId with
// the app secret_key — hash = HMAC-SHA256(secret_key, "user:" + userId) — so
// per-user targeting can't be spoofed with a raw id.
//   final hash = await myBackend.fetchOneloIdentity(currentUser.id);
//   await onelo.identify(currentUser.id, userIdHash: hash);

// ⚡ FIRST READ: for a RETURNING user, identify() restores THAT user's cached
// snapshot SYNCHRONOUSLY — feature() below is already correct with ZERO wait.
// Do NOT reflexively `await onelo.features.ready()` before every read "to be
// safe" — that blocks every launch on the network, cache or not, which is
// worse than the problem it solves. The flicker only exists for a genuinely
// NEW identity/device with no cached snapshot yet; if that matters for one
// specific widget, track a separate loading flag scoped to it and render a
// skeleton — never fold "not yet known" into hidden, and never block first
// paint on it.

// Read features synchronously anywhere. Updates arrive automatically in REAL
// TIME over SSE (Deploy / kill-switch / plan change), with a 60s poll +
// foreground/wake resync as fallback.
//
// ⚠️ THE SDK NEVER READS THE DRAFT — a status set in the dashboard Registry does
// nothing until Deploy is clicked. "I changed it and nothing happened" is almost
// always a missing Deploy click, not a code bug.
//
// ⚠️ Render by STATUS — do NOT gate visibility with isEnabled: it's true only
// for enabled/new/beta, so it HIDES greyed (locked) / coming_soon / upsell.
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: enabled → plain · new/beta → SAME plus badgeLabel
// ('New'/'Beta' — no extra branch needed) · greyed → 🔒 tap-to-upgrade ·
// upsell → 'Available in <plan>' · coming_soon → 'Coming Soon' · hidden →
// nothing (isVisible false, above). No badge on new/beta in your build? Print
// `f` right before this block before assuming a rendering bug — an
// undeployed dashboard status looks identical to one from here.
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
