# Node.js — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=node field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=node field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
// DELIVERY IS RETRIED: events buffer in memory and ship as one HTTP batch.
// A batch that fails on the network, or gets a 5xx, is RETRIED with
// exponential backoff; a 429 is honoured via Retry-After; other 4xx are
// terminal (retrying them is pointless). The buffer is emptied before the
// send, so a batch that exhausts its attempts is dropped and logged — never
// re-queued. Feature Health is a strong signal, not an audit log.
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

## usage
<!-- onelo:snippet sdk=monitor lang=node field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// ─── STEP 0 — INVENTORY BEFORE YOU INSTRUMENT (do this first) ────────────────
// Do NOT write a single monitor.track() / captureMessage() call until you have listed what
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
//      operation | file | event name | monitor.track() or captureMessage() | covered / skipped + why
//    Show it to the developer. A skipped operation must say WHY it was skipped
//    (see WHAT NOT TO INSTRUMENT below) — an unmeasured operation and a
//    deliberately-excluded one must never look the same in your report.
//
// 4. THEN implement, and keep the table in the PR description or a comment.
//
// 5. VERIFY. After implementing, actually trigger one instrumented operation,
//    wait ~15s (the flush interval — see below), reload the Onelo dashboard's
//    Feature Health tab and confirm a row appears for it. "The code compiles
//    and calls monitor.track()" is not verification; a row on the dashboard is.
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
//   await onelo.monitor.track('invoice_issue', async () => {
//     await issueInvoice(order)
//   }, { meta: { provider: 'stripe' } })
//
//   → ONE row in Feature Health: success rate, avg duration, error label.
//     The exception still propagates, so your control flow is unchanged.
//
// USE captureMessage() FOR INSTANTANEOUS THINGS — a marker with no duration:
// a rate-limit trip, a config reload, a feature toggled off by a kill switch:
//
//   onelo.monitor.captureMessage('rate-limit triggered', {
//     level: 'warning', featureName: 'rate_limit',
//   })
//
// DO NOT split one operation into _started / _completed events — that creates
// two separate unrelated rows in Feature Health. Use track() instead.
//
// FINE-GRAINED — if one request has several steps you need to diagnose
// individually (DB query, third-party call, PDF render), wrap each in its own
// track() with a descriptive name:
//
//   const pdf = await onelo.monitor.track('invoice_render', () => renderInvoice(order))
//   await onelo.monitor.track('invoice_upload', () => upload(pdf), { meta: { size: pdf.length } })
//
//   → One row per step, each with independent success rates and timing.
//     Add granularity only when the coarse single-event view is not enough.
//
// EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
//   meta: { plan: 'pro', region: 'eu', provider: 'stripe' }
//   → filter sessions by those values to debug "only fails for free-plan users".

// ─── WHEN "RESOLVED" IS NOT "SUCCEEDED" ──────────────────────────────────────
// track() sets ok:false only when the callback THROWS. An operation that
// returns empty-handed — null session, zero rows, cancelled picker — is still
// recorded as a success. If "no result" means the feature failed for your user,
// raise it inside the callback and handle it outside:
//
//   try {
//     const rows = await onelo.monitor.track('library_load', async () => {
//       const found = await loadLibrary(tenantId)
//       if (found.length === 0) throw new Error('empty_result')  // wrong tenant? RLS?
//       return found
//     })
//   } catch {
//     // already recorded as an error event with its duration — return 404, not 200 []
//   }
//
// One truthful event per attempt, with timing on the failure path too. Do NOT
// emit a second captureMessage() for the failure — that splits one feature into two rows.

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
