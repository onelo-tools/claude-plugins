---
name: onelo-monitor
description: Audits and instruments Onelo Monitor SDK usage (error and performance tracking) across every user-facing feature of an app. Use whenever a developer adds, pastes, wires or mentions Onelo Monitor, asks to track errors/crashes/performance/feature health with Onelo, asks which parts of their app should be monitored, or wants existing monitor instrumentation reviewed for anti-patterns — error-only features that don't auto-resolve, operations that resolve empty-handed but log ok:true, failure-mode event names, event() calls that should be track(), or missing facet meta. Also use after inserting the Monitor snippet, to make sure coverage is product-complete rather than one demo call. Covers Swift (iOS/macOS), Python, JavaScript/TypeScript, Node, PHP, Electron, React Native, Android/Kotlin and Flutter.
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
- [ ] 0a · SDK installed? (Swift `import OneloSwift` / Python `from onelo import monitor` / JS `import { Onelo } from '@onelo/js'`) — if not, install (references/sdk-setup.md)
- [ ] 0b · Installed version vs LATEST tag — if behind OR unsure, UPDATE before anything else
- [ ] 0c · Smoke-test: app still builds/imports after install/update (swift build / python -c "import <entry>")
- [ ] 1  · Detect language(s) + monitor.init + crash capture → report the map
- [ ] 2  · Choose mode (audit / coverage scan / instrument)
- [ ] 3/4 · List every USER-FACING feature FIRST (incl. SDK-presented flows), then build the proposal — each feature ends covered / gap / skipped+reason → WAIT for approval
- [ ] 5  · Apply approved changes
- [ ] 6  · Independent verify (subagent)
- [ ] 7  · Report
```

## Snippets — the code that gets inserted

One file per platform. Open ONLY the one you detected.

<!-- snippet-index:start -->
| Platform | Snippet file |
|---|---|
| Android — Kotlin | [`references/snippets/android.md`](references/snippets/android.md) |
| Electron | [`references/snippets/electron.md`](references/snippets/electron.md) |
| Flutter — Dart | [`references/snippets/flutter.md`](references/snippets/flutter.md) |
| JavaScript / TypeScript | [`references/snippets/javascript.md`](references/snippets/javascript.md) |
| React Native | [`references/snippets/react-native.md`](references/snippets/react-native.md) |
| Swift — iOS | [`references/snippets/swift-ios.md`](references/snippets/swift-ios.md) |
| Node.js — backend | [`references/snippets/node.md`](references/snippets/node.md) |
| PHP — backend | [`references/snippets/php.md`](references/snippets/php.md) |
| Python — backend | [`references/snippets/python.md`](references/snippets/python.md) |
<!-- snippet-index:end -->

## Reference files (load ONLY the one(s) for the detected SDK)

- Swift — iOS & macOS: [references/swift.md](references/swift.md)
- Python — backends: [references/python.md](references/python.md)
- JavaScript / TypeScript — web & SSR/Node (`@onelo/js`): [references/js.md](references/js.md)
- Electron, React Native, Android/Kotlin, Flutter: [references/other-platforms.md](references/other-platforms.md)
- SDK install / version / init (Swift, Python & JS): [references/sdk-setup.md](references/sdk-setup.md)

**Every platform in the snippet table above is supported.** Node and PHP have no
per-language reference file yet — for those, use the audit rules below (they are
language-agnostic) plus the platform's snippet file for exact syntax, and say so
in your report. Never guess an API from another language: if a call shape isn't in
the platform's snippet file or its reference file, read
`packages/onelo-<sdk>/` instead of improvising.

---

## The model you're optimizing for (read once)

Onelo Monitor turns SDK events into **Feature Health** and **incidents**. Each
event carries a `featureName`, `ok` (true/false), a `source` (event / track /
…), an optional `durationMs`, and `meta`. The dashboard aggregates per
`featureName` over a time window and derives status from the error rate.

Three facts drive every audit rule below:

1. **Recovery needs success traffic.** Auto-resolve fires only when the error rate
   FALLS below threshold while events are still arriving (`check_alerts` skips a
   feature with no events in the window). A feature emitting ONLY `ok:false` never
   produces the `ok:true` events that would lower the rate, and goes silent once
   the error stops — so the incident **won't clear itself** and must be resolved
   manually. The #1 cause of stale incidents.
2. **The name is permanent UI.** `featureName` shows in Feature Health on success
   AND failure, so a name describing the failure (`backend_error`) reads as a
   standing alarm even when everything is healthy.
3. **The dashboard believes what you send.** Nothing infers failure — an operation
   only counts as failed if you say so. A feature that isn't instrumented is
   absent, and one instrumented wrongly is worse: it asserts health it never
   checked (Rule A2). Green must be earned.

---

## Audit rule catalogue (language-agnostic — the core of this skill)

These describe the data model and dashboard, not syntax, so they hold for every
SDK. Per-language signals, grep patterns and fix syntax live in the reference
file. Apply in order; severity A → A2 → B → … → G. (A2 shares Rule A's ⛔ tier —
both make the dashboard lie — and is applied immediately after A. It is numbered
A2 rather than inserted as a new "B" so the existing rule letters, which the
reference files cite, stay stable.)

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

### Rule A2 — Resolved ≠ succeeded (false green) ⛔
**Signal:** a `track()` whose callback can finish WITHOUT doing the thing — it
returns `nil` / `null` / `None`, an empty list, zero rows, `false`, or a
"user cancelled" sentinel — and nothing inside the callback raises.
**Why:** `track()` sets `ok:false` ONLY when the callback throws. An operation
that returns empty-handed is recorded as a **success**, so a wholly broken feature
shows 100% healthy and no incident ever opens. This is worse than no
instrumentation, because the dashboard now actively asserts the feature works.
**Real case:** `loadAuthView()` resolves `null` both when the user dismisses the
sheet AND when the embed never paints — a sign-in that never rendered logged
`ok:true`.
**Fix:** decide inside the callback and **raise/throw there**, then handle it
outside the `track()` — so the ONE feature carries both outcomes:
```
track("sign_in") {
  let session = try await loadAuthView()      // resolves nil on BOTH paths
  guard let session else { throw AuthFailed.noSession }   // ← raise INSIDE
  return session
}
```
Let only genuine breakage throw where you can tell it from a user-cancel (a
cancel flag, a timeout, "did the view ever appear"); if you can't, throw anyway
and carry the ambiguity in `meta` — a false green costs more than a noisy rate.
**NEVER** emit a second `event("sign_in_failed", ok:false)` for the failure path —
that is Rules A+B in one move and splits one feature into two dashboard rows.

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
every thread — and to know which features you've made that claim about. Three bars:

- **Features — be EXHAUSTIVE in the INVENTORY.** Enumerate every user-facing
  feature before deciding what to instrument (Phase 4 step 1). You may then skip
  many of them, but each skip is explicit and reasoned. Silence about a feature
  is the failure mode this whole skill exists to prevent.

- **Errors & exceptions — be COMPREHENSIVE.** Every failure path must reach
  Onelo. That means: (1) global crash capture wired (Swift: auto at init; Python:
  `install_excepthook=True` + a framework integration); (2) background threads /
  tasks covered — their exceptions vanish if nothing catches them; (3) any
  `catch` / `except` that swallows an error gets a `capture()` (then re-raises if
  it used to propagate); (4) operations that fail by returning EMPTY rather than
  throwing — see Rule A2.
- **Operations (success path) — be SELECTIVE.** Wrap SIGNIFICANT operations in
  `track()` — network, AI/model, export, payment, sync, anything awaited or
  fallible. Do NOT track trivial UI or high-frequency events (Rule F).

Exhaustive in the inventory, comprehensive on failures, selective on successes —
that is what makes the dashboard trustworthy: green means green, nothing fails in
silence, and every gap is one you chose.

---

## Phase 0 — Ensure the SDK is present & current (always first)

Before auditing or instrumenting:

1. **Is the SDK installed?** (Swift: `import OneloSwift` + `Onelo(...)`; Python:
   `from onelo import ... monitor`; JS: `import { Onelo } from '@onelo/js'` +
   `new Onelo(...)`.) If NOT present, install it first —
   [references/sdk-setup.md](references/sdk-setup.md).
2. **Is it current?** The snippets this skill inserts may use newer APIs — e.g.
   Python `monitor.track()` needs **onelo-python ≥ 0.5.0a23**, older installs lack
   it and the inserted code breaks. Find the latest with
   `git ls-remote --tags https://github.com/onelo-tools/onelo-<sdk>` → highest
   `*-staging` tag; compare to the installed version. If behind **or you can't
   tell → UPDATE now** (sdk-setup.md). Do not instrument a stale SDK.
3. **Smoke-test after ANY install/update.** An upgrade can REMOVE or tighten
   things existing code relies on — that's how a removed `discovery_key` and a
   blank `feature_environment` crash-looped Turingo's backend. Verify the app
   still builds/imports first (Swift: `swift build`; Python: `python -c "import
   main"`). If it FAILS, the update broke EXISTING code → fix that first.
4. Only then proceed. Never instrument against an absent, stale, or non-starting SDK.

> Note: "installed & current" (Phase 0) is separate from "Monitor initialised"
> (Phase 1). The SDK can be present for auth yet have no `monitor.init()` — that's
> a Phase 1 gap, fixed via sdk-setup.md's init section.

---

## Keys — which one, and where it comes from

| Side | Key | Where |
|---|---|---|
| Client / frontend | **publishable** `onelo_pk_live_…` — a public app identifier, safe to ship | dashboard → **SDK** tab, top |
| Server / backend | **secret** `onelo_sk_live_…` — a trusted credential | dashboard → **API Keys → Secret keys** |

A backend authenticates as the server, not as an app, so it uses the secret key —
never a publishable one. Read it from the environment (`ONELO_SECRET_KEY`); never
hard-code it and never let it reach client code. Ask for the key you need
**before** proposing changes.

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
4. Load ONLY the matching reference file(s). If a detected platform has no
   reference file (Node, PHP), do NOT stop — proceed per "Platform coverage".

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
2. Apply Rules A–G (including A2 — for every existing `track()`, ask "can this
   callback return empty-handed without throwing?"; grep alone won't show it, you
   must read the callback body). Build a plan grouped by rule, most severe first
   (A → A2 → B → … → G). Treat
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
⛔ FALSE GREEN · resolves empty-handed, logs ok:true ({k})
  #2  sign_in                AuthView.swift:24  nil session not thrown → guard + throw INSIDE the callback
⚠ FAILURE-MODE NAME ({k})
  #3  customer_portal_open_failed  Portal.swift:33   → customer_portal_open (ok:false)
↻ event() → track() ({k})
  #4  settings_model_post     Settings.swift:201    → wrap the POST in track()
◇ MISSING FACET META ({k})
  #5  stt_transcribe          Pipeline.swift:55     → add meta["model"]
✓ {x} features look good
⊘ {s} deliberately skipped — see the skip list in the Phase 7 report
```

## Phase 4 — Coverage scan (mode 2)

Find what SHOULD be monitored but isn't — the reconnaissance pass. Be thorough;
coverage is the whole point of this mode.

1. **Enumerate USER-FACING features FIRST — before any grep.** Greps are
   code-shaped: they find `async` / `catch` / fire-and-forget. A whole class of
   features has none of those signals (a blocking, SDK-presented consent gate has
   no local `catch` at all) and falls straight through the net — that is exactly
   how a real integration covered 5 of 13 features while every grep came back
   clean. Build the product-shaped list by READING the app:
   - every **UI entry point** — button, menu item, form submit, shortcut,
     drag-drop, context menu, deep link;
   - every **route / screen / view** the user can reach;
   - everything **crossing a boundary** — network, disk, DB, clipboard, file
     picker, camera, notifications, IPC, third-party SDK;
   - **startup and teardown** — first launch, restore/load of the user's data,
     migration, sign-out, delete, quit;
   - **the flows the Onelo SDK itself presents** — sign-in/sign-up, store and
     checkout, customer portal, consent gate, feedback form. "The SDK handles it"
     is **not** a reason to leave these unmeasured: they *render* via the SDK but
     *fail inside the developer's app* — a bad `apiUrl`, a blocked origin, a
     webview that never paints. Onelo cannot see that from its side; only an event
     from the app can. These are also the likeliest false greens (Rule A2).

   Carry this list through the whole phase. Each entry ends in exactly one state —
   **covered**, **gap**, or **skipped + reason**. Never looked at = gap, not skip.
2. Run the reference file's **coverage patterns** to enumerate, per language:
   - **operations** worth a `track()` — async/awaited funcs, network/HTTP, DB,
     disk, subprocess, AI/model calls;
   - **error sites** — every `catch` / `except`, `try?` / `try!`, `throw` /
     `raise`;
   - **background threads / tasks** where an exception would vanish (`Task {}`,
     `DispatchQueue.async`, `threading.Thread`, `asyncio.create_task`, executors,
     Celery tasks).
   Merge these hits INTO the step-1 list — they add sites the product list
   missed, they do not replace it.
3. Drop any site already instrumented (a `monitor.` call within a few lines).
4. Classify each remaining site (operation → track; swallowing catch → capture;
   uncovered thread/task → wrap or ensure global capture) and assign a snake_case
   name (Rules B + G) + facet meta (Rule D). Anything you are deliberately NOT
   instrumenting goes in the **skipped** list with a one-line reason — never drop
   it silently, or it becomes indistinguishable from something you never saw.
5. Show the coverage proposal (template below), grouped by kind — errors and
   threads FIRST (they are the silent-failure risk). **Wait for approval** (same
   commands as Phase 3).

```
Coverage — {f} user-facing features: {covered} covered · {gaps} gaps · {skipped} skipped
            ({covered_sites}/{total_sites} code sites instrumented)

🧵 UNCOVERED THREADS / TASKS · errors vanish here ({k})
  #1  Task {} in SyncEngine.swift:40   throwing body, no capture  → wrap in track("sync")
🔥 SWALLOWED ERRORS ({k})
  #2  catch in Upload.swift:88         error ignored              → capture(error, "upload")
⏱ UNTRACKED OPERATIONS ({k})
  #3  func fetchProfile() async throws Api.swift:21               → track("fetch_profile")
🚪 UNMEASURED USER-FACING FLOWS ({k})
  #4  consent gate            App.tsx:61    SDK-presented, blocking, no local catch → track("consent_gate")
  #5  notes load on launch    Store.ts:14   silent empty list on failure            → track("notes_load") + Rule A2
✓ COVERED ({x})
  notes_save · checkout · pdf_export
⊘ DELIBERATELY SKIPPED ({s}) — each needs a reason
  theme toggle       Rule F — no outcome, fires per click, nothing can fail
  scroll position    Rule F — high-frequency, restore failure is invisible to the user
```

The three lists must account for **every** entry from step 1. If the numbers
don't add up (`covered + gaps + skipped ≠ f`), you have entries you never
classified — go back to step 1 rather than shipping the table.

## Phase 5 — Instrument & apply

For each approved item (an audit fix, a coverage item, or a single named op):

1. (single named op, mode 3) Ask what to track in plain words; pick the primitive
   from the reference file's decision rules (operation → track; instantaneous →
   event; already-caught error → capture). Derive a snake_case name (Rule G) +
   facet meta (Rule D).
2. Take the canonical snippet from
   the snippet file for this platform (table at the top of this skill) — do NOT improvise SDK calls
   and do NOT adapt another language's snippet. That file ships inside this
   plugin, baked from `@onelo/snippets` at publish time, so it is exactly what
   the dashboard and /docs show. No network call needed.
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
first scan's grep patterns. Its job: find fallible operations, error paths,
background tasks, **unmeasured user-facing features and false greens** not covered
by `onelo.monitor`. Exact prompt + output:
[references/verification.md](references/verification.md).
- **Small scope:** one subagent over the changed area. **Large scope** (a whole
  backend): one per module from the Phase 1 map; merge + dedup the gap lists.
Run it **once** (no loop). Report whatever it returns as REMAINING gaps — never
silently fix them under the prior approval.

Coverage numbers in the report come from THIS pass, not from self-report.

## Phase 7 — Report

```
✓ {applied} change(s) applied
📊 Coverage (verified): {covered}/{f} user-facing features · {covered_sites}/{total_sites} operations · {e}/{E} error paths
⚠ {k} error-only features need a PRODUCT decision (Rule A) — not auto-fixed
⛔ {a2} features may log a FALSE GREEN (Rule A2) — listed below
🧵 {t} background threads/tasks still uncovered — errors there are invisible
🔎 {g} gaps found by independent verification — listed below for you to action

⊘ DELIBERATELY SKIPPED ({s}) — nothing here is an oversight:
  | Feature / site | Why skipped |
  |---|---|
  | theme toggle   | Rule F — no outcome, nothing can fail |
  | {…}            | {reason} |

→ After shipping: archive any renamed old names in Features → Registry
```

The skip table is **mandatory**, even when empty (`⊘ none`). Without it, a feature
you consciously excluded and one you never looked at read identically — which is
precisely how partial coverage passes for complete. Note here too if a detected
platform had no reference file: its coverage number is a floor, not a ceiling.

Never report success while a Rule-A or Rule-A2 finding, a swallowed error, an
uncovered thread, an unmeasured user-facing flow, or an independent-verification
gap is silently unaddressed — list them.

---

## Naming convention (canonical)

snake_case `{object}_{verb}`; object = what the user acts on (`checkout`,
`pdf_export`, `ai_response`). For `track()` name the FEATURE, not the outcome;
the past-tense suffix is for instantaneous events only (`tab_viewed`).
**PITFALL — check-style names:** name the check, not its failure mode:
`permission_check` (ok:false, error:"denied"), NOT `permission_denied` — a
failure-mode name reads as a permanent alert even on success.

## What NOT to instrument

**The test is not "is this background work?" — it is "would a FAILURE here be
visible to the user?"** If yes, instrument it, however quiet or infrastructural it
looks. Loading a user's saved data at startup is "background persistence" by every
code-shaped definition, and it is also the highest-value event in the app: when it
fails silently the user sees an empty screen and concludes their work is gone.
Same for a background sync, a token refresh, a migration, an autosave — nobody
clicked them, and each ruins the user's day when it breaks. Silent data loss is
never out of scope.

Genuinely skip only:
- **High-frequency signals** — mouse move / hover / scroll, every keystroke (track
  the submit instead), anything faster than ~1×/second/user (Rule F).
- **Things that cannot fail** — a pure in-memory UI toggle, a local theme switch:
  no boundary crossed, no failure mode, nothing to report.
- **Work whose failure the user can never perceive** — a cache warm that silently
  falls through to the real fetch, a metrics ping. If a failure would degrade or
  change ANYTHING the user sees, it does not belong here.
- **PII in meta** — never, regardless of value.

Everything you skip goes in the Phase 4 / Phase 7 skip list with its reason.

## Platform coverage

All nine platforms in the snippet table are supported — Swift, Python, JS/TS,
Node, PHP, Electron, React Native, Android/Kotlin and Flutter. Depth differs:

| Depth | Platforms | What you have |
|---|---|---|
| Full reference file | Swift, Python, JS/TS | detect signals, primitives, grep + coverage patterns, insertion rules |
| Shared reference file | Electron, React Native, Android/Kotlin, Flutter | [references/other-platforms.md](references/other-platforms.md) — same sections, condensed |
| Snippet only | Node, PHP | the snippet file + the language-agnostic rules A–G |

For a snippet-only platform run the phases normally (the rules are
language-agnostic), take exact syntax from the snippet file, and note in Phase 7
that no per-language grep set exists yet — the coverage number is a floor, not a
ceiling. Never stop because a platform lacks a reference file; never guess an API
shape — read `packages/onelo-<sdk>/` if the snippet doesn't show it.
