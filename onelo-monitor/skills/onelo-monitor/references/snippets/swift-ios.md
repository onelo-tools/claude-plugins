# Swift — iOS

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=swift field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// Xcode → File → Add Package Dependencies:
// https://github.com/onelo-tools/onelo-swift
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=swift field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
 * overflow) and sent to Onelo in a single HTTP batch.
 *
 * DELIVERY IS RE-QUEUED, ONCE PER FLUSH: exactly ONE request goes out per flush.
 * A batch the server did not accept — network failure, 5xx, or a 429 — goes back
 * in the buffer and rides the next 15s tick. The flush timer IS the retry, so a
 * backend outage never multiplies the requests your users' apps send (an
 * in-flight retry loop would triple them at the worst possible moment). A 429
 * additionally holds off the next flushes for its Retry-After. Only a permanent
 * rejection (4xx other than 429 — bad key, malformed payload) is dropped, and it
 * is logged with its status rather than discarded quietly.
 *
 * WHAT IS STILL LOST: retry is IN-MEMORY only — nothing is persisted to disk, so
 * events still buffered when the process dies are gone. The buffer is capped (200
 * events); during a long outage the OLDEST re-queued events are dropped first so
 * live telemetry keeps flowing, and every drop is counted in a log line. So
 * Feature Health is a strong signal, not an audit log: use it to spot what is
 * failing, never as the record of truth for billing, compliance or anything that
 * must be complete.
 *
 * SESSION: the session ID is install-bound — it persists across launches and
 * resets only on app uninstall.
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
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```swift
// ─── STEP 0 — INVENTORY BEFORE YOU INSTRUMENT (do this first) ────────────────
// Do NOT write a single track()/event() call until you have listed the app's
// features. Instrumenting the parts you happened to read produces a dashboard
// that looks healthy because the broken half is not being measured.
//
// THE GATING TRAP — read this before you enumerate anything. If sign-in,
// paywall or the consent gate stands in front of EVERY feature you instrument,
// a cold start can emit ZERO events: a user stuck at sign-in never reaches any
// of the operations you measured. You can satisfy the coverage rule below
// perfectly — every feature instrumented, every skip justified — and still ship
// an app that looks completely unintegrated on Feature Health for as long as a
// user hasn't gotten past the gate. The fix is not "instrument more" — it's
// making sure at least ONE event can fire BEFORE any gate: an app-open ping at
// startup, or the gate's own attempt/success/failure (sign-in attempted,
// paywall shown, consent accepted/declined). Keep that in mind through step 1.
//
// 1. ENUMERATE. Walk the codebase and list every feature a user can trigger
//    end-to-end. A feature = a user-initiated operation with an OUTCOME that
//    can succeed or fail. Cover, at minimum:
//      • every entry point in the UI (button, menu item, form submit, shortcut)
//      • every route / screen the user can reach
//      • everything that crosses a boundary: network, filesystem, storage,
//        clipboard, subprocess, native bridge, third-party SDK
//      • startup and teardown paths (restore session, load state, migrate data)
//      • flows the Onelo SDK itself presents: sign-in, store, customer portal,
//        consent gate, feedback. The SDK renders them, but they fail inside YOUR
//        app — a user who cannot get past them has no working product. "The SDK
//        handles it" is not a reason to leave them unmeasured. These are also
//        your best candidate for an event that fires BEFORE the gating trap
//        above bites: instrument the attempt, not just the eventual success.
//
// 2. FIND THE SILENT FAILURES. Grep for swallowed errors: an empty catch block,
//    a catch that returns null / [] / a default, a ?? fallback that hides a
//    failed read, and fire-and-forget calls (void somePromise(),
//    .catch(() => {})). These are the highest-value events in any codebase —
//    they are exactly the failures that reach the user and reach nobody else.
//    Instrument them FIRST.
//
// 3. WRITE THE TABLE. Before coding, output one row per feature:
//      feature | file | event name | track() or event() | covered / skipped + why
//    Show it to the developer. A skipped feature must say WHY it was skipped
//    (see WHAT NOT TO INSTRUMENT below) — an unmeasured feature and a
//    deliberately-excluded one must never look the same in your report.
//
// 4. THEN implement, and keep the table in the PR description or a comment.
//
// 5. VERIFY. After implementing, actually trigger one instrumented feature,
//    wait ~15s (the flush interval — see below), reload the Onelo dashboard's
//    Feature Health tab and confirm a row appears for it. "The code compiles
//    and calls event()" is not verification; a row on the dashboard is.
//
// COVERAGE RULE: every feature in the table is either instrumented, or listed
// with a stated reason. "I did not get to it" is not a reason.

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

// ─── GRANULARITY (depth is yours — coverage is not) ──────────────────────────
// You choose how DEEP to go on a given feature: one event for the whole
// operation, or one per diagnosable step. You do NOT choose which features get
// measured — that is settled by the STEP 0 inventory above.
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

// ─── WHEN "RESOLVED" IS NOT "SUCCEEDED" ──────────────────────────────────────
// track() sets ok:false only when the callback THROWS. An operation that
// returns empty-handed — null session, zero rows, cancelled picker — is still
// recorded as a success. If "no result" means the feature failed for your user,
// raise it inside the callback and handle it outside:
//
//   do {
//     let rows = try await onelo.monitor.track("library_load", meta: ["source": "disk"]) {
//       let found = try await loadLibrary()
//       if found.isEmpty { throw LibraryError.empty }   // denied? wrong path?
//       return found
//     }
//   } catch {
//     // already recorded as ok:false with its duration — show the empty state
//   }
//
// One truthful event per attempt, with timing on the failure path too. Do NOT
// emit a second event() for the failure — that splits one feature into two rows.

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
