# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=android field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=android field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

## usage
<!-- onelo:snippet sdk=monitor lang=android field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```kotlin
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

// ─── GRANULARITY (depth is yours — coverage is not) ──────────────────────────
// You choose how DEEP to go on a given feature: one event for the whole
// operation, or one per diagnosable step. You do NOT choose which features get
// measured — that is settled by the STEP 0 inventory above.
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

// ─── WHEN "RESOLVED" IS NOT "SUCCEEDED" ──────────────────────────────────────
// track() sets ok:false only when the callback THROWS. An operation that
// returns empty-handed — null session, zero rows, cancelled picker — is still
// recorded as a success. If "no result" means the feature failed for your user,
// raise it inside the callback and handle it outside:
//
//   lifecycleScope.launch {
//     try {
//       val rows = onelo.monitor.track("library_load", meta = mapOf("source" to "disk")) {
//         val found = loadLibrary()
//         if (found.isEmpty()) throw IllegalStateException("empty")  // denied? wrong path?
//         found
//       }
//     } catch (e: Exception) {
//       // already recorded as ok = false with its duration — show the empty state
//     }
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
//   session_id  → UUID per app launch, groups all events into one session
//   user_id     → from setUserId() or Onelo Auth (if used)
//   platform    → e.g. 'electron', 'web', 'ios', 'android', 'flutter'
//   timestamp   → UTC, stamped when the event HAPPENS, not when the batch is
//                 sent — so a batch delayed by an outage still reports the real
//                 time of the event rather than the moment delivery recovered
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
