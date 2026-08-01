# Swift — iOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=swift field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

## init
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

## usage
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
