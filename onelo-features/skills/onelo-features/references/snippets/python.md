# Python — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install git+https://github.com/onelo-tools/onelo-python.git
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=python field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# ─────────────────────────────────────────────────────────────────
# Onelo Features SDK — Python (server-side / backend)
#
# 🔑 KEY — your live secret key (onelo_sk_live_*). Acts like a classic
#    server API key: SSE stream, identify(), monitor, auth verification.
#    KEEP IT THE SAME in dev/staging/prod. Get it from: dashboard →
#    API Keys → Secret keys.
#
# 🌐 FEATURE ENVIRONMENT — "test" or "live". THIS selects which feature
#    snapshot the server reads AND whether feature DISCOVERY is active —
#    NOT the key (no separate test/discovery key needed anymore). In
#    dev/staging set "test": the SDK reads the Test snapshot, registers
#    the slugs your code checks into the dashboard registry, and shows
#    online under the Test tab. In prod set "live" (or omit → live).
#    Set the SAME value in your client app (Swift OneloFeatureEnvironment)
#    so app + backend resolve the same snapshot.
#    Registry growth is bound to this app + INSTANCE — a stable per-process
#    id (set ONELO_INSTANCE_ID for containers; otherwise auto-generated and
#    persisted). The dashboard authorizes/unpins that instance under
#    Features → Registry → Deploy access. No key type is involved.
#    SETUP — just an env var per deployment target:
#      .env.staging    →  ONELO_FEATURE_ENVIRONMENT=test
#      .env.production →  ONELO_FEATURE_ENVIRONMENT=live   (or omit)
#
# 🔒 Keys: keep OUT of git (.env in .gitignore). Load prod secrets via
#    Vault / AWS Secrets Manager / K8s Secrets / etc.
# ─────────────────────────────────────────────────────────────────

# ── Module-level setup (runs once per process) ──────────────────
import os
from onelo import Onelo

onelo = Onelo(
    secret_key=os.environ["ONELO_SECRET_KEY"],  # onelo_sk_live_*
    feature_environment=os.environ.get("ONELO_FEATURE_ENVIRONMENT"),  # "test" in staging, "live"/unset in prod
    api_url="https://api.onelo.tools",
    # Optional: ONELO_INSTANCE_ID for a stable instance identity in
    # containers (else a per-process id is generated + persisted).
    # Optional knobs (sensible defaults — uncomment to override):
    # app_version="1.0.0",     # surfaced in SDK telemetry
    # strategy="auto",         # "auto" | "sse" | "polling" (SDK-side SSE fallback)
)

# Optional: register all known features upfront so they appear in
# the dashboard registry without waiting for code paths to execute.
onelo.features.declare([
    # add your feature names here, e.g. "chat-stream", "voice-stream"
])

# Recommended: block until first refresh so the very first request
# already sees real feature state (avoids 'hidden' fail-closed defaults).
# Both forms are equivalent — pick whichever reads better in your code.
onelo.features.ready(timeout=2.0)   # parity with Swift's onelo.features.ready()
# onelo.ready(timeout=2.0)          # alias on the client
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=python field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
# ── Usage (inside your route handler) ───────────────────────────
# ⚠️ identify() sets ONE process-global identity and the SDK swaps its
# whole cache to that user's targeted snapshot (full SSE reconnect on
# each switch). It is NOT per-request state. Use it only when the
# entire process acts as a single user — CLI tools, worker jobs,
# single-tenant services. In a multi-user backend, calling
# identify(user.id) per request would race: concurrent requests share
# the one identity, so user A can be evaluated with user B's targeting.
#
# Multi-user backend rule of thumb:
#   • Global flags (no per-user / per-plan targeting): just call
#     feature(name) — no identify() at all. This is the common case.
#   • Per-user / per-plan targeting server-side: use for_user(user_id)
#     (below). It is STATELESS and multi-user-safe — resolves THIS
#     user's plan-gated features without touching the global identity,
#     so concurrent requests can't race.
#
# Heads-up: if a plan-gated feature (status upsell/greyed) is read through
# the global feature() path, the SDK logs a one-time warning pointing you to
# for_user() — it's evaluating the gate against the shared identity, not the
# request's user. Silence with ONELO_SUPPRESS_GATING_WARNING=1 if intentional
# (e.g. a single-user CLI showing a teaser).

# Global flag (no targeting) — the common case:
@app.get("/export")
async def export(user = Depends(current_user)):
    if not onelo.features.feature("advanced-export").is_enabled:
        raise HTTPException(404)
    # ... run the feature

# Per-user / per-plan gating — resolve for THIS request's user. Cached
# per-user for ~30s so a busy backend (HTTP handlers, WebSocket servers)
# doesn't re-hit the network every request. Fail-closed: a network error
# resolves every feature to hidden, never raises.
@app.websocket("/ws/face")
async def face(ws, user = Depends(current_user)):
    uf = await onelo.features.for_user(user.id)
    if not uf.feature("face-stream").is_enabled:
        await ws.close()
        return
    # ... stream

# Server-rendered upsell — tell the user WHICH plan unlocks a locked feature.
# The backend resolves the plan; you just render its label. Works on both
# feature() and for_user(). upgrade_hint is None when there's nothing to upsell.
@app.get("/reports")
async def reports(user = Depends(current_user)):
    feat = (await onelo.features.for_user(user.id)).feature("advanced-reports")
    if feat.is_enabled:
        return render_reports()
    if feat.upgrade_hint:                       # e.g. "Pro"
        return render_locked(f"Available in {feat.upgrade_hint}", cta=feat.upgrade_cta)
    return render_hidden()

# After YOUR backend processes a plan change (e.g. your own Stripe webhook,
# or a grant/revoke), drop the cached snapshot so the NEXT for_user() is fresh
# immediately instead of waiting out the ~30s TTL:
# onelo.features.invalidate_user(user_id)   # pass nothing to clear every user

# Single-user processes (worker / CLI) may pin an identity once:
# onelo.identify(job_user_id)   # ...and onelo.identify(None) to clear

# Other property checks on the Feature returned by feature(name):
# .is_visible          → True for any visible status; False for "hidden" AND for
#                        any status this SDK build doesn't recognise (fail-closed)
# .is_greyed           → True when status is "greyed"
# .is_new              → True when status is "new"
# .is_beta             → True when status is "beta"
# .is_coming_soon      → True when status is "coming_soon"
# .is_upsell           → True when status is "upsell"
# .is_known            → False if the backend sent a status newer than this SDK
#                        build — a hint to bump the SDK (still fail-closed as hidden)
# .status              → wire status ("enabled" | "new" | "beta" | "coming_soon" | "greyed" | "upsell" | "hidden")
#
# Upsell metadata (attached to plan-gated features — for server-rendered CTAs):
# .upgrade_hint        → human plan label to render, e.g. "Pro"; None when nothing to upsell
# .required_plan_label → same human label as a raw field (.required_plan = machine key)
# .upgrade_cta         → True if you enabled a tap-to-upgrade action for this feature
# .reason              → why this status resolved ("plan" | "user_override" | "paywall_off" | ...)
```
<!-- /onelo:snippet -->
