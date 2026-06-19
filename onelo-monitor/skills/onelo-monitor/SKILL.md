---
name: onelo-monitor
description: Audits and instruments Onelo Monitor SDK usage (error and performance tracking). Use when a developer adds Onelo Monitor to an app, or wants existing monitor instrumentation reviewed for anti-patterns — error-only features that don't auto-resolve, failure-mode event names, event() calls that should be track(), or missing facet meta. Currently covers Swift (iOS/macOS) and Python; other SDKs are deferred.
disable-model-invocation: true
allowed-tools: Bash Glob Grep Read Edit Task
---

# onelo-monitor — audit & instrument

Two jobs, one skill:
- **Audit** existing Onelo Monitor instrumentation and fix anti-patterns.
- **Instrument** a new operation with the correct primitive, name, and meta.

This skill does NOT mass-wrap code. Monitoring is selective by nature —
over-instrumenting creates noise (alert fatigue, write amplification, a dropped
event buffer), which is worse than gaps. Add tracking where a question needs
answering, not everywhere it's possible.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

This is mandatory. Copy the list below into your reply and check items off as you
go. Do NOT jump to detection or instrumentation before Phase 0 is done.

```
- [ ] 0a · SDK installed? (Swift `import OneloSwift` / Python `from onelo import monitor`) — if not, install (references/sdk-setup.md)
- [ ] 0b · Installed version vs LATEST tag — if behind OR unsure, UPDATE before anything else
- [ ] 0c · Smoke-test: app still builds/imports after install/update (swift build / python -c "import <entry>")
- [ ] 1  · Detect language(s) + monitor.init + crash capture → report the map
- [ ] 2  · Choose mode (audit / coverage scan / instrument)
- [ ] 3/4 · Build proposal → WAIT for approval
- [ ] 5  · Apply approved changes
- [ ] 6  · Independent verify (subagent)
- [ ] 7  · Report
```

## Reference files (load ONLY the one(s) for the detected SDK)

- Swift — iOS & macOS: [references/swift.md](references/swift.md)
- Python — backends: [references/python.md](references/python.md)
- SDK install / version / init (Swift & Python): [references/sdk-setup.md](references/sdk-setup.md)

JS (`@onelo/js`), Electron, React Native, Kotlin, Flutter and Go are **not
covered yet** — their Monitor surface may be outdated. If the only SDK present is
one of those, say so and stop (see "Unsupported SDKs"). Never guess their API
from another language.

---

## The model you're optimizing for (read once)

Onelo Monitor turns SDK events into **Feature Health** and **incidents**. Each
event carries a `featureName`, `ok` (true/false), a `source` (event / track /
…), an optional `durationMs`, and `meta`. The dashboard aggregates per
`featureName` over a time window and derives status from the error rate.

Two facts drive every audit rule below:

1. **Recovery needs success traffic.** Auto-resolve fires only when the error
   rate FALLS below threshold while events are still arriving (`check_alerts`
   skips a feature with no events in the window). A feature that emits ONLY
   `ok:false` never produces the `ok:true` events that would lower the rate —
   and once the error stops it goes silent, so the auto-resolve check never runs.
   Such an incident **won't clear itself**; it has to be resolved manually
   (reopen-on-regression makes closing it safe). This is the #1 cause of stale
   incidents.
2. **The name is permanent UI.** The `featureName` shows in Feature Health on
   success AND failure. A name describing the failure (`backend_error`) reads as
   a standing alarm even when everything is healthy.

---

## Audit rule catalogue (language-agnostic — the core of this skill)

These describe the data model and dashboard, not syntax, so they hold for every
SDK. Per-language signals, grep patterns and fix syntax live in the reference
file. Apply in order; severity A → G.

### Rule A — Error-only feature ⛔ (highest priority)
**Signal:** every call site for the feature passes `ok:false` (or it is a
`capture`/global-error-only feature).
**Why:** error rate is pinned at 100% and the feature goes silent once the error
stops → the auto-resolve check never fires → the incident won't clear itself and
must be resolved manually (so these pile up open).
**Fix:** emit the success path too — turn the error-only marker into a `track()`
that wraps the real operation, so the SAME feature reports both ok and error. If
a success path genuinely cannot exist (e.g. a pure crash signal), flag it
`resolve-manually` — never leave it silently.

### Rule B — Failure-mode name ⚠
**Signal:** the name contains/ends with `_error`, `_failed`, `_failure`,
`_denied`, `_missing`, `_timeout`, `_unavailable`, or is `unhandled` used as a
product feature.
**Why:** reads as a permanent alert; almost always co-occurs with Rule A.
**Fix:** rename to the neutral feature/operation and carry the failure in
`ok:false` + `error:"…"`. `permission_denied` → `permission_check`
(ok:false, error:"denied"); `backend_error` → `backend_call`. The old name must
be archived in **Features → Registry** after the rename ships.

### Rule C — event() that should be track()
**Signal:** an instantaneous event wraps (or sits right beside) an operation with
duration — network I/O, disk, an awaited call.
**Why:** you lose the success-rate and latency you'd get for free from `track()`.
**Fix:** wrap the operation in `track()`.

### Rule D — Missing facet meta ◇
**Signal:** an AI/model call without `meta.model`; a triggered action without
`meta.trigger`; a multi-env app without `meta.environment`.
**Why:** these three exact keys become dashboard columns/filters; without them you
can't slice (e.g. "fails only for gpt-4").
**Fix:** add the relevant key(s). Only `model`, `trigger`, `environment` are
special — everything else is plain meta.

### Rule E — _started / _completed split
**Signal:** a pair like `x_started` + `x_completed` (or `_begin`/`_end`).
**Why:** two unrelated rows instead of one operation with a success rate +
duration.
**Fix:** collapse into a single `track("x")` around the operation.

### Rule F — High-frequency event()
**Signal:** an event inside a loop, a render/draw path, or a scroll / mouse-move /
keystroke handler — anything that can fire more than ~1×/second/user.
**Why:** floods the in-memory buffer (oldest events dropped) and amplifies writes.
**Fix:** remove it, or aggregate client-side and emit one summary.

### Rule G — Name not snake_case {object}_{verb}
**Signal:** camelCase, spaces, non-`[a-z0-9_]`, or vague (`data`, `handler`).
**Fix:** `pdf_export`, `checkout`, `ai_response`. Past-tense verb suffix only for
instantaneous events (`tab_viewed`, `plan_selected`); for `track()` name the
FEATURE, not the outcome (`track()` records ok/error/duration itself).

---

## Coverage model (what "quality monitoring" means)

The goal is to catch a feature **both when it works and when it doesn't**, across
every thread. Two different bars:

- **Errors & exceptions — be COMPREHENSIVE.** Every failure path must reach
  Onelo; a silent uncaught exception is the worst outcome. That means: (1) global
  crash capture wired (Swift: automatic at init; Python: `install_excepthook=True`
  + a framework integration); (2) background threads / tasks covered — their
  exceptions vanish if nothing catches them; (3) any `catch` / `except` that
  swallows an error gets a `capture()` (then re-raises if it used to propagate).
- **Operations (success path) — be SELECTIVE.** Wrap SIGNIFICANT operations in
  `track()` — network, AI/model, export, payment, sync, anything awaited or
  fallible. `track()` reports ok AND error + duration for that feature, which is
  exactly "works / doesn't work". Do NOT track trivial UI or high-frequency
  events (Rule F).

Comprehensive on failures, selective on successes — that combination is what makes
the dashboard trustworthy: green means green, and nothing fails in silence.

---

## Phase 0 — Ensure the SDK is present & current (always first)

Before auditing or instrumenting:

1. **Is the SDK installed?** (Swift: `import OneloSwift` + `Onelo(...)`; Python:
   `from onelo import ... monitor`.) If NOT present, install it first —
   [references/sdk-setup.md](references/sdk-setup.md).
2. **Is it current?** The snippets this skill inserts may use newer APIs.
   Notably, Python `monitor.track()` (context manager / decorator) needs
   **onelo-python ≥ 0.5.0a23** — older installs lack it, so the inserted code
   would break. **Find the latest:** `git ls-remote --tags https://github.com/onelo-tools/onelo-python`
   (or `…/onelo-swift`) → highest `*-staging` tag (e.g. `v0.5.0a24-staging`).
   Compare to the installed version (`pip show onelo` / grep `Package.resolved`).
   If installed < latest **OR you can't tell → UPDATE now** (sdk-setup.md) before
   proceeding. Do not instrument a stale SDK.
3. **Smoke-test after ANY install/update.** An SDK upgrade can REMOVE or tighten
   things existing code relies on (not just add APIs) — that's exactly how a
   removed `discovery_key` and a blank `feature_environment` crash-looped Turingo's
   backend. Verify the app still builds/imports BEFORE instrumenting:
   - Swift: `swift build` (or an Xcode build) must succeed.
   - Python: import the app entry, e.g. `python -c "import main"` (your FastAPI
     `module:app`) must not raise.
   If it FAILS, the update broke EXISTING code → fix that first; do not instrument.
4. Only then proceed. Never instrument against an absent, stale, or non-starting SDK.

> Note: "installed & current" (Phase 0) is separate from "Monitor initialised"
> (Phase 1). The SDK can be present for auth yet have no `monitor.init()` — that's
> a Phase 1 gap, fixed via sdk-setup.md's init section.

---

## Phase 1 — Detect (always first)

Survey the ground before touching anything:

1. **Enumerate every language/SDK present.** A repo often has more than one (a
   Swift app + a Python backend). Run each reference file's detect signals and map
   which directories belong to which platform. Report the map.
2. **Confirm Onelo Monitor is initialised** for each (grep for the init). An SDK
   present with no monitor init is the first gap to report.
3. **Verify crash capture is wired** (Swift: auto at init; Python:
   `install_excepthook=True` + integration). Un-wired global capture = unhandled
   errors are invisible — flag it before anything else.
4. Load ONLY the matching reference file(s). If the only SDK present is an
   unsupported one, stop per "Unsupported SDKs".

## Phase 2 — Choose mode

Ask the developer:

```
[1] Audit existing instrumentation   (anti-patterns in what's already there)
[2] Coverage scan                    (find operations / error paths NOT yet monitored)
[3] Instrument one operation         (you name it, I wrap it)
```

Default to **[2]** for a codebase new to Monitor (find the gaps first), **[1]**
for one already heavily instrumented, **[3]** when the developer already knows the
single thing they want. "Both [1]+[2]" is common for a first pass.

## Phase 3 — Audit (mode 1)

1. Run the reference file's grep patterns to list every monitor call site and
   every `featureName`. If the developer supplies app context, optionally enrich
   with the live registry + per-feature error rate via
   `GET {apiBase}/api/monitor/features?app_id=…` — a code-only audit still works.
2. Apply Rules A–G. Build a plan grouped by rule, most severe first (A → G). Treat
   the plan as a verifiable intermediate output the developer signs off on before
   any edit.
3. Show the proposal table (template below): id, `file:line`, current, proposed,
   rule.
4. **Wait for approval.** Commands: `fix #N`, `fix all-names`, `skip #N`,
   `explain #N`, `apply`, `cancel`. Re-render the table after each edit command.

Proposal table template:

```
Audit — {N} features, {M} findings (most severe first)

⛔ ERROR-ONLY · won't auto-resolve, needs manual ({k})
  #1  backend_error          Backend.swift:88   only ok:false   → make track() / resolve-manually
⚠ FAILURE-MODE NAME ({k})
  #2  customer_portal_open_failed  Portal.swift:33   → customer_portal_open (ok:false)
↻ event() → track() ({k})
  #3  settings_model_post     Settings.swift:201    → wrap the POST in track()
◇ MISSING FACET META ({k})
  #4  stt_transcribe          Pipeline.swift:55     → add meta["model"]
✓ {x} features look good
```

## Phase 4 — Coverage scan (mode 2)

Find what SHOULD be monitored but isn't — the reconnaissance pass. Be thorough;
coverage is the whole point of this mode.

1. Run the reference file's **coverage patterns** to enumerate, per language:
   - **operations** worth a `track()` — async/awaited funcs, network/HTTP, DB,
     disk, subprocess, AI/model calls;
   - **error sites** — every `catch` / `except`, `try?` / `try!`, `throw` /
     `raise`;
   - **background threads / tasks** where an exception would vanish (`Task {}`,
     `DispatchQueue.async`, `threading.Thread`, `asyncio.create_task`, executors,
     Celery tasks).
2. Drop any site already instrumented (a `monitor.` call within a few lines).
3. Classify each remaining site (operation → track; swallowing catch → capture;
   uncovered thread/task → wrap or ensure global capture) and assign a snake_case
   name (Rules B + G) + facet meta (Rule D).
4. Show the coverage proposal (template below), grouped by kind — errors and
   threads FIRST (they are the silent-failure risk). **Wait for approval** (same
   commands as Phase 3).

```
Coverage — {covered}/{total} significant sites instrumented

🧵 UNCOVERED THREADS / TASKS · errors vanish here ({k})
  #1  Task {} in SyncEngine.swift:40   throwing body, no capture  → wrap in track("sync")
🔥 SWALLOWED ERRORS ({k})
  #2  catch in Upload.swift:88         error ignored              → capture(error, "upload")
⏱ UNTRACKED OPERATIONS ({k})
  #3  func fetchProfile() async throws Api.swift:21               → track("fetch_profile")
✓ {x} operations already covered
```

## Phase 5 — Instrument & apply

For each approved item (an audit fix, a coverage item, or a single named op):

1. (single named op, mode 3) Ask what to track in plain words; pick the primitive
   from the reference file's decision rules (operation → track; instantaneous →
   event; already-caught error → capture). Derive a snake_case name (Rule G) +
   facet meta (Rule D).
2. Fetch the canonical snippet — do NOT improvise SDK calls:
   ```bash
   curl -sf "${ONELO_API_BASE:-https://app.onelo.tools}/api/snippets?sdk=monitor&lang={lang}"
   ```
   Use its `usage` / `init` fields; fall back to the reference file's snippet only
   if the fetch fails.
3. Insert per the reference file's insertion rules. One change per site; touch
   nothing else; never alter existing error propagation.

## Phase 6 — Verify (independent — ALWAYS)

Verification runs after every apply, in two stages. It is non-negotiable: this is
what makes the audit trustworthy rather than a self-graded checklist.

### Stage 1 — Self-check (close the loop on the plan)
Re-run the Phase 3/4 greps and diff against the approved plan:
- every approved item now has a `monitor.` call in range;
- every renamed name is gone.
Emit a pass/fail line per item. Any fail → fix it and re-check before Stage 2.

### Stage 2 — Independent subagent (fresh eyes — ALWAYS)
Dispatch a subagent (via `Task`) that has NOT seen the plan, the proposal, or
which sites were just edited — so it cannot rubber-stamp and won't anchor on the
first scan's grep patterns. Its only job: find fallible operations / error paths /
background tasks NOT covered by `onelo.monitor`. Exact prompt + structured output:
[references/verification.md](references/verification.md).
- **Small scope:** one subagent over the changed area.
- **Large scope** (e.g. a whole backend): fan out one subagent per module from the
  Phase 1 map; merge + dedup their gap lists.
Run it **once** (no loop). Report whatever gaps it returns as REMAINING gaps —
never silently fix them under the prior approval.

Coverage numbers in the report come from THIS pass, not from self-report.

## Phase 7 — Report

```
✓ {applied} change(s) applied · {skipped} skipped
📊 Coverage (verified): {covered}/{total} operations + {e}/{E} error paths instrumented
⚠ {k} error-only features need a PRODUCT decision (Rule A) — not auto-fixed
🧵 {t} background threads/tasks still uncovered — errors there are invisible
🔎 {g} gaps found by independent verification — listed below for you to action
→ After shipping: archive any renamed old names in Features → Registry
```

Never report success while a Rule-A finding, a swallowed error, an uncovered
thread, or an independent-verification gap is silently unaddressed — list them.

---

## Naming convention (canonical)

snake_case `{object}_{verb}`. Object = what the user acts on (`checkout`,
`pdf_export`, `ai_response`). For `track()` name the FEATURE, not the outcome. The
past-tense `_verb` suffix is for instantaneous events only (`tab_viewed`,
`plan_selected`).

**PITFALL — check-style names:** name the check, not its failure mode:
`permission_check` (ok:false, error:"denied"), NOT `permission_denied`. A
failure-mode name reads as a permanent alert in Feature Health even on success.

## What NOT to instrument

Mouse move / hover / scroll; every keystroke (track the submit instead); internal
UI toggles with no outcome; anything faster than ~1×/second/user; init /
persistence / background work with no user-facing outcome; PII in meta.

## Unsupported SDKs (deferred)

JavaScript (`@onelo/js`), Electron, React Native, Kotlin, Flutter and Go are not
covered in this version — their Monitor surface may be outdated. If one of those
is the only SDK present, tell the developer it isn't supported yet and stop. Do
not guess the API from Swift or Python.
