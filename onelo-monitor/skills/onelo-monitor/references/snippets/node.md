# Node.js — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=node field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

## init
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

## usage
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
