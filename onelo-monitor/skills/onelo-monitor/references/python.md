# Onelo Monitor — Python (backends)

Per-language signals, primitives, grep patterns and fix syntax. The
language-agnostic audit rules (A–G) live in SKILL.md.

## Contents
- Detect signals
- Primitives (track / capture_exception / capture_message / capture_event)
- Auto-instrumentation (integrations — prefer these)
- Grep patterns
- Coverage scan patterns
- Insertion rules
- Crash capture (excepthook)
- Snippet source (baked in this plugin)
- Good vs bad examples

## Detect signals
- **SDK:** `from onelo import monitor` + a `monitor.init(...)` call. Package:
  `onelo-python`.
- **Framework** (decides which integration to recommend): FastAPI / Starlette /
  Litestar (ASGI), Django, Flask (WSGI). Read deps from `pyproject.toml` /
  `requirements.txt`.

## Primitives — pick the right one
| Want to record | Use | Notes |
|---|---|---|
| an operation (duration / can fail) | `monitor.track("name", meta=…)` | Context manager (`with` / `async with`) or decorator. Measures duration, emits ok or error, **re-raises**. (Added 0.5.0a23.) |
| an exception you caught | `monitor.capture_exception(exc=None, feature_name=…)` | call inside an `except` block; full stack + chained cause |
| a free-form message | `monitor.capture_message("…", level=…)` | `level` error/fatal → `ok=false` |
| a pre-built event (rare) | `monitor.capture_event(MonitorEvent(...))` | low-level; integrations use it |

**There is no `event()` in Python.** For an instantaneous `ok=true` marker, use
`with monitor.track("name"):` around the (trivial) block, or
`monitor.capture_event(MonitorEvent(feature_name="name", ok=True))`. Prefer
`track()` for anything that actually does work.

`monitor.track` forms:
```python
with monitor.track("checkout", meta={"plan": plan}):
    process_payment()

async with monitor.track("ai_response", meta={"model": "gpt-4"}):
    await call_model()

@monitor.track("pdf_export")
async def export(doc): ...
```

## Auto-instrumentation (prefer over manual where it exists)
Before hand-instrumenting, wire the integration — it captures unhandled errors and
request context for free, per request, with scope isolation:
- ASGI (FastAPI / Starlette / Litestar): `OneloMonitorASGIMiddleware`
- Django: `onelo.monitor.integrations.django.OneloMonitorMiddleware`
- Flask / WSGI: `onelo.monitor.integrations.flask.install(app)`
- Outgoing HTTP breadcrumbs: `integrations.httpx.wrap_async_transport()` /
  `integrations.requests.install_session(...)`
- logging → monitor: `integrations.logging.OneloLoggingHandler`

`monitor.init(install_excepthook=True)` (the default) wires sys / threading /
asyncio excepthooks, so truly-unhandled errors are reported even without a
framework handler.

## Grep patterns
Call sites:
```bash
grep -rnE "monitor\.(track|capture_exception|capture_message|capture_event)\(" --include=*.py .
```
Manual timing that should be `track()` (the anti-pattern `track()` replaces):
```bash
grep -rnE "time\.(time|perf_counter)\(\)" --include=*.py .
```
Feature names:
```bash
grep -rhoE "monitor\.track\(\s*\"[a-zA-Z0-9_]+\"|feature_name=\"[a-zA-Z0-9_]+\"" --include=*.py .
```
Init present + integrations wired:
```bash
grep -rnE "monitor\.init\(|OneloMonitor(ASGI|)Middleware|integrations\." --include=*.py .
```

## Coverage scan patterns (Phase 4)
Operations worth `track()` (await / network / DB / subprocess):
```bash
grep -rnE "async def |await |httpx\.|requests\.|aiohttp|urllib|psycopg|sqlalchemy|subprocess\.|\.execute\(" --include=*.py .
```
Error sites (where failures should be captured):
```bash
grep -rnE "except |raise " --include=*.py .
```
Background threads / tasks where an exception vanishes:
```bash
grep -rnE "threading\.Thread|asyncio\.create_task|ThreadPoolExecutor|ProcessPoolExecutor|run_in_executor|@(celery_app\.task|shared_task|app\.task)" --include=*.py .
```
Already instrumented — skip:
```bash
grep -rnE "monitor\.(track|capture_exception|capture_message|capture_event)\(|integrations\." --include=*.py .
```
**Quality note:** request-level errors are best covered by the ASGI/WSGI
integration — hand-instrument only OPERATIONS inside handlers and BACKGROUND work
the integration can't see (Celery/RQ tasks, `create_task`, executors). An
`asyncio.create_task(...)` whose exception nobody awaits surfaces only via the loop
excepthook — wrap its body in `track()` for a named feature.

## Insertion rules
- **track()** context manager wraps the operation block — keep the body, indent it
  under `with monitor.track(...):` (or `async with` for awaited work). For whole
  functions use the `@monitor.track("name")` decorator.
- **capture_exception()** goes inside the existing `except` block (no args = the
  current exception). Do NOT add a bare `except` just to capture — let the
  excepthook / integration catch genuinely-unhandled errors.
- Never swallow: if the code used to propagate, `capture_exception()` then
  `raise`. (`track()` re-raises for you.)

## Crash capture
`install_excepthook=True` (init default) installs `sys.excepthook`,
`threading.excepthook`, and the asyncio loop exception handler — no per-handler
wiring needed.

## Snippet source (Phase 4)

The exact API shape lives in the snippet file for this platform — see the
platform table in SKILL.md. It is baked into this plugin
from `@onelo/snippets` at publish time. Take `install` / `init` / `usage` from
there and replace the feature name.

Never write the call from memory and never adapt another language's snippet —
the "Good vs bad" examples below are for judging NAMES and PRIMITIVES, not for
copying syntax.

## Good vs bad (examples pattern)
- `start = time(); …; capture_event(MonitorEvent(ok=True, duration_ms=…))` → `with monitor.track("op"):`  *(Rule C)*
- `try: … except: monitor.capture_message("failed", level="error")` → `monitor.capture_exception()` (keeps the stack)
- error-only `capture_message("backend down", level="error", feature_name="backend_error")` → `with monitor.track("backend_call"):` (emits ok too)  *(Rules A+B)*
- `with monitor.track("ai_response"):` with no meta → add `meta={"model": "gpt-4"}`  *(Rule D)*
