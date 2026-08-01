# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
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
<!-- onelo:snippet sdk=monitor lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
/*
 * PLACEMENT: Create ONE Onelo instance in main(), before runApp().
 * Pass it down via InheritedWidget, Provider, or a global singleton.
 * Never create inside a Widget build() method.
 *
 * GLOBAL ERROR CAPTURE: Installed AUTOMATICALLY at init — Onelo hooks
 * FlutterError.onError and PlatformDispatcher.instance.onError to capture
 * framework and async errors as 'global_error' events. It chains any handler
 * already installed. If you set your OWN FlutterError.onError AFTER creating
 * Onelo (e.g. a crash reporter set up later), that replaces Onelo's hook — call
 * onelo.monitor.registerGlobalHandlers() again and it re-wraps so BOTH run
 * (a no-op if nothing changed, so it's always safe to call).
 *
 * AUTO-ATTACHED ON ERRORS: stack trace, error type, recent breadcrumbs,
 * active feature flags, and device context are attached to every error event.
 * Sensitive values (tokens, passwords, card numbers) are redacted ON-DEVICE
 * before the event leaves the app.
 *
 * IDENTITY: When you use Onelo Auth, the signed-in user is attached to monitor
 * events automatically. With your own auth, call onelo.monitor.setUserId(...).
 *
 * LIFECYCLE: The SDK buffers events and flushes every 15 seconds. Errors flush
 * immediately. Flutter has no reliable app-quit hook, so the SDK relies on the
 * periodic flush — call onelo.monitor.flush() before a known risky moment.
 *
 * HOW IT WORKS: Events are buffered in memory (max 200, oldest dropped on
 * overflow) and sent in a single HTTP batch. Failed batches are dropped — NOT
 * retried or persisted in this version. Session ID is install-bound and
 * persists across launches (stored in secure storage; on iOS that is
 * Keychain-backed, so it can survive a reinstall).
 */
import 'package:onelo/onelo.dart';

void main() {
  final onelo = Onelo(
    publishableKey: 'onelo_pk_live_YOUR_KEY',
    apiUrl: 'https://api.onelo.tools',
    callbackScheme: 'myapp',
    environment: 'production', // optional — surfaces as the "Environment" column
  );

  runApp(MyApp(onelo: onelo));
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=monitor lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// IDENTIFY YOUR USER
// If using Onelo Auth — user_id is set automatically, nothing to do.
// If using your own auth system:
// onelo.monitor.setUserId('user-123');  // after login
// onelo.monitor.setUserId(null);         // after logout

// AUTO-TRACK: wrap any async operation. track() re-throws — use try/catch.
// Signature: track(name, fn, {meta}) — the operation is the 2nd positional arg.
try {
  final result = await onelo.monitor.track('checkout', () => processPayment(), meta: {
    'plan': user.plan,          // correlate errors with plan types
    'itemCount': cart.length,   // what they were doing
    'method': 'stripe',         // which provider / path
  });
} catch (e) {
  // error is already recorded in Onelo — handle UI feedback here
}

// INSTANTANEOUS EVENT: use event() only for things with no duration.
onelo.monitor.event('tab_viewed', MonitorEventOptions(
  ok: true,
  meta: {'tab': 'export'},
));

// ERROR WITHOUT track()
onelo.monitor.event('ai_stream', MonitorEventOptions(
  ok: false,
  error: 'timeout',
  meta: {'model': 'gpt-4', 'failedAt': 'stream_start'},
));

// MANUAL CAPTURE: record a caught exception with its type + stack.
try {
  riskyThing();
} catch (e, stack) {
  onelo.monitor.capture(e, featureName: 'import', stack: stack, meta: {'source': 'csv'});
}

// BREADCRUMBS: leave a trail that gets attached to the NEXT error, so a
// failure carries the recent activity that led to it.
onelo.monitor.breadcrumbNavigation('checkout');
onelo.monitor.breadcrumbInfo('applied coupon SPRING');

// FEATURE FLAGS: feature() calls are tracked automatically — no extra code
// needed. The active flag values are auto-attached to error events.

// ─── NAMING CONVENTION ───────────────────────────────────────────────────────
// Use snake_case: {object}_{past_tense_verb}
// Object  → what the user acts on: checkout, pdf_export, ai_response, sync
// Verb    → past tense, result-oriented: completed, failed, started, retried
//
// Good:  checkout  pdf_export  ai_response  onboarding  sync  tab_viewed
// Bad:   checkout_completed  export_error  doSync  exportClicked
//
// Name the FEATURE, not the outcome — track() records ok/error/duration automatically.
// Use _verb suffix only for event() calls (instantaneous, no duration):
//   tab_viewed  button_clicked  plan_selected  modal_dismissed
//
// Avoid _started / _completed suffixes — use track() to wrap the operation instead.
//
// PITFALL — check-style events: don't name an event after its failure mode.
// The name shows up in Feature Health even on success, so failure-mode names
// read like permanent alerts.
//   ✗ permission_missing, binary_missing, network_unavailable
//   ✓ permission_check, embedded_binary_check, network_check
//     with ok: false, error: "denied" | "not_in_bundle" | "timeout"

// ─── GRANULARITY — YOU DECIDE ────────────────────────────────────────────────
// You control how detailed the tracking is. There is no "correct" level —
// choose based on what questions you need to answer in Feature Health.
//
// USE track() FOR OPERATIONS — it wraps async work, measures time automatically,
// and sends ONE event covering the whole operation:
//
//   await onelo.monitor.track('pdf_export', () async {
//     await generateAndUploadPdf(doc);
//   }, meta: {'pages': doc.pages, 'format': 'a4'})
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     track() re-throws errors so your app handles them normally.
//     Always wrap in try/catch.
//
// USE event() FOR INSTANTANEOUS THINGS — a click, a tab switch, a state change:
//
//   onelo.monitor.event('tab_viewed', MonitorEventOptions(ok: true, meta: {'tab': 'export'}))
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead:
//
//   ✗ event('export_started', ...)   then   event('export_completed', ...)
//   ✓ track('export', () async { /* entire export operation */ }, meta: {...})
//
// FINE-GRAINED — if one operation has multiple steps you need to diagnose
// individually, use track() for each step with descriptive names:
//
//   await onelo.monitor.track('pdf_render', () => render(doc), meta: {'pages': pages})
//   await onelo.monitor.track('pdf_upload', () => upload(file), meta: {'size': size})
//
//   → Two rows, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   meta: {'plan': 'pro', 'pages': 42, 'format': 'pdf'}
//   → filter sessions by those values to debug "only fails for free-plan users".

// ─── SPECIAL META KEYS (opt-in — set these to get dashboard columns) ─────────
// These keys, when present in meta, surface as dedicated columns + filters
// in Feature Health. Set them yourself in your event meta — use these EXACT
// names so the dashboard auto-categorizes:
//   meta.model        → "Model" column     (e.g., "gpt-4", "claude-opus-4-7")
//   meta.trigger      → "Trigger" column   (e.g., "ptt", "vad", "scheduled")
//   meta.environment  → "Environment"      ("production" | "staging" | "dev")
// Custom keys still work — they just don't get column / filter status.
// "Release" and "App build" are auto-filled from app.version / app.build below
// — you do NOT set them manually.
//
// ─── WHAT SDK AUTO-ATTACHES (do NOT add these to meta manually) ──────────────
//   session_id  → install-bound id, persists across launches (groups events)
//   user_id     → from setUserId() or Onelo Auth (if used)
//   platform    → 'flutter' (constant for this SDK)
//   meta.app    → { version, build, bundleId } — Release + App build columns
//   meta.device → { os, osVersion, locale, timezone } (+ model on errors)
//   on errors   → stack, errorType, recent breadcrumbs, active flags auto-attached
//   timestamp   → UTC, set at event creation time
//   meta.sdk    → { name, version } — the SDK's OWN version, shown as the
//                 "SDK version" column in Feature Health. Auto — you never set
//                 it. (A blank column means the event came from a build that
//                 predates this, or from a non-SDK source.)

// ─── WHAT NOT TO INSTRUMENT ──────────────────────────────────────────────────
//   ✗ Mouse move, hover, scroll position (too frequent, no actionable signal)
//   ✗ Every keystroke in an input field (track _submitted instead)
//   ✗ Internal UI state toggles with no outcome (e.g. dropdown open/close)
//   ✗ Events faster than ~1 per second per user (aggregate on your side first)
//   ✗ PII in meta — no emails, full names, passwords, or card numbers
```
<!-- /onelo:snippet -->
