# Python — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=python field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```python
pip install "onelo[fastapi] @ git+https://github.com/onelo-tools/onelo-python.git"   # extras: [django] / [flask] / [litestar]
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=monitor lang=python field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
# DELIVERY IS RETRIED: events buffer in memory and ship as one HTTP batch.
# A batch that fails on the network, or gets a 5xx, is RETRIED up to 3 times
# with exponential backoff (0.5s → 1s → 2s); a 429 is not retried but sets a
# Retry-After hold-off, and other 4xx are terminal. The buffer is emptied
# before the send, so a batch that exhausts its 3 attempts is dropped and
# logged — never re-queued. Feature Health is a strong signal, not an audit log.
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

## usage
<!-- onelo:snippet sdk=monitor lang=python field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```python
# ─── STEP 0 — INVENTORY BEFORE YOU INSTRUMENT (do this first) ────────────────
# Do NOT write a single monitor.track() / capture_message() call until you have listed what
# this service actually does. Instrumenting the handlers you happened to read
# produces a dashboard that looks healthy because the broken half is not being
# measured.
#
# 1. ENUMERATE. Walk the codebase and list every unit of work the service
#    performs end-to-end — anything with an OUTCOME that can succeed or fail:
#      • every HTTP route / endpoint (including webhooks and callbacks), grouped
#        by the business operation it performs, not by URL shape
#      • every background job, queue consumer, scheduled/cron task
#      • every outbound third-party integration: payment provider, mail, storage,
#        search, LLM provider, any other API you do not control
#      • every boundary crossing inside the process: database, cache, filesystem,
#        subprocess
#      • startup, migration and shutdown paths (config load, schema migration,
#        warm-up, graceful drain)
#
# 2. FIND THE SILENT FAILURES. Grep for swallowed errors: an empty except/catch,
#    a handler returning an empty list or a default on failure, a fallback value
#    that hides a failed read, a fire-and-forget task nobody awaits, and any
#    place that logs and continues. These are the highest-value events in any
#    backend — they are exactly the failures that reach the caller and reach
#    nobody else. Instrument them FIRST.
#
# 3. WRITE THE TABLE. Before coding, output one row per unit of work:
#      operation | file | event name | monitor.track() or capture_message() | covered / skipped + why
#    Show it to the developer. A skipped operation must say WHY it was skipped
#    (see WHAT NOT TO INSTRUMENT below) — an unmeasured operation and a
#    deliberately-excluded one must never look the same in your report.
#
# 4. THEN implement, and keep the table in the PR description or a comment.
#
# 5. VERIFY. After implementing, actually trigger one instrumented operation,
#    wait ~15s (the flush interval — see below), reload the Onelo dashboard's
#    Feature Health tab and confirm a row appears for it. "The code compiles
#    and calls monitor.track()" is not verification; a row on the dashboard is.
#
# THE GATING TRAP — if auth middleware or a feature flag stands in front of
# EVERY route you instrumented, a healthy-but-locked-out deployment (bad
# credentials, a misconfigured flag) still shows zero events, indistinguishable
# from "not integrated." Make sure at least one operation you instrument sits
# OUTSIDE any such gate — a health-check route, the auth middleware's own
# accept/reject outcome, or process startup itself.
#
# COVERAGE RULE: every operation in the table is either instrumented, or listed
# with a stated reason. "I did not get to it" is not a reason.

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

# ─── NAMING CONVENTION ───────────────────────────────────────────────────────
# Use snake_case: {object}_{past_tense_verb}
# Object  → what the user acts on: checkout, pdf_export, ai_response, sync
# Verb    → past tense, result-oriented: completed, failed, started, retried
#
# Good:  checkout  pdf_export  ai_response  onboarding  sync  tab_viewed
# Bad:   checkout_completed  export_error  doSync  exportClicked
#
# Name the FEATURE, not the outcome — track() records ok/error/duration automatically.
# Use _verb suffix only for capture_message() calls (instantaneous, no duration):
#   tab_viewed  button_clicked  plan_selected  modal_dismissed
#
# Avoid _started / _completed suffixes — use track() to wrap the operation instead.
#
# PITFALL — check-style events: don't name an event after its failure mode.
# The name shows up in Feature Health even on success, so failure-mode names
# read like permanent alerts.
#   ✗ permission_missing, binary_missing, network_unavailable
#   ✓ permission_check, embedded_binary_check, network_check
#     with ok=False, error="denied" | "not_found" | "timeout"

# ─── GRANULARITY (depth is yours — coverage is not) ──────────────────────────
# You choose how DEEP to go on a given route or job: one event for the whole
# request, or one per diagnosable step. You do NOT choose which routes and jobs
# get measured — that is settled by the STEP 0 inventory above.
#
# USE track() FOR OPERATIONS — it wraps the work, measures wall-clock time, and
# emits ONE event covering the whole operation:
#
#   with monitor.track("invoice_issue", meta={"provider": "fakturownia"}):
#       issue_invoice(order)
#
#   → ONE row in Feature Health: success rate, avg duration, error label.
#     The exception still propagates, so your control flow is unchanged.
#
# USE capture_message() FOR INSTANTANEOUS THINGS — a marker with no duration:
# a rate-limit trip, a config reload, a feature toggled off by a kill switch:
#
#   monitor.capture_message("rate-limit triggered", level="warning",
#                           feature_name="rate_limit")
#
# DO NOT split one operation into _started / _completed events — that creates
# two separate unrelated rows in Feature Health. Use track() instead.
#
# FINE-GRAINED — if one request has several steps you need to diagnose
# individually (DB query, third-party call, PDF render), wrap each in its own
# track() with a descriptive name:
#
#   with monitor.track("invoice_render", meta={"lines": len(order.items)}):
#       pdf = render_invoice(order)
#   with monitor.track("invoice_upload", meta={"size": len(pdf)}):
#       upload(pdf)
#
#   → One row per step, each with independent success rates and timing.
#     Add granularity only when the coarse single-event view is not enough.
#
# EVERYTHING IN META IS QUERYABLE in Feature Health sessions:
#   meta={"plan": "pro", "region": "eu", "provider": "stripe"}
#   → filter sessions by those values to debug "only fails for free-plan users".

# ─── WHEN "RESOLVED" IS NOT "SUCCEEDED" ──────────────────────────────────────
# track() sets ok:false only when the block RAISES. An operation that
# returns empty-handed — null session, zero rows, cancelled picker — is still
# recorded as a success. If "no result" means the feature failed for your user,
# raise it inside the block and handle it outside:
#
#   try:
#       with monitor.track("library_load", meta={"source": "db"}):
#           rows = load_library(tenant_id)
#           if not rows:
#               raise EmptyLibrary()      # wrong tenant? RLS blocking the read?
#   except EmptyLibrary:
#       # already recorded as an error event with its duration — return 404, not 200 []
#       rows = []
#
# One truthful event per attempt, with timing on the failure path too. Do NOT
# emit a second capture_message() for the failure — that splits one feature into two rows.

# ─── WHAT SDK AUTO-ATTACHES ─────────────────────────────────────────────────
#   trace_id           → per request (or per background task via continue_trace)
#   user_id            → from monitor.set_user(...)
#   request context    → method, scrubbed URL, scrubbed headers
#   sdk / app / flags  → versions + active feature-flag snapshot
#   stack / frames     → on every captured exception (in_app classifier)
#   breadcrumbs        → up to 100 most recent (HTTP, log, custom)

# ─── WHAT NOT TO INSTRUMENT ─────────────────────────────────────────────────
#   ✗ DEBUG-level logs (would flood the breadcrumb buffer in busy services)
#   ✗ Health checks (/healthz) — not failures, no signal
#   ✗ Validation errors the framework already returns as 4xx — usually expected
#   ✗ PII in meta — secrets are scrubbed by the SDK, but better not to send them
```
<!-- /onelo:snippet -->
