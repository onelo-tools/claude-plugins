# PHP — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=php field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=php field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
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

// Optional: register known features upfront so they appear in the dashboard
// registry without waiting for code paths to execute.
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
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
<?php
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
