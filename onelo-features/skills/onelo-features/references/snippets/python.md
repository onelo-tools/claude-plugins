# Python — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=python field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```python
pip install git+https://github.com/onelo-tools/onelo-python.git
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=python field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

# Register EVERY feature name you gate, upfront — not optional. A feature()
# call only registers a name the first time that code path actually runs; a
# route that's rarely hit (an admin panel, a rare branch) can sit gated in
# your code and still never reach the dashboard Registry on a fresh deploy.
# declare() registers the whole list immediately, independent of traffic.
onelo.features.declare([
    # add every feature name you gate, e.g. "chat-stream", "voice-stream"
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
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```python
# ─── STEP 0 — INVENTORY BEFORE YOU GATE (hard precondition) ──────────────────
# Do NOT write a single
#     onelo.features.feature("advanced-export")
# call until the table in step 4 exists and the developer has SEEN it.
# In the examples below the feature name is already chosen
# for you — choosing it is the actual job, and it is the step integrations skip.
# Gating whatever you happened to read produces a registry that LOOKS complete
# while most of the product stays ungated, unsellable and invisible in the
# dashboard.
#
# 1. ENUMERATE — three passes, in this order. Skip these paths entirely:
#       node_modules/  dist/  build/  out/  .next/  vendor/  __pycache__/
#       .build/  DerivedData/  xcuserdata/  Generated/
#       *.test.*  *.spec.*  *_test.*  test/  tests/
#    Skip anything already wrapped in a feature check a few lines above.
#    a) ENTRY POINTS — every route / endpoint / RPC / GraphQL resolver a client can
#       reach, plus every CLI command, scheduled job and queue consumer. These are
#       inherently the right unit; the atom filter does not apply to them.
#    b) CALLERS — everything that invokes an entry point: a frontend fetch, a
#       webhook sender, another service, a cron entry. A caller is NOT its own
#       feature — it INHERITS the entry point's name. One feature = one entry-point
#       row + its linked caller rows, never two names.
#    c) CAPABILITIES — a unit of work with end-to-end logic that is not a route:
#       generate a report, run an export, sync a third party, send a digest.
#       Name the ACTION, not the function that happens to hold it.
#
# 2. QUALIFY. A feature is a unit you could plausibly SELL or GATE — not every
#    function. Route handlers, jobs and consumers: KEEP. Drop the plumbing:
#    middleware, serializers, DB models, migrations, config loaders, health checks,
#    client factories. Internal / debug / admin-only endpoints: ASK before gating —
#    do not gate them by default.
#
#    ⚠ THE EXCEPTION THAT PAYS — a candidate with NO user-facing call site.
#    The rule is that a capability normally needs at least one call site from the
#    UI, so pure plumbing — init paths, persistence, background sync nobody
#    triggers — must NOT become a feature. BUT:
#    on a backend most sellable behaviour has no UI at all,
#    so this is the NORM here, not the edge case.
#    If a candidate changes product BEHAVIOUR — a config boolean, a branch on plan
#    or tier, a per-tenant toggle, a switchable integration (email notifications,
#    webhooks, remove-branding, custom domain, data export) — KEEP it, mark it
#    needs-confirmation, and ASK the developer. Silently filtering one out is the
#    expensive outcome: the developer never learns it was missed. Plumbing with no
#    behavioural effect still gets dropped — that distinction IS the qualify step.
#
# 3. THREAD IT ACROSS THE STACK, THEN SETTLE ONE NAME. A feature rarely lives in
#    one place: a trigger, the destination it opens, and the backend handler where
#    it actually ends are ONE feature. Group them into one thread, e.g.
#        feedback
#          ├─ client   ReportBugButton   (trigger)
#          ├─ client   FeedbackSheet     (destination)
#          └─ backend  routes/feedback   (handler — where it ends)
#    A scan sees an HTTP call, not which endpoint answers it — ASK when the link is
#    not obvious, and show the thread you believe in so it can be corrected.
#
#    ⚠ Onelo keys the registry BY NAME. The same name declared from two platforms
#    becomes ONE entry tagged with both; two DIFFERENT names silently create two
#    features and the tagging never happens. So: one feature = one name, at every
#    point of the thread, on every platform. Never rename between platforms to
#    "make it clearer".
#
#    Then, on every PAID feature, ask: "is there a part of this you want to sell
#    separately?" A sub-feature takes the parent's name plus a hyphen
#    (feedback → feedback-bug). Developers rarely volunteer these, and that is
#    exactly where the upsell lives.
#
# 4. WRITE THE TABLE — before you write any code. One row per feature:
#      feature | file:line | proposed name | destination/trigger/capability | gated / skipped + why
#    Show it to the developer and WAIT for approval. Group by feature, not by file,
#    with sub-features nested under their parent and every needs-confirmation
#    candidate called out.
#
# 5. THEN implement — one THREAD at a time, never one file at a time, and keep the
#    table in the PR description. Gating half a thread is worse than not gating it:
#    the UI hides while the handler stays open, so the feature is off for honest
#    users and on for anyone who knows the URL.
#
# ⚠️ REGISTRATION IS NOT OPTIONAL — call declare() with every name in the table,
#    once, at startup (see "Upfront declaration" below for the exact call). A
#    feature() call only registers a name the FIRST TIME that code path actually
#    RUNS — a feature gated inside a conditionally-rendered component (an
#    early-return, an unselected item, a rarely-hit branch) can sit in your table
#    as "gated" and still never reach the dashboard Registry on a fresh install,
#    because nothing ever called feature() for it. declare() registers the whole
#    table immediately, independent of what happens to render. Do this for EVERY
#    thread you implement in this pass, not just ones you notice are conditional
#    — you cannot always tell from the table alone which destinations are
#    reachable on first load.
#
# COVERAGE RULE: every candidate you enumerated is either gated, or present in the
# table with a stated reason for the skip. "I did not get to it" is not a reason.
# A deliberate skip and an unexamined miss must never look the same in your report.
#
# ─── NAMING CONVENTION ───────────────────────────────────────────────────────
# kebab-case, lowercase, [a-z0-9-] only, max 48 chars. Name the ACTION or the
# DESTINATION — never the widget that happens to open it.
#   Good:  advanced-export · analytics-dashboard · export-recording · settings-window
#   Bad:   export-button · ExportButton · exportBtn · advanced_export · screen2
# Convert PascalCase / camelCase to kebab-case, then strip a trailing suffix:
#   View, Screen, Page, Activity, Fragment, ViewController, ViewModel, Handler,
#   Controller, Service, Widget.
#   AnalyticsDashboardView → analytics-dashboard · ExportHandler → export
# Collision? append -2, -3. Sub-feature? parent name + hyphen: feedback-bug.
# The SETTLED name is what goes in the call: onelo.features.feature("advanced-export")
# — and the SAME string is used at every point of the thread, on every platform.

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
#
# SELF-CHECK before calling this done — verify EVERY status against stubbed
# data, don't eyeball it: enabled/new/beta → is_enabled True, render_reports()
# · greyed/upsell/coming_soon → is_enabled False, upgrade_hint set → locked
# view · hidden → is_enabled False, upgrade_hint None → render_hidden(). No
# upgrade_hint where you expect one? THE SDK NEVER READS THE DRAFT — a status
# set in the dashboard Registry does nothing until Deploy is clicked. Print
# `feat` before assuming a bug; an undeployed status looks identical to one.
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
