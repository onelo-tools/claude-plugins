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
>
> Search by MULTIPLE angles — by imported library, by function signature
> (`async` / `throws`), and by walking files in the directory — not a single grep.
> A site counts as covered only if a monitor call clearly wraps or reports it.
> Return JSON only, no prose:
> ```json
> { "gaps": [ { "file": "...", "line": 0, "kind": "operation|error|thread",
>   "symbol": "...", "why": "one line" } ], "covered": 0, "total": 0 }
> ```

## Use the result
- `gaps == []` → coverage confirmed; put `covered/total` into the Phase 7 report.
- `gaps != []` → list them in the report as REMAINING gaps for the developer to
  action. Do NOT silently fix them under the prior approval — they are a new round
  the developer must see and approve.

## Why no loop
One independent pass is the check. Auto-looping until "clean" risks chasing noise
and over-instrumenting (Rule F). The developer decides whether the returned gaps
warrant another round.
