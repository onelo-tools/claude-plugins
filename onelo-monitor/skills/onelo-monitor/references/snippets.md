# Onelo Monitor — snippets

The official Onelo Monitor code for every supported platform, **baked into this
file when the plugin was published** — straight from `@onelo/snippets`, the same
source the dashboard **SDK** tab and **/docs** render from.

Each section has three parts: **install** (adding the package), **init**
(starting Monitor once at app startup, including crash capture) and **usage**
(`track` / `event` / `capture` at a call site).

Use these verbatim. The language reference files ([swift.md](swift.md),
[python.md](python.md), [js.md](js.md)) explain *where* and *why* to place a
call and how to name it — but the API shape comes from here, never from memory
and never adapted from another language.

If a platform has no section below, Monitor has no supported snippet for it.
Say so and stop rather than inventing a call.

---

## JS

### install
<!-- onelo:snippet sdk=monitor lang=js field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=js field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
/*
 * PLACEMENT: Create ONE Onelo instance at module level — not inside a
 * component, hook, or request handler. Export it and import wherever needed.
 * Re-creating the instance loses the buffer and resets the session ID.
 *
 * LIFECYCLE: The SDK flushes automatically. For SPAs, no teardown needed.
 * For SSR (Next.js, Remix), create the instance once at server startup,
 * not inside a request handler.
 *
 * HOW IT WORKS: Events are buffered in memory and sent to Onelo in a single
 * HTTP batch every 15 seconds. Errors (ok: false) flush immediately.
 *
 * SESSION: A UUID session ID is generated at init and attached to every event.
 * It resets on page reload. Use setUserId() to identify the current user.
 */
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  // Optional — set once here, never per-event. These surface as the
  // "Release", "App build" and "Environment" columns in Feature Health:
  // appVersion: '1.2.0',
  // appBuild: '420',
  // environment: 'production',
})

export default onelo
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=js field=usage -->
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
//                       (you can also set this once via the 'environment'
//                        option on new Onelo() — a per-event value wins)
// Custom keys still work — they just don't get column / filter status.
// "Release" and "App build" come from the optional appVersion / appBuild you
// pass to new Onelo() above — set them ONCE there, never per-event.
//
// ─── WHAT SDK AUTO-ATTACHES (do NOT add these to meta manually) ──────────────
//   session_id  → UUID per app launch, groups all events into one session
//   user_id     → from setUserId() or Onelo Auth (if used)
//   platform    → 'js'
//   meta.sdk    → { name: '@onelo/js', version } so the dashboard can flag
//                 outdated SDKs — no setup needed
//   meta.app    → { version, build } from appVersion / appBuild in your config
//   timestamp   → UTC, stamped server-side when the batch is received

// ─── WHAT NOT TO INSTRUMENT ──────────────────────────────────────────────────
//   ✗ Mouse move, hover, scroll position (too frequent, no actionable signal)
//   ✗ Every keystroke in an input field (track _submitted instead)
//   ✗ Internal UI state toggles with no outcome (e.g. dropdown open/close)
//   ✗ Events faster than ~1 per second per user (aggregate on your side first)
//   ✗ PII in meta — no emails, full names, passwords, or card numbers
```
<!-- /onelo:snippet -->

---

## SWIFT

### install
<!-- onelo:snippet sdk=monitor lang=swift field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=swift field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
/*
 * PLACEMENT: Initialize Onelo ONCE for the lifetime of the app.
 *   • SwiftUI: hold it as @StateObject on your @main App struct, then inject
 *     via .environmentObject(onelo) — see example below.
 *   • AppKit/UIKit: create in AppDelegate and expose as a singleton.
 * Never create inside a View — SwiftUI recreates views frequently.
 *
 * AUTOMATIC ERROR CAPTURE: NSException handlers + MetricKit (crashes,
 * hangs, CPU exceptions) are installed at init. Stack traces, breadcrumbs,
 * device context, and feature-flag state are auto-attached to every error.
 * No setup required. NOTE: MetricKit delivers reports daily, on real
 * devices only — crashes are NOT visible in the iOS Simulator and may
 * take up to 24h to arrive in production.
 *
 * LIFECYCLE: The SDK flushes every 15 seconds. Errors flush immediately.
 *   • macOS: call destroy() in applicationWillTerminate.
 *   • iOS:   applicationWillTerminate fires unreliably — also flush on
 *            UIApplication.willResignActiveNotification or ScenePhase
 *            .background so events aren't lost when the app is suspended.
 *   func applicationWillTerminate(_ notification: Notification) {
 *     onelo.monitor.destroy()   // invalidates flush timer + triggers a
 *                                // final flush; HTTP send is fire-and-forget
 *                                // (does NOT block termination). Force-quit
 *                                // / system shutdown may drop the last batch.
 *                                // Idempotent — safe to call from multiple
 *                                // lifecycle hooks; second call is a no-op.
 *   }
 *
 * MANUAL FLUSH: call onelo.monitor.flush() before a known risky moment
 * (e.g. just before app-initiated quit, or after a critical user action
 * you want to ship immediately). Safe to call from any thread; non-blocking.
 *
 * THREAD SAFETY: All public methods (event, track, capture, breadcrumb,
 * flush, setUserId) are safe to call from ANY thread. Internally the SDK
 * uses a serial queue — no need to wrap calls in MainActor.run { }.
 *
 * HOW IT WORKS: Events are buffered in memory (max 200, oldest dropped on
 * overflow) and sent to Onelo in a single HTTP batch. Failed batches are
 * dropped — they are NOT retried or persisted to disk in this version, so
 * extended offline periods will lose events. Session ID is install-bound
 * (persists across launches, resets only on app uninstall).
 *
 * PRIVACY: Sensitive keys (password, token, authorization, cookie, etc.)
 * and patterns (Bearer tokens, JWT, Stripe keys, credit-card numbers) are
 * automatically redacted from `error` strings and `meta` payloads before
 * leaving the device.
 */
import OneloSwift

// SwiftUI setup — @StateObject ensures the same instance survives view rebuilds.
@main
struct MyApp: App {
  // callbackScheme is your app's custom URL scheme — used by Onelo Auth to
  // return from the hosted sign-in page (e.g. "myapp://auth-callback").
  // The Monitor SDK itself does not use it; safe to pass any unique string
  // if you don't use Onelo Auth, but it must match your Info.plist scheme.
  @StateObject private var onelo = Onelo(
    publishableKey: "onelo_pk_live_YOUR_KEY",
    callbackScheme: "myapp",
    baseURL: URL(string: "https://api.onelo.tools")!
  )
  var body: some Scene {
    WindowGroup { ContentView().environmentObject(onelo) }
  }
}
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=swift field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// IDENTIFY YOUR USER
// If using Onelo Auth — user_id is set automatically, nothing to do.
// If using your own auth system:
// onelo.monitor.setUserId("user-123")  // after login
// onelo.monitor.setUserId(nil)          // after logout
//
// STARTUP IDENTITY RACE (Onelo Auth users — read this once):
// Onelo Auth restores the session from Keychain ASYNCHRONOUSLY at startup
// (~hundreds of ms). Events emitted BEFORE auth becomes ready will ship with
// user_id="anonymous" — even for users who are actually signed in. This is
// not a bug; it's the cost of not blocking startup on disk I/O.
//
// If you need accurate user attribution on startup events, await readiness:
//
//   try? await onelo.auth.awaitReady()  // default 5s timeout; no throw on time
//   onelo.monitor.event("app_launched", options: .init(ok: true))
//
// Or gate non-critical startup events on onelo.auth.currentSession != nil
// and accept that pre-restore events are anonymous.

// AUTO-TRACK (sync) — track() re-throws, wrap in do/catch.
// On error: stack trace, breadcrumbs, and active feature-flag state are
// auto-attached to the event.
do {
  let result = try onelo.monitor.track("checkout", meta: [
    "plan": user.plan,          // correlate errors with plan types
    "method": "stripe",         // which provider / path
  ]) {
    try processPayment()
  }
} catch {
  // The thrown error is already recorded in Onelo before being re-thrown:
  //   • error string  → localizedDescription
  //   • errorType     → String(describing: type(of: error)) (e.g. "URLError")
  //   • stack + breadcrumbs + active flags are auto-attached
  // Handle UI feedback here.
}

// AUTO-TRACK (async)
do {
  let data = try await onelo.monitor.track("ai_response", meta: [
    "model": "gpt-4",
  ]) {
    try await fetchAI()
  }
} catch { }

// INSTANTANEOUS EVENT: use event() only for things with no duration.
onelo.monitor.event("tab_viewed", options: MonitorEventOptions(
  ok: true,
  meta: ["tab": "export"]
))

// ERROR WITHOUT track()
onelo.monitor.event("ai_stream", options: MonitorEventOptions(
  ok: false,
  error: "timeout",
  meta: ["model": "gpt-4", "failedAt": "stream_start"]
))

// MANUAL ERROR CAPTURE: full stack trace + breadcrumbs + flag state attached.
// Use when you've already caught an error outside of track() and want full context.
do {
  try riskyOperation()
} catch {
  onelo.monitor.capture(error, featureName: "checkout")
}

// BREADCRUMBS: optional trail of context attached to the next captured error.
// Up to 100 most-recent breadcrumbs are kept. Devs can reconstruct what
// happened before failure.
onelo.monitor.breadcrumb(info: "user clicked Pay")
onelo.monitor.breadcrumb(navigationTo: "CheckoutScreen")

// RECIPE — drop a breadcrumb right BEFORE a risky operation so it lands
// in the error context if the operation fails:
onelo.monitor.breadcrumb(info: "starting ai_stream, model=gpt-4")
try await onelo.monitor.track("ai_stream") { try await streamAI() }

// AUTO-TRACE HTTP: every request on this session becomes a breadcrumb.
// Sensitive headers (Authorization, Cookie, X-API-Key) and query params
// (token, key, secret) are scrubbed before storage.
// LIFETIME: urlSession() returns a NEW URLSession on every call. Create it
// once at app startup (e.g. lazy property on AppDelegate or a service object)
// and reuse it for all requests — do NOT call urlSession() per-request.
let session = onelo.monitor.urlSession()
let (data, _) = try await session.data(from: url)

// FEATURE FLAGS: reading a flag (onelo.features.feature("x").isEnabled) does NOT
// emit a monitor event by itself. Instead, the active flag snapshot is attached
// to any error captured afterwards — so you can answer "is this error
// correlated with flag X being enabled?" without instrumenting reads.

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
//   try await onelo.monitor.track("pdf_export", meta: [
//     "pages": doc.pages, "format": "a4"
//   ]) {
//     try await generateAndUploadPdf(doc)
//   }
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     track() re-throws errors so your app handles them normally.
//     Always wrap in do/catch.
//
// USE event() FOR INSTANTANEOUS THINGS — a click, a tab switch, a state change:
//
//   onelo.monitor.event("tab_viewed", options: MonitorEventOptions(
//     ok: true, meta: ["tab": "export"]
//   ))
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead:
//
//   ✗ event("export_started", ...)   then   event("export_completed", ...)
//   ✓ track("export") { /* entire export operation */ }
//
// FINE-GRAINED — if one operation has multiple steps you need to diagnose
// individually, use track() for each step with descriptive names:
//
//   try await onelo.monitor.track("pdf_render", meta: ["pages": pages]) {
//     try await render(doc)
//   }
//   try await onelo.monitor.track("pdf_upload", meta: ["size": file.size]) {
//     try await upload(file)
//   }
//
//   → Two rows, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   meta: ["plan": "pro", "pages": 42, "format": "pdf"]
//   Native types supported: Bool, Int, Double, String, Date (ISO-8601),
//   UUID, URL, plus nested dictionaries and arrays.
//   Unsupported types (custom enums, NSDate, class instances, etc.) fall
//   back to String(describing:) — they ship as strings, never throw.
//   Keep meta small: aim for under ~20 keys / 8 KB per event. Never put a
//   full API response or large blob in here.
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
//   session_id   → UUID per install (persists across launches, resets on uninstall)
//   user_id      → from setUserId() or Onelo Auth (if used)
//   platform     → "swift" (constant for this SDK)
//   timestamp    → UTC, set at event creation time
//   sdk          → { name: "onelo-swift", version }    → "SDK version" facet
//   app          → { version, build, bundleId } from Info.plist
//                  • app.build   → "App build" facet
//                  • app.version → "Release" facet
//
// ON ERROR EVENTS ONLY (auto-attached when ok=false):
//   stack        → full call stack with module + symbol + offset
//   breadcrumbs  → up to 100 most-recent breadcrumbs
//   flags        → active feature-flag evaluations at time of error
//   device       → model, OS, locale, timezone, memory, connection type

// ─── WHAT NOT TO INSTRUMENT ──────────────────────────────────────────────────
//   ✗ Mouse move, hover, scroll position (too frequent, no actionable signal)
//   ✗ Every keystroke in an input field (track _submitted instead)
//   ✗ Internal UI state toggles with no outcome (e.g. dropdown open/close)
//   ✗ Events faster than ~1 per second per user (aggregate on your side first)
//   ✗ PII in meta — sensitive keys/values are auto-redacted, but it's better
//     to never send them in the first place. No emails (unless intentional),
//     full names, passwords, tokens, or card numbers.
```
<!-- /onelo:snippet -->

---

## ELECTRON

### install
<!-- onelo:snippet sdk=monitor lang=electron field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

### init
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

### usage
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

---

## REACTNATIVE

### install
<!-- onelo:snippet sdk=monitor lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
/*
 * PLACEMENT: Create ONE Onelo instance at module level (e.g. src/onelo.ts).
 * Import and reuse it across the app. Never create inside a component or hook
 * — that would reset the session ID and lose buffered events on re-render.
 *
 * LIFECYCLE: The SDK buffers events and flushes every 15 seconds.
 * Errors flush immediately. Call onelo.destroy() in your root component's
 * cleanup if needed, though React Native apps typically don't need explicit teardown.
 *
 * HOW IT WORKS: Events are buffered in memory and sent to Onelo in a single
 * HTTP batch every 15 seconds. Errors (ok: false) flush immediately. A UUID
 * session ID is generated at init and attached to every event — it persists
 * for the lifetime of the JS bundle.
 *
 * GLOBAL ERROR CAPTURE: unhandled JS errors are captured automatically as
 * 'global_error' events (via ErrorUtils) — no setup needed. The SDK chains the
 * previous handler, so React Native's red-box / fatal path still runs.
 *
 * BACKGROUND: React Native may suspend JS execution in the background.
 * The 15s flush timer pauses automatically — no special handling needed.
 */
import { Onelo } from '@onelo/react-native'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // app id / package name for X-Bundle-Id — iOS auto-derives, ANDROID needs it (else 403 under enforcement)
  // Optional — set once here, never per-event. These surface as the
  // "Release", "App build" and "Environment" columns in Feature Health:
  // appVersion: '1.2.0',
  // appBuild: '420',
  // environment: 'production',
})

export default onelo
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// IDENTIFY YOUR USER
// If using Onelo Auth — user_id is set automatically, nothing to do.
// If using your own auth system, call setUserId() after login/logout:
// onelo.monitor.setUserId('user-123')  // after login
// onelo.monitor.setUserId(null)         // after logout

// AUTO-TRACK: wrap any async operation to measure time and capture errors.
// track() re-throws so your app handles it normally — always use try/catch.
try {
  const result = await onelo.monitor.track('checkout', () => processPayment(), {
    meta: {
      plan: user.plan,
      itemCount: cart.length,
      method: 'stripe',
    }
  })
} catch (err) {
  // error is already recorded in Onelo — handle UI feedback here
}

// INSTANTANEOUS EVENT: use event() only for things with no duration.
onelo.monitor.event('tab_viewed', {
  ok: true,
  meta: { tab: 'export' }
})

// ERROR WITHOUT track()
onelo.monitor.event('ai_stream', {
  ok: false,
  error: 'timeout',
  meta: { model: 'gpt-4', failedAt: 'stream_start' }
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
//                       (you can also set this once via the 'environment'
//                        option on new Onelo() — a per-event value wins)
// Custom keys still work — they just don't get column / filter status.
// "Release" and "App build" come from the optional appVersion / appBuild you
// pass to new Onelo() above — set them ONCE there, never per-event.
//
// ─── WHAT SDK AUTO-ATTACHES (do NOT add these to meta manually) ──────────────
//   session_id  → UUID per app launch, groups all events into one session
//   user_id     → from setUserId() or Onelo Auth (if used)
//   platform    → 'reactnative' (constant for this SDK)
//   timestamp   → UTC, set at event creation time
//   meta.sdk    → { name: '@onelo/react-native', version } — shown as the
//                 "SDK version" column in Feature Health. Auto — never set it.
//   meta.app    → { version, build } from appVersion / appBuild in your config
//   meta.device → OS, version, timezone (+ model / brand on error events)

// ─── WHAT NOT TO INSTRUMENT ──────────────────────────────────────────────────
//   ✗ Mouse move, hover, scroll position (too frequent, no actionable signal)
//   ✗ Every keystroke in an input field (track _submitted instead)
//   ✗ Internal UI state toggles with no outcome (e.g. dropdown open/close)
//   ✗ Events faster than ~1 per second per user (aggregate on your side first)
//   ✗ PII in meta — no emails, full names, passwords, or card numbers
```
<!-- /onelo:snippet -->

---

## ANDROID

### install
<!-- onelo:snippet sdk=monitor lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
/*
 * PLACEMENT: Initialize Onelo ONCE in your Application class, not in an
 * Activity or Fragment. Activities can be recreated (rotation, back stack)
 * which would reset the session ID and lose buffered events.
 *
 *   class MyApp : Application() {
 *     val onelo by lazy { Onelo(OneloConfig(...), this) }
 *   }
 *
 * Access it anywhere via: (application as MyApp).onelo
 * Or use a dependency injection framework (Hilt, Koin) to provide it.
 *
 * LIFECYCLE: The SDK buffers events and flushes every 15 seconds using
 * coroutines on Dispatchers.IO. Errors flush immediately.
 * Call onelo.monitor.flushBlocking(2000L) in Application.onTerminate() to flush on shutdown.
 *
 * HOW IT WORKS: Events are buffered in memory (max 200) and sent to Onelo
 * in a single HTTP batch. Session ID is a UUID generated at init — resets
 * on process restart, persists across activity recreation.
 *
 * COROUTINES: track() is a suspend function — call it inside a coroutine
 * scope (lifecycleScope, viewModelScope, etc.). event() is regular fun.
 */
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val config = OneloConfig(
  publishableKey = "onelo_pk_live_YOUR_KEY",
  apiUrl = "https://api.onelo.tools",
)
val onelo = Onelo(config, this)
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// IDENTIFY YOUR USER
// If using Onelo Auth — user_id is set automatically, nothing to do.
// If using your own auth system:
// onelo.monitor.setUserId("user-123")  // after login
// onelo.monitor.setUserId(null)         // after logout

// AUTO-TRACK: track() is a suspend function — call inside a coroutine.
// It re-throws exceptions so your app handles them normally.
lifecycleScope.launch {
  try {
    val result = onelo.monitor.track("checkout", meta = mapOf(
      "plan" to user.plan,         // correlate errors with plan types
      "itemCount" to cart.size,    // what they were doing
      "method" to "stripe",        // which provider / path
    )) { processPayment() }
  } catch (e: Exception) {
    // error is already recorded in Onelo — handle UI feedback here
  }
}

// INSTANTANEOUS EVENT: use event() only for things with no duration.
onelo.monitor.event("tab_viewed", ok = true, meta = mapOf(
  "tab" to "export",
))

// ERROR WITHOUT track()
onelo.monitor.event("ai_stream", ok = false, error = "timeout", meta = mapOf(
  "model" to "gpt-4",
  "failedAt" to "stream_start",
))

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
//     with ok = false, error = "denied" | "not_in_bundle" | "timeout"

// ─── GRANULARITY — YOU DECIDE ────────────────────────────────────────────────
// You control how detailed the tracking is. There is no "correct" level —
// choose based on what questions you need to answer in Feature Health.
//
// USE track() FOR OPERATIONS — it wraps suspend work, measures time automatically,
// and sends ONE event covering the whole operation:
//
//   onelo.monitor.track("pdf_export", meta = mapOf("pages" to doc.pages, "format" to "a4")) {
//     generateAndUploadPdf(doc)
//   }
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     track() re-throws errors so your app handles them normally.
//     Always wrap in try/catch.
//
// USE event() FOR INSTANTANEOUS THINGS — a click, a tab switch, a state change:
//
//   onelo.monitor.event("tab_viewed", ok = true, meta = mapOf("tab" to "export"))
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead:
//
//   ✗ event("export_started", ...)   then   event("export_completed", ...)
//   ✓ track("export", meta = ...) { /* entire export operation */ }
//
// FINE-GRAINED — if one operation has multiple steps you need to diagnose
// individually, use track() for each step with descriptive names:
//
//   onelo.monitor.track("pdf_render", meta = mapOf("pages" to pages)) { render(doc) }
//   onelo.monitor.track("pdf_upload", meta = mapOf("size" to size)) { upload(file) }
//
//   → Two rows, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   meta = mapOf("plan" to "pro", "pages" to 42, "format" to "pdf")
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

---

## FLUTTER

### install
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

### init
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

### usage
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

---

## PYTHON

### install
<!-- onelo:snippet sdk=monitor lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install "onelo[fastapi] @ git+https://github.com/onelo-tools/onelo-python.git"   # extras: [django] / [flask] / [litestar]
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=python field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# PLACEMENT: call monitor.init() ONCE during app startup, before any
# request handler runs. For FastAPI / Litestar do it in a lifespan or
# top-level module; for Django put it in settings.py.
#
# PER-REQUEST ISOLATION: each integration forks an isolation scope per
# request so user_id / breadcrumbs from concurrent requests never bleed
# into each other (ContextVars under the hood).
#
# CRASH CAPTURE: install_excepthook=True wires sys.excepthook,
# threading.excepthook, and the active asyncio loop exception handler
# so uncaught errors are reported even if no framework error handler ran.
#
# PRIVACY: sensitive headers (Authorization, Cookie, X-API-Key) and query
# params (token, key, secret, password, ...) are scrubbed before any data
# leaves your process — same regex set as the Swift / Electron SDK.
import os
from onelo import Onelo, monitor

# ─── ENVIRONMENT VARIABLES — set these BEFORE starting your backend ──────────
# This is a BACKEND (server-side) integration, so it authenticates with your
# SERVER SECRET key — NOT a publishable key (publishable keys are for client
# apps). Create the secret key in the Onelo dashboard → API Keys → Secret keys,
# then expose it to the process via env (never hard-code a secret in source):
#
#   ONELO_SECRET_KEY    REQUIRED   your server secret key, "onelo_sk_live_..."
#                                  (create it: Onelo dashboard → API Keys)
#   ONELO_API_URL       REQUIRED   Onelo API base URL → "https://api.onelo.tools"
#
# (Monitor has no test/live split — that's a Features-only concept
# (ONELO_FEATURE_ENVIRONMENT). Monitor is a single event stream.)
#
onelo = Onelo(
    secret_key=os.environ["ONELO_SECRET_KEY"],              # onelo_sk_live_* — server-side only
    api_url=os.environ.get("ONELO_API_URL", "https://api.onelo.tools"),  # Onelo API base URL
)

monitor.init(
    onelo=onelo,                # reuse one client for auth + features + monitor
    release=os.environ.get("ONELO_RELEASE"),  # your build id — reads ONELO_RELEASE (else GIT_COMMIT_SHA)
    install_excepthook=True,
    # High-throughput backend? Sample client-side BEFORE the network so an error
    # storm can't thrash the buffer or trip the hourly quota. Default 1.0 = keep all.
    # sample_rate=0.25,          # keep-probability for ERROR events (ok=False)
    # success_sample_rate=0.05,  # keep-probability for success / track events (usually the high-volume ones)
)

# FastAPI:
from fastapi import FastAPI
from onelo.monitor.integrations import OneloMonitorASGIMiddleware
app = FastAPI()
app.add_middleware(OneloMonitorASGIMiddleware)

# Django:  add to MIDDLEWARE in settings.py
#   "onelo.monitor.integrations.django.OneloMonitorMiddleware",
#
# Flask:
#   from onelo.monitor.integrations.flask import install
#   install(app)
#
# Litestar / Starlette:  pass OneloMonitorASGIMiddleware in middleware=[...]
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=python field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# IDENTIFY THE USER on each request (Onelo Auth populates this for you;
# call manually only if you have your own auth):
monitor.set_user({"id": user.id, "email": user.email})

# TRACK AN OPERATION — measures wall-clock time and emits ONE event:
# ok=True on success, or an error event (full stack + breadcrumbs + active
# feature flags) if the block raises. The exception still propagates, so your
# control flow is unchanged. Same semantics as the Swift SDK's track().
# Python has no trailing-closure idiom, so track() works three ways — pick one:

# 1) context manager (sync)
with monitor.track("checkout", meta={"plan": user.plan}):
    process_payment()

# 2) context manager (async)
async with monitor.track("ai_response", meta={"model": "gpt-4"}):
    await call_model()

# 3) decorator (works on sync AND async functions)
@monitor.track("pdf_export")
async def export(doc):
    await render_and_upload(doc)

# Name the FEATURE, not the outcome — track() records ok/error/duration for you.
# Prefer track() for operations; use capture_exception() (below) only for errors
# you catch OUTSIDE a track() block.

# CAPTURE AN EXCEPTION manually. Always call from inside an except block:
try:
    process_payment()
except Exception:
    monitor.capture_exception()  # uses the active sys.exc_info()

# Or pass an explicit exception:
try:
    risky()
except RuntimeError as exc:
    monitor.capture_exception(exc, feature_name="payment", meta={"plan": user.plan})

# INSTANTANEOUS EVENT: no duration, just a marker.
monitor.capture_message("rate-limit triggered", level="warning",
                        feature_name="rate_limit")

# BREADCRUMBS: trail of context attached to the next captured event.
# Up to 100 most-recent are kept per request.
monitor.add_breadcrumb("loaded user", category="db")
monitor.add_breadcrumb("called Stripe", category="http")

# AUTO-TRACE OUTGOING HTTP: wrap your shared httpx / requests client once.
import httpx
from onelo.monitor.integrations.httpx import wrap_async_transport
client = httpx.AsyncClient(transport=wrap_async_transport())

import requests
from onelo.monitor.integrations.requests import install_session
session = install_session(requests.Session())

# AUTO-CAPTURE FROM logging: errors -> events, warnings/info -> breadcrumbs.
import logging
from onelo.monitor.integrations.logging import OneloLoggingHandler
logging.getLogger().addHandler(OneloLoggingHandler())

# BACKGROUND JOBS (Celery / RQ): producer attaches the carrier, worker
# decorates the task and inherits user_id / tags / trace_id automatically.
@celery_app.task(bind=True)
@monitor.continue_trace_task
def my_task(self, *args, **kwargs):
    monitor.add_breadcrumb("task body running")
    do_work()

# Producer:
my_task.apply_async(args=(...,), headers={"onelo": monitor.carrier()})

# FEATURE FLAG CORRELATION (auto): pass onelo=client to monitor.init and
# every captured event ships with active flag values, so the dashboard can
# show "this error spiked when flag X went from off to on" without extra
# wiring on your side.

# ─── WHAT SDK AUTO-ATTACHES ─────────────────────────────────────────────────
#   trace_id           → per request (or per background task via continue_trace)
#   user_id            → from monitor.set_user(...)
#   request context    → method, scrubbed URL, scrubbed headers
#   sdk / app / flags  → versions + active feature-flag snapshot
#   stack / frames     → on every captured exception (in_app classifier)
#   breadcrumbs        → up to 100 most recent (HTTP, log, custom)

# ─── WHAT NOT TO INSTRUMENT ─────────────────────────────────────────────────
#   ✗ DEBUG-level logs (would flood the breadcrumb buffer in busy services)
#   ✗ Health checks (/healthz) — they aren't failures and add no signal
#   ✗ Validation errors that the framework already returns as 4xx — those are
#     usually expected behaviour, not bugs
#   ✗ PII in meta — passwords, tokens, full credit-card numbers are scrubbed
#     by the SDK, but it's better not to send them in the first place
```
<!-- /onelo:snippet -->

---

## NODE

### install
<!-- onelo:snippet sdk=monitor lang=node field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=node field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// PLACEMENT: call onelo.monitor.init() ONCE during startup, before any
// request is served.
//
// PER-REQUEST ISOLATION: each framework adapter forks an isolation scope per
// request (AsyncLocalStorage) so user / breadcrumbs from concurrent requests
// never bleed into each other.
//
// CRASH CAPTURE: by default init() observes uncaughtException +
// unhandledRejection WITHOUT changing your process's exit behavior.
//
// PRIVACY: sensitive headers (Authorization, Cookie, X-API-Key) and query
// params (token, key, secret, password, ...) are scrubbed before any data
// leaves your process — same regex set as the Swift / Python SDK.
import { Onelo } from '@onelo/node'
import { monitorRequestHandler, monitorErrorHandler } from '@onelo/node/express'

// ─── ENVIRONMENT VARIABLES — set these BEFORE starting your backend ──────────
// This is a BACKEND integration, so it authenticates with your SERVER SECRET
// key — NOT a publishable key. Create it in the Onelo dashboard → API Keys →
// Secret keys and read it from the environment (never hard-code a secret):
//
//   ONELO_SECRET_KEY    REQUIRED   "onelo_sk_live_..."  (server-side only)
//   ONELO_API_URL       REQUIRED   Onelo API base URL → "https://api.onelo.tools"
//
// (Monitor has no test/live split — that's a Features-only concept. Monitor is
// a single event stream.)
const onelo = new Onelo({
  secretKey: process.env.ONELO_SECRET_KEY!,               // onelo_sk_live_*
  apiUrl: process.env.ONELO_API_URL ?? 'https://api.onelo.tools',
})

onelo.monitor.init({
  release: process.env.ONELO_RELEASE, // your build id — reads ONELO_RELEASE (else GIT_COMMIT_SHA)
  environment: process.env.NODE_ENV, // production / staging / development (else ONELO_ENVIRONMENT / ENVIRONMENT)
  // AUTO-CAPTURE (opt-in) — instrument the app without wrapping every call:
  captureConsole: true,   // console.error → event (an Error arg → full stack); console.warn → breadcrumb
  httpBreadcrumbs: true,  // each outgoing fetch → an http breadcrumb (URL scrubbed); Onelo's own traffic excluded
  // (Source lines for in-app exception frames are attached automatically.)
  // High-throughput backend? Sample client-side BEFORE the network so an error
  // storm can't thrash the buffer or trip the hourly quota. Default 1.0 = keep all.
  // sampleRate: 0.25,          // keep-probability for ERROR events (ok=false)
  // successSampleRate: 0.05,   // keep-probability for track/success events
})

// Express — wrap requests in a per-request scope, then capture thrown errors:
app.use(monitorRequestHandler(onelo))   // place FIRST, before your routes
// ...your routes...
app.use(monitorErrorHandler(onelo))     // place LAST (after routes)

// Next.js  → import { withMonitor } from '@onelo/node/next'
//            export const GET = withMonitor(onelo, async (req) => { ... })
// Fastify   → import { registerOneloMonitor } from '@onelo/node/fastify'
//            registerOneloMonitor(fastify, onelo)
// Hono      → import { oneloMonitor } from '@onelo/node/hono'
//            app.use('*', oneloMonitor(onelo))

// GRACEFUL SHUTDOWN — flush buffered events before the process exits:
process.on('SIGTERM', async () => { await onelo.close(); process.exit(0) })
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=monitor lang=node field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// IDENTIFY THE USER (Onelo Auth's requireUser does this for you; call
// manually only if you use your own auth):
onelo.monitor.setUser({ id: user.id })

// TRACK AN OPERATION — measures wall-clock time and emits ONE event: ok=true on
// success, or an error event (full stack + breadcrumbs + active feature flags)
// if it throws. The result/exception propagates unchanged. Sync OR async:
await onelo.monitor.track('checkout', async () => {
  await processPayment()
})
const total = onelo.monitor.track('sum_totals', () => a + b)   // sync

// Name the FEATURE, not the outcome — track() records ok/error/duration for you.

// CAPTURE AN EXCEPTION you catch outside a track() block:
try {
  await risky()
} catch (err) {
  onelo.monitor.captureException(err, { featureName: 'payment', meta: { plan: user.plan } })
}

// INSTANTANEOUS EVENT / MESSAGE (no duration, just a marker):
onelo.monitor.captureMessage('rate-limit triggered', {
  level: 'warning',
  featureName: 'rate_limit',
})

// BREADCRUMBS: trail of context attached to the next captured event
// (up to 100 most-recent per request):
onelo.monitor.addBreadcrumb('loaded user', { category: 'db' })
onelo.monitor.addBreadcrumb('called Stripe', { category: 'http' })

// WEBSOCKET / IN-PROCESS HANDLERS (no HTTP middleware) — fork an isolation scope
// and attach the user so events captured inside carry userId + isolated breadcrumbs:
await onelo.monitor.runIdentified(userId, async () => {
  await handleSocketMessage()
})

// CROSS-PROCESS JOBS (BullMQ / SQS / a separate worker) — a background job runs
// OUTSIDE the request, so its scope is empty. Attach a carrier when you enqueue,
// then restore it in the worker so errors there carry the request's user + tags.
// Pass your publishable key to HMAC-sign it (the worker verifies before trusting):
//   Producer (request side):
await queue.add('send-invoice', {
  ...payload,
  onelo: onelo.monitor.carrier({ key: onelo.key }),
})
//   Worker (job side):
await onelo.monitor.continueTrace(job.data.onelo, async () => {
  await sendInvoice()   // an error here is attributed to the dispatching user
}, { key: onelo.key })

// FEATURE FLAG CORRELATION (auto): every captured event ships the active flag
// snapshot, so the dashboard can show "this error spiked when flag X turned on"
// — no extra wiring on your side.

// ─── WHAT SDK AUTO-ATTACHES ─────────────────────────────────────────────────
//   userId               → from onelo.monitor.setUser(...)
//   request context      → method + path (scrubbed)
//   sdk / app / flags     → SDK version + active feature-flag snapshot
//   stack / frames        → on every captured exception (in_app classifier)
//   breadcrumbs           → up to 100 most recent
//   release / environment → from monitor.init(...)

// ─── WHAT NOT TO INSTRUMENT ─────────────────────────────────────────────────
//   ✗ Health checks (/healthz) — not failures, no signal
//   ✗ Validation errors the framework already returns as 4xx — usually expected
//   ✗ PII in meta — secrets are scrubbed by the SDK, but better not to send them
```
<!-- /onelo:snippet -->

---

## PHP

### install
<!-- onelo:snippet sdk=monitor lang=php field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=monitor lang=php field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
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

### usage
<!-- onelo:snippet sdk=monitor lang=php field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
<?php
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
