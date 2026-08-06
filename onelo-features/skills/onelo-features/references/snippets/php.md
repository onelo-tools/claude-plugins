# PHP — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=php field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=php field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

// Register EVERY feature name you gate, upfront — not optional. A feature()
// call only registers a name the first time that code path actually runs; a
// route that's rarely hit can sit gated in your code and still never reach
// the dashboard Registry on a fresh deploy. declare() registers the whole
// list immediately, independent of traffic.
$onelo->features->declare([
    // 'chat-stream', 'voice-stream',
]);

// Recommended: block until the first snapshot lands so the very first request
// sees real state (not the 'hidden' fail-closed default).
$onelo->features->ready(2.0);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=php field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
<?php
// ─── STEP 0 — INVENTORY BEFORE YOU GATE (hard precondition) ──────────────────
// Do NOT write a single
//     $onelo->features->isEnabled('advanced-export')
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
// The SETTLED name is what goes in the call: $onelo->features->isEnabled('advanced-export')
// — and the SAME string is used at every point of the thread, on every platform.

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
//
// SELF-CHECK before calling this done — verify EVERY status against stubbed
// data, don't eyeball it: enabled/new/beta → isEnabled() true, 'reports' view
// · greyed/upsell/coming_soon → isEnabled() false, upgradeHint() set →
// 'locked' view · hidden → isEnabled() false, upgradeHint() null → 'hidden'
// view. No upgradeHint() where you expect one? THE SDK NEVER READS THE
// DRAFT — a status set in the dashboard Registry does nothing until Deploy is
// clicked. Dump $feat before assuming a bug; an undeployed status looks
// identical to one.
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
