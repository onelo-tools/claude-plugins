# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=electron field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=electron field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
/*
 * PLACEMENT: Create ONE Onelo instance in your main process (main.ts).
 * Never instantiate in renderer or preload — the SDK runs in Node.js only.
 * If you need to trigger events from the renderer, expose a thin IPC handler
 * in preload (contextBridge) that calls onelo.monitor.event() in main.
 *
 * LIFECYCLE: Call onelo.destroy() when the app is about to quit so any
 * remaining buffered events are flushed before the process exits:
 *   app.on('before-quit', async (e) => { e.preventDefault(); await onelo.destroy(); app.exit(0) })
 *
 * HOW IT WORKS: Events are buffered in memory and sent to Onelo in a single
 * HTTP batch every 15 seconds. Errors (ok: false) flush immediately.
 * Unhandled EXCEPTIONS are captured automatically as 'global_error' events —
 * no extra code needed. Unhandled PROMISE REJECTIONS are intentionally NOT
 * auto-captured on Electron: registering a Node 'unhandledRejection' listener
 * would suppress the process's default crash-on-rejection, and a monitoring SDK
 * must never silently change your app's termination policy. Wrap risky async
 * work in onelo.monitor.track() to record its rejections instead.
 *
 * SINGLETON: Export the onelo instance from a shared module (e.g. onelo.ts)
 * and import it wherever needed. Never create more than one instance.
 */
import { Onelo } from '@onelo/electron'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // your app id — REQUIRED for codesign attestation (a signed build 403s without it)
})

export default onelo
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=monitor lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// IDENTIFY YOUR USER
// If using Onelo Auth — user_id is set automatically, nothing to do.
// If using your own auth system, call setUserId() after login/logout:
// onelo.monitor.setUserId('user-123')  // after login
// onelo.monitor.setUserId(null)         // after logout

// AUTO-TRACK: wrap any async operation to measure time and capture errors.
// track() re-throws the error so your app can handle it normally.
// Always wrap in try/catch — the event is recorded either way.
try {
  const result = await onelo.monitor.track('checkout', () => processPayment(), {
    meta: {
      plan: user.plan,          // correlate errors with plan types
      itemCount: cart.length,   // what they were doing
      method: 'stripe',         // which provider / path
    }
  })
} catch (err) {
  // error is already recorded in Onelo — handle UI feedback here
}

// INSTANTANEOUS EVENT: use event() only for things with no duration —
// a tab switch, a button click, a state change. NOT for operations with start/end.
onelo.monitor.event('tab_viewed', {
  ok: true,
  meta: {
    tab: 'export',              // what the user navigated to
  }
})

// ERROR WITHOUT track(): if you catch an error outside of track(), record it manually.
onelo.monitor.event('ai_stream', {
  ok: false,
  error: 'timeout',             // short error label — shown in Feature Health
  meta: {
    model: 'gpt-4',
    failedAt: 'stream_start',
  }
})

// FEATURE FLAGS: feature() calls are tracked automatically — no extra code needed.

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
//   await onelo.monitor.track('pdf_export', async () => {
//     await generateAndUploadPdf(doc)
//   }, { meta: { pages: doc.pages, format: 'a4' } })
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     track() re-throws errors so your app handles them normally.
//     Always wrap in try/catch.
//
// USE event() FOR INSTANTANEOUS THINGS — a click, a tab switch, a state change:
//
//   onelo.monitor.event('tab_viewed', { ok: true, meta: { tab: 'export' } })
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead:
//
//   ✗ event('export_started', ...)   then   event('export_completed', ...)
//   ✓ track('export', async () => { /* entire export operation */ }, { meta })
//
// FINE-GRAINED — if one operation has multiple steps you need to diagnose
// individually, use track() for each step with descriptive names:
//
//   await onelo.monitor.track('pdf_render', () => render(doc), { meta: { pages } })
//   await onelo.monitor.track('pdf_upload', () => upload(file), { meta: { size } })
//
//   → Two rows, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   meta: { plan: 'pro', pages: 42, format: 'pdf' }
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
//   session_id  → UUID per app launch, groups all events into one session
//   user_id     → from setUserId() or Onelo Auth (if used)
//   platform    → e.g. 'electron', 'web', 'ios', 'android', 'flutter'
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
