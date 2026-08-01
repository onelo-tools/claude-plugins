# Python — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=monitor lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install "onelo[fastapi] @ git+https://github.com/onelo-tools/onelo-python.git"   # extras: [django] / [flask] / [litestar]
```
<!-- /onelo:snippet -->

## init
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

## usage
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
