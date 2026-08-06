# PHP — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=php field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=php field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
<?php
// PLACEMENT: create ONE Onelo client and call $onelo->monitor->init() ONCE at
// startup — in a Laravel service provider, a bootstrap file, or your
// front-controller before it handles the request.
//
// PER-REQUEST ISOLATION: under PHP-FPM every request is its OWN process, so the
// monitor scope (user / breadcrumbs / tags) is naturally request-isolated — no
// middleware needed. In a PERSISTENT runtime (Laravel Octane, Swoole,
// RoadRunner, a queue worker) the process is reused across requests, so call
// $onelo->monitor->resetScope() at the start of each request / job.
//
// DELIVERY: PHP has no background thread, so events buffer in memory and are
// sent in ONE HTTP batch at request end via register_shutdown_function. In a
// long-running worker, call $onelo->monitor->flush() when each job finishes.
//
// DELIVERY IS RETRIED: a batch that fails on the network, or gets a 5xx, is
// RETRIED up to 3 times with blocking exponential backoff (500ms → 1s → 2s);
// a 429 is not retried but sets a Retry-After hold-off, and other 4xx are
// terminal. The buffer is emptied before the send, so a batch that exhausts
// its 3 attempts is dropped and written to error_log — never re-queued.
// Feature Health is a strong signal, not an audit log.
//
// CRASH CAPTURE: init() installs a global exception handler + a fatal-error
// shutdown check, so uncaught exceptions AND fatal errors are reported even if
// no framework handler ran. It chains to any previously-set exception handler.
//
// PRIVACY: sensitive headers (Authorization, Cookie, X-API-Key) and query
// params (token, key, secret, password, ...) are scrubbed before any data
// leaves your process — same regex set as the Swift / Python / Node SDK.

use Onelo\Onelo;

// ─── ENVIRONMENT VARIABLES — set these BEFORE starting your app ──────────────
// This is a BACKEND (server-side) integration, so it authenticates with your
// SERVER SECRET key — NOT a publishable key. Create it in the Onelo dashboard →
// API Keys → Secret keys and read it from the environment (never hard-code it):
//
//   ONELO_SECRET_KEY   REQUIRED   "onelo_sk_live_..."  (server-side only)
//   ONELO_API_URL      REQUIRED   Onelo API base URL → "https://api.onelo.tools"
//
// (Monitor has no test/live split — that's a Features-only concept. Monitor is
// a single event stream.)
$onelo = new Onelo(
    secretKey: getenv('ONELO_SECRET_KEY') ?: throw new \RuntimeException('ONELO_SECRET_KEY is required'),
    apiUrl: getenv('ONELO_API_URL') ?: 'https://api.onelo.tools',
);

$onelo->monitor->init([
    'release' => getenv('ONELO_RELEASE') ?: null,     // your build id — reads ONELO_RELEASE (else GIT_COMMIT_SHA)
    'environment' => getenv('APP_ENV') ?: 'production', // production / staging / local (else ONELO_ENVIRONMENT / ENVIRONMENT / APP_ENV)
    // AUTO-CAPTURE (opt-in) — instrument the app without wrapping every call:
    'captureErrors' => true,   // PHP warnings/notices → breadcrumbs (honours error_reporting); uncaught exceptions & fatal errors are ALWAYS captured as events
    // (Source lines for in-app exception frames are attached automatically.)
    // High-throughput backend? Sample client-side BEFORE the network so an error
    // storm can't thrash the buffer or trip the hourly quota. Default 1.0 = keep all.
    // 'sampleRate' => 0.25,          // keep-probability for ERROR events (ok=false)
    // 'successSampleRate' => 0.05,   // keep-probability for track / success events
]);

// ─── REQUEST SCOPE + AUTO EXCEPTION CAPTURE (recommended) ───────────────────
// Mount the monitor middleware ONCE and every request gets: a fresh isolated
// scope (resetScope), request context (method + scrubbed URL & headers), and
// automatic capture of any exception that escapes your handler — re-thrown so
// your normal error page still renders. It's also what makes user attribution
// safe under a PERSISTENT runtime (Laravel Octane / Swoole / RoadRunner): the
// middleware resets the scope per request, so an auth->monitor bridge can't leak
// a user across requests. Under plain PHP-FPM it's optional (each request is its
// own process) but still adds request context.
//
// Laravel — register the client as a singleton, then add the middleware:
//   $this->app->singleton(Onelo::class, fn () => $onelo);          // service provider
//   // app/Http/Kernel.php — prepend to the global $middleware array:
//   \Onelo\Laravel\OneloMonitor::class,
//
// PSR-15 (Slim / Mezzio / any PSR-15 stack) — add to the pipeline FIRST:
//   $app->add(new \Onelo\Psr15\OneloMonitorMiddleware($onelo));
//
// No framework / a queue worker / long-running job — reset the scope yourself
// at the top of each unit of work:
//   $onelo->monitor->resetScope();

// GRACEFUL SHUTDOWN is automatic under PHP-FPM (the shutdown drain). In a
// long-running worker, flush before the process stops:
//   $onelo->monitor->close();   // flushes any remaining buffered events
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=monitor lang=php field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
<?php
// ─── STEP 0 — INVENTORY BEFORE YOU INSTRUMENT (do this first) ────────────────
// Do NOT write a single monitor->track() / captureMessage() call until you have listed what
// this service actually does. Instrumenting the handlers you happened to read
// produces a dashboard that looks healthy because the broken half is not being
// measured.
//
// 1. ENUMERATE. Walk the codebase and list every unit of work the service
//    performs end-to-end — anything with an OUTCOME that can succeed or fail:
//      • every HTTP route / endpoint (including webhooks and callbacks), grouped
//        by the business operation it performs, not by URL shape
//      • every background job, queue consumer, scheduled/cron task
//      • every outbound third-party integration: payment provider, mail, storage,
//        search, LLM provider, any other API you do not control
//      • every boundary crossing inside the process: database, cache, filesystem,
//        subprocess
//      • startup, migration and shutdown paths (config load, schema migration,
//        warm-up, graceful drain)
//
// 2. FIND THE SILENT FAILURES. Grep for swallowed errors: an empty except/catch,
//    a handler returning an empty list or a default on failure, a fallback value
//    that hides a failed read, a fire-and-forget task nobody awaits, and any
//    place that logs and continues. These are the highest-value events in any
//    backend — they are exactly the failures that reach the caller and reach
//    nobody else. Instrument them FIRST.
//
// 3. WRITE THE TABLE. Before coding, output one row per unit of work:
//      operation | file | event name | monitor->track() or captureMessage() | covered / skipped + why
//    Show it to the developer. A skipped operation must say WHY it was skipped
//    (see WHAT NOT TO INSTRUMENT below) — an unmeasured operation and a
//    deliberately-excluded one must never look the same in your report.
//
// 4. THEN implement, and keep the table in the PR description or a comment.
//
// 5. VERIFY. After implementing, actually trigger one instrumented operation,
//    wait ~15s (the flush interval — see below), reload the Onelo dashboard's
//    Feature Health tab and confirm a row appears for it. "The code compiles
//    and calls monitor->track()" is not verification; a row on the dashboard is.
//
// THE GATING TRAP — if auth middleware or a feature flag stands in front of
// EVERY route you instrumented, a healthy-but-locked-out deployment (bad
// credentials, a misconfigured flag) still shows zero events, indistinguishable
// from "not integrated." Make sure at least one operation you instrument sits
// OUTSIDE any such gate — a health-check route, the auth middleware's own
// accept/reject outcome, or process startup itself.
//
// COVERAGE RULE: every operation in the table is either instrumented, or listed
// with a stated reason. "I did not get to it" is not a reason.

// IDENTIFY THE USER on each request (Onelo Auth populates this for you; call
// manually only if you use your own auth):
$onelo->monitor->setUser(['id' => $user->id, 'email' => $user->email]);

// TRACK AN OPERATION — measures wall-clock time and emits ONE event: ok=true on
// success, or an error event (full stack + breadcrumbs + active feature flags)
// if the closure throws. The exception still propagates, so your control flow is
// unchanged — and the closure's return value is passed back to you:
$intent = $onelo->monitor->track('checkout', fn () => processPayment($cart), [
    'plan' => $user->plan,   // meta — queryable in Feature Health
]);

// Name the FEATURE, not the outcome — track() records ok/error/duration for you.
// Prefer track() for operations; use captureException() only for errors you
// catch OUTSIDE a track() block.

// CAPTURE AN EXCEPTION you catch outside a track() block:
try {
    risky();
} catch (\Throwable $e) {
    $onelo->monitor->captureException($e, 'payment', ['plan' => $user->plan]);
}

// INSTANTANEOUS EVENT / MESSAGE (no duration, just a marker):
$onelo->monitor->captureMessage('rate-limit triggered', 'warning', 'rate_limit');

// BREADCRUMBS: trail of context attached to the next captured event
// (up to 100 most-recent per request):
$onelo->monitor->addBreadcrumb('loaded user', 'db');
$onelo->monitor->addBreadcrumb('called Stripe', 'http');

// TAGS for slicing Feature Health in the dashboard:
$onelo->monitor->setTag('region', 'eu');

// CROSS-PROCESS JOBS (Laravel queue / a separate worker) — a queued job runs
// OUTSIDE the request, so its scope is empty. Capture a carrier when you dispatch,
// then restore it in the worker so errors there carry the request's user + tags.
// Pass your publishable key to HMAC-sign it (the worker verifies before trusting):
//   Producer (request side) — stash the carrier on the job:
SendInvoice::dispatch($payload, $onelo->monitor->carrier(key: $onelo->key));
//   Worker (inside Job::handle) — restore it around the work:
$onelo->monitor->continueTrace($this->carrier, function () {
    $this->sendInvoice();   // an error here is attributed to the dispatching user
}, key: $onelo->key);

// FEATURE FLAG CORRELATION (auto): every captured event ships the active flag
// snapshot, so the dashboard can show "this error spiked when flag X turned on"
// — no extra wiring on your side.

// ─── NAMING CONVENTION ───────────────────────────────────────────────────────
// Use snake_case: {object}_{past_tense_verb}
// Object  → what the user acts on: checkout, pdf_export, ai_response, sync
// Verb    → past tense, result-oriented: completed, failed, started, retried
//
// Good:  checkout  pdf_export  ai_response  onboarding  sync  tab_viewed
// Bad:   checkout_completed  export_error  doSync  exportClicked
//
// Name the FEATURE, not the outcome — track() records ok/error/duration automatically.
// Use _verb suffix only for captureMessage() calls (instantaneous, no duration):
//   tab_viewed  button_clicked  plan_selected  modal_dismissed
//
// Avoid _started / _completed suffixes — use track() to wrap the operation instead.
//
// PITFALL — check-style events: don't name an event after its failure mode.
// The name shows up in Feature Health even on success, so failure-mode names
// read like permanent alerts.
//   ✗ permission_missing, binary_missing, network_unavailable
//   ✓ permission_check, embedded_binary_check, network_check
//     with ok: false, error: "denied" | "not_found" | "timeout"

// ─── GRANULARITY (depth is yours — coverage is not) ──────────────────────────
// You choose how DEEP to go on a given route or job: one event for the whole
// request, or one per diagnosable step. You do NOT choose which routes and jobs
// get measured — that is settled by the STEP 0 inventory above.
//
// USE track() FOR OPERATIONS — it wraps the work, measures wall-clock time, and
// emits ONE event covering the whole operation:
//
//   $onelo->monitor->track('invoice_issue', fn () => issueInvoice($order), [
//       'provider' => 'stripe',
//   ]);
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     The exception still propagates, so your control flow is unchanged.
//
// USE captureMessage() FOR INSTANTANEOUS THINGS — a marker with no duration:
// a rate-limit trip, a config reload, a feature toggled off by a kill switch:
//
//   $onelo->monitor->captureMessage('rate-limit triggered', 'warning', 'rate_limit');
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead.
//
// FINE-GRAINED — if one request has several steps you need to diagnose
// individually (DB query, third-party call, PDF render), wrap each in its own
// track() with a descriptive name:
//
//   $pdf = $onelo->monitor->track('invoice_render', fn () => renderInvoice($order));
//   $onelo->monitor->track('invoice_upload', fn () => upload($pdf), ['size' => strlen($pdf)]);
//
//   → One row per step, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   ['plan' => 'pro', 'region' => 'eu', 'provider' => 'stripe']
//   → filter sessions by those values to debug "only fails for free-plan users".

// ─── WHEN "RESOLVED" IS NOT "SUCCEEDED" ──────────────────────────────────────
// track() sets ok:false only when the closure THROWS. An operation that
// returns empty-handed — null session, zero rows, cancelled picker — is still
// recorded as a success. If "no result" means the feature failed for your user,
// raise it inside the closure and handle it outside:
//
//   try {
//       $rows = $onelo->monitor->track('library_load', function () use ($tenantId) {
//           $found = loadLibrary($tenantId);
//           if ($found === []) { throw new \RuntimeException('empty_result'); }
//           return $found;
//       }, ['source' => 'db']);
//   } catch (\Throwable $e) {
//       // already recorded as an error event with its duration — return 404, not 200 []
//   }
//
// One truthful event per attempt, with timing on the failure path too. Do NOT
// emit a second captureMessage() for the failure — that splits one feature into two rows.

// ─── WHAT SDK AUTO-ATTACHES ─────────────────────────────────────────────────
//   userId                → from $onelo->monitor->setUser(...)
//   sdk / app / flags      → SDK version + active feature-flag snapshot
//   stack / frames         → on every captured exception (in_app classifier)
//   breadcrumbs            → up to 100 most recent
//   release / environment  → from monitor->init(...)
//   sessionId              → per process (groups one worker/pod's events)

// ─── WHAT NOT TO INSTRUMENT ─────────────────────────────────────────────────
//   ✗ Health checks (/healthz) — not failures, no signal
//   ✗ Validation errors the framework already returns as 4xx — usually expected
//   ✗ PII in meta — secrets are scrubbed by the SDK, but better not to send them
```
<!-- /onelo:snippet -->
