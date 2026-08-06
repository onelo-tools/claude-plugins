# Independent verification (Phase 6, Stage 2)

The verifier subagent must be **independent**: it does NOT see the proposal, the
plan, or which sites were just edited. Fresh eyes prevent rubber-stamping and stop
it anchoring on the first scan's grep patterns.

## Dispatch (via `Task`)
- **Small scope:** one subagent over the changed area.
- **Large scope** (a whole backend / app): one subagent per module from the Phase
  1 map, fanned out in parallel; then merge + dedup the returned gap lists.
- Run **once** per apply. No loop — report whatever it finds.

## Prompt — give the subagent ONLY this plus the scope path
> You are independently auditing `<path>` for Onelo Monitor coverage. Assume NO
> prior analysis exists. Find every site that can fail or runs concurrently but is
> NOT covered by `onelo.monitor` (`track` / `capture` / `event`):
>
> 1. **Fallible operations** — awaited calls, network/HTTP **including calls
>    through a stored client** (e.g. `self._client.post(...)`, not only literal
>    `httpx.` / `requests.`), DB queries, disk I/O, subprocess, AI/model calls.
> 2. **Error paths** — `catch` / `except` blocks that swallow without a capture.
> 3. **Background threads / tasks** whose exception would vanish (`Task {}`,
>    `DispatchQueue.async`, `threading.Thread`, `asyncio.create_task`, executors,
>    Celery/RQ tasks).
> 4. **User-facing features with no monitor event at all.** Do this WITHOUT grep:
>    list every UI entry point (button, menu item, form submit, shortcut), every
>    route/screen, startup/teardown paths including loading the user's saved data,
>    and every flow the Onelo SDK presents (sign-in, store, customer portal,
>    consent gate, feedback). A blocking SDK-presented gate has no local `catch`
>    and no `async` in the caller, so no grep will ever surface it. "The SDK
>    handles it" does NOT make it covered — it renders via the SDK but fails
>    inside this app.
> 5. **False greens** — a `track()` whose callback can return `null` / `None` /
>    `[]` / `false` / a cancel sentinel without throwing, or that contains
>    `try?` / `except: pass`. It records ok:true on a broken run. Read the
>    callback body; this is invisible to grep.
>
> Search by MULTIPLE angles — by imported library, by function signature
> (`async` / `throws`), by walking files in the directory, AND by reading the
> UI layer for what a user can actually do — not a single grep.
> A site counts as covered only if a monitor call clearly wraps or reports it.
> Return JSON only, no prose:
> ```json
> { "gaps": [ { "file": "...", "line": 0,
>   "kind": "operation|error|thread|feature|false_green",
>   "symbol": "...", "why": "one line" } ],
>   "covered": 0, "total": 0, "features_unmeasured": ["..."] }
> ```

## Use the result
- `gaps == []` → coverage confirmed; put `covered/total` into the Phase 7 report.
- Anything in `features_unmeasured` that you deliberately excluded belongs in the
  Phase 7 **skip table with its reason** — not dropped. If you can't state why it
  was skipped, it was a miss, not a skip.
- `gaps != []` → list them in the report as REMAINING gaps for the developer to
  action. Do NOT silently fix them under the prior approval — they are a new round
  the developer must see and approve.

## Why no loop
One independent pass is the check. Auto-looping until "clean" risks chasing noise
and over-instrumenting (Rule F). The developer decides whether the returned gaps
warrant another round.
