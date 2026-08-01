# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
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

## usage
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
