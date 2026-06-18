# Python Patterns

Detection rules, grep patterns, insertion points, and snippet templates for
Python backend projects (FastAPI, Django, Flask, aiohttp).

## Contents
- Python-specific classification rules
- Grep patterns
- Symbol extraction
- Insertion point
- Snippets
- Trigger detection
- Codegen for `FEATURE_REGISTRY`
- Project structure assumptions

## Python-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Backend Python is *all* server-side, so the "atom vs. screen" split looks
different from frontend frameworks: there are no buttons, cells, or chips
to drop. The unit of instrumentation is a **route handler** (HTTP entry
point). Background workers, helper functions, and ORM model methods are
plumbing — never instrument them.

Treat each of these as a `screen` (always keep — Phase 2.5 Rule 1 applies):

- **FastAPI route handler.** Function decorated with `@app.<verb>(...)`,
  `@router.<verb>(...)` (APIRouter), or `@<router>.api_route(...)`.
- **Flask route handler.** Function decorated with `@app.route(...)` or
  `@<blueprint>.route(...)`.
- **Django function-based view (FBV).** Top-level function in `views.py`
  whose first positional arg is `request`. Cross-check with `urls.py`
  references when ambiguous.
- **Django class-based view (CBV).** Class subclassing `View`, `APIView`,
  `ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`,
  `TemplateView`, or any DRF generic view.
- **aiohttp handler.** `async def handler(request):` referenced via
  `app.router.add_<verb>(...)` or decorated with `@routes.<verb>(...)`.

### Python-only atoms to drop

On top of the generic Phase 2.5 list, drop these even when they live in
`views.py` / `routers.py`:

- **Helper functions** (`def _internal_helper(...)`, `def get_or_404(...)`)
  — leading underscore or no `request`/`Depends` parameter.
- **Pydantic models / dataclasses / TypedDicts** — request/response shape
  definitions, never an entry point.
- **Django model methods** (`def save(self, ...)`, `def __str__(self)`) —
  ORM plumbing.
- **Django middleware classes** — apply to every request; instrumenting
  them would gate the whole app.
- **Celery tasks, RQ jobs, APScheduler callbacks, FastAPI background
  tasks** — async workers, not user-visible features. (See "Trigger
  detection" below for when these matter as *callers*.)
- **`__init__.py` factory functions** (`def create_app():`) — bootstrap
  code, gates the entire app.

## Grep patterns

Run each pattern. Collect file + line number + handler signature.

```bash
# FastAPI route handlers (app or router instance)
grep -rnE '@(app|router|[a-z_]+_router)\.(get|post|put|patch|delete|head|options|api_route)\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  --exclude-dir=migrations --exclude-dir=tests --exclude-dir=test \
  . 2>/dev/null

# Flask route handlers (app or blueprint)
grep -rnE '@(app|[a-z_]+_(blueprint|bp)|bp)\.route\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  . 2>/dev/null

# Django function-based views (heuristic: top-level def with `request` first arg)
grep -rnE '^(def|async def) [a-z_][a-z0-9_]*\(request' \
  --include='views.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  . 2>/dev/null

# Django class-based views (subclasses of View / generic / DRF)
grep -rnE '^class [A-Z][A-Za-z0-9]*\((View|APIView|ListView|DetailView|CreateView|UpdateView|DeleteView|TemplateView|GenericAPIView|ListAPIView|RetrieveAPIView|CreateAPIView|UpdateAPIView|DestroyAPIView|ViewSet|ModelViewSet|ReadOnlyModelViewSet)' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  --exclude-dir=migrations \
  . 2>/dev/null

# aiohttp decorator-based handlers
grep -rnE '@routes\.(get|post|put|patch|delete|head|options|view)\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  . 2>/dev/null

# aiohttp imperative registration (router.add_get etc.)
grep -rnE '\.router\.add_(get|post|put|patch|delete|head|options|route)\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=venv \
  . 2>/dev/null
```

**Skip paths:** `__pycache__/`, `.venv/`, `venv/`, `env/`, `.tox/`,
`migrations/`, `tests/`, `test/`, `*_test.py`, `test_*.py`, `*.pyc`,
`.pytest_cache/`, `build/`, `dist/`, `.eggs/`, `*.egg-info/`,
`site-packages/`.

For a `@routes.<verb>` or `@app.<verb>` match at line N, the function
declaration is on line N+1 (or N+2 if multiple decorators stack). Read
forward to capture the function name.

## Symbol extraction

Derive a kebab-case feature name from each match:

| Source | Example | Feature name |
|---|---|---|
| Route path (preferred) | `@app.get("/api/chat/stream")` | `chat-stream` |
| Route with version prefix | `@router.post("/v1/users/messages")` | `users-messages` |
| Path params dropped | `@app.get("/users/{user_id}/messages")` | `users-messages` |
| Function name fallback | `def list_messages(request):` (Django) | `list-messages` |
| Class name fallback | `class MessageListView(ListView):` | `message-list` |
| aiohttp imperative | `app.router.add_get("/health", health_check)` | `health` |

### Rules

1. **Prefer the route path** when present — strip leading `/`, leading
   `api/`, leading version prefix (`v1/`, `v2/`), drop `{path-params}`,
   join remaining segments with `-`, lowercase.
2. **Fall back to function/class name** when no path is available
   (Django FBV without a routed counterpart, internal handlers).
3. **Strip suffixes** from class names: `View`, `APIView`, `Handler`,
   `Endpoint`, `Controller`, `ViewSet` (matches the generic Phase 2.5
   suffix list).
4. **Convert to kebab-case** — snake_case → kebab-case (`list_messages`
   → `list-messages`); PascalCase → kebab-case
   (`MessageListView` → `message-list`).
5. **Truncate to 48 chars**, deduplicate with `-2`, `-3` if needed.

## Insertion point

Every insertion runs **after** auth/permission resolution (so `identify`
has a real user_id) and **before** business logic. Frame: "deny early on
disabled, but only once we know who you are."

### FastAPI

```python
@app.post("/chat/stream")
async def chat_stream(user: User = Depends(current_user)):
    # ← INSERT HERE — Depends() params have already been resolved.
    onelo.identify(user.id)
    if not onelo.features.feature("chat-stream").is_enabled:
        raise HTTPException(404, "Feature not available")
    # existing business logic continues below
```

**Where to insert:** first line of the function body (after the
signature `:` line), past any `Depends(...)` parameters which resolve
on the signature itself. If the handler has no auth dependency,
skip the `identify` line — gate purely on the feature.

### Django function-based view

```python
def chat_stream(request):
    # ← INSERT HERE — `request.user` is populated by AuthenticationMiddleware.
    if request.user.is_authenticated:
        onelo.identify(str(request.user.id))
    if not onelo.features.feature("chat-stream").is_enabled:
        return HttpResponseNotFound()
    # existing logic
```

**Where to insert:** first line of the function body. The `is_authenticated`
guard is required because Django views can be reached anonymously by
default — `identify(None)` would emit a warning.

### Django class-based view

```python
class ChatStreamView(View):
    def get(self, request, *args, **kwargs):
        # ← INSERT HERE — top of each HTTP method.
        if request.user.is_authenticated:
            onelo.identify(str(request.user.id))
        if not onelo.features.feature("chat-stream").is_enabled:
            return HttpResponseNotFound()
        # existing logic

    def post(self, request, *args, **kwargs):
        # ← INSERT HERE too — instrument every public verb separately.
        ...
```

**Where to insert:** first line of each `get` / `post` / `put` /
`patch` / `delete` / `head` method present on the class. Do not
instrument `dispatch()` — it runs for every verb and the gate would
short-circuit DRF's permission classes. For DRF `ViewSet` /
`ModelViewSet`, instrument `list`, `create`, `retrieve`, `update`,
`partial_update`, `destroy` individually.

### Flask

```python
@app.route("/chat/stream", methods=["POST"])
def chat_stream():
    # ← INSERT HERE — `session` and `g` are populated by Flask's request context.
    user_id = session.get("user_id")  # or g.user.id, or whatever your auth uses
    if user_id:
        onelo.identify(user_id)
    if not onelo.features.feature("chat-stream").is_enabled:
        abort(404)
    # existing logic
```

**Where to insert:** first line of the function body. Flask has no
single canonical auth pattern — adapt the `user_id` extraction to
match the project's existing auth (Flask-Login `current_user.id`,
JWT `get_jwt_identity()`, session, etc.).

### aiohttp

```python
async def chat_stream(request):
    # ← INSERT HERE — request is the only parameter, available immediately.
    user_id = request.headers.get("X-User-Id")  # or request['user'].id from middleware
    if user_id:
        onelo.identify(user_id)
    if not onelo.features.feature("chat-stream").is_enabled:
        return web.Response(status=404)
    # existing logic
```

**Where to insert:** first line of the handler body. aiohttp middleware
typically attaches the user to `request['user']` — adapt the extraction
to match. If the handler is wrapped by `@aiohttp_security.has_permission`
or similar, insert *after* those decorators have resolved (they wrap
the whole call, so the function body is still the right spot).

### Module-scope `onelo` instance assumption

All snippets assume an `onelo` instance is reachable in the file's
scope. The typical patterns:

- **Same-module init** — common for small FastAPI/Flask projects:
  the file that defines `app = FastAPI()` also defines `onelo = Onelo(...)`.
- **Module import** — larger projects centralize: `from app.onelo import onelo`
  or `from .core import onelo`.
- **Django apps singleton** — `from .apps import onelo` (set in
  `AppConfig.ready()`, see "Project structure assumptions" below).

If the import isn't already present, add it at the top of the file
alongside other project imports.

## Snippets

> **API note:** The Python SDK is **synchronous**. There is no `await`
> on `feature(...)`, `is_enabled`, or `declare(...)`. The `Feature`
> object exposes `is_enabled`, `is_visible`, `is_greyed`, `is_new`,
> `is_beta`, `is_coming_soon` as **properties** (no parentheses).
> Calling them as methods (`feat.is_enabled()`) raises `TypeError:
> 'bool' object is not callable`.
>
> Reads from the in-process cache (~1µs) and never block on the network —
> a long-lived background thread keeps the cache fresh via SSE.

### Choosing the right check

Default to **`is_enabled`** for "execute this handler or 404" gates. It
matches the dashboard's primary on/off semantics and is what the SDK's
README leads with.

| Check | True when status is | Use for |
|---|---|---|
| `is_enabled` | `enabled`, `new`, `beta` | **Default.** Binary gate: serve the route or return 404. Paid endpoints, A/B rollouts. |
| `is_visible` | anything except `hidden` | When a "coming soon" / placeholder response is more useful than a hard 404. The status itself is the signal for *which* placeholder to send. |
| `is_greyed`, `is_upsell`, `is_new`, `is_beta`, `is_coming_soon` | the matching status | Branching the response body on a specific promotional state (e.g., return `{"upsell": true}` instead of data). |

### FastAPI route handler

```python
if not onelo.features.feature("$NAME").is_enabled:
    raise HTTPException(404, "Feature not available")
```

If the project uses a custom `404` model:

```python
feat = onelo.features.feature("$NAME")
if not feat.is_enabled:
    raise HTTPException(status_code=404, detail={"error": "feature_disabled", "feature": feat.name})
```

### Django view (FBV or CBV method)

```python
if not onelo.features.feature("$NAME").is_enabled:
    return HttpResponseNotFound()
```

For DRF `APIView`:

```python
from rest_framework.response import Response
from rest_framework import status

if not onelo.features.feature("$NAME").is_enabled:
    return Response({"error": "Feature not available"}, status=status.HTTP_404_NOT_FOUND)
```

### Flask route handler

```python
if not onelo.features.feature("$NAME").is_enabled:
    abort(404)
```

### aiohttp handler

```python
if not onelo.features.feature("$NAME").is_enabled:
    return web.Response(status=404)
```

For JSON-returning aiohttp handlers:

```python
if not onelo.features.feature("$NAME").is_enabled:
    return web.json_response({"error": "Feature not available"}, status=404)
```

### Dev-mode default

The Python SDK has no `feature_default_status` constructor argument in
v0.1.0a1 — newly instrumented features always start as `hidden` until
toggled in the dashboard. Until that ships (planned for the same
release as the JS/Swift `featureDefaultStatus` parity), use the
dashboard's "set status to enabled on first appearance" workflow on
the staging tenant for local development.

### Rich destination states (opt-in)

For routes that should serve a placeholder instead of a hard 404:

```python
feat = onelo.features.feature("$NAME")
if feat.is_coming_soon:
    return {"status": "coming_soon", "message": "We're building this!"}
if feat.is_greyed:
    raise HTTPException(402, "Upgrade required")  # 402 Payment Required
if not feat.is_enabled:
    raise HTTPException(404, "Feature not available")
# ... real handler logic
```

This is opt-in — write it once per route where rich states matter.

## Trigger detection

Backend Python rarely has "triggers" in the frontend sense (a button
opens a screen). Routes themselves *are* the destinations. The trigger
concept maps onto **internal callers** of those routes — Celery tasks,
APScheduler jobs, RQ workers, or other backend services hitting an
internal HTTP endpoint via `requests`/`httpx`/`aiohttp.ClientSession`.

In most projects, **trigger detection for Python is low-value** and
should be skipped: backend services don't render UI, so there's no
"orphaned trigger" failure mode the way there is in a frontend app.
Enable trigger detection only when the developer explicitly opts in —
e.g., a Celery task that only enqueues work for users who have a
particular feature enabled, where you want the task itself gated to
match.

### Grep patterns (opt-in)

```bash
# Internal HTTP client calls (requests, httpx)
grep -rnE '(requests|httpx)\.(get|post|put|patch|delete)\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv \
  . 2>/dev/null

# httpx.AsyncClient and requests.Session usage
grep -rnE '(client|session)\.(get|post|put|patch|delete)\(' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv \
  . 2>/dev/null

# aiohttp client (distinct from server)
grep -rnE 'aiohttp\.ClientSession|ClientSession\(\)' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv \
  . 2>/dev/null

# Celery tasks (caller code, decorated with @shared_task or @app.task)
grep -rnE '@(shared_task|[a-z_]+\.task)' \
  --include='*.py' \
  --exclude-dir=__pycache__ --exclude-dir=.venv \
  . 2>/dev/null
```

### Trigger snippet

```python
feat = onelo.features.feature("$NAME")
if not feat.is_visible:
    return  # or `continue` in a loop, or skip the task
# existing trigger logic (HTTP call, task enqueue, etc.)
```

`is_visible` (rather than `is_enabled`) keeps the trigger active for
non-`hidden` statuses — `greyed`, `coming_soon`, `beta`, etc. — so an
admin's intended UX renders even when the feature isn't fully on.

## Codegen for `FEATURE_REGISTRY`

### Generated file

Path: `<package>/_generated/feature_registry.py`. The leading
underscore in `_generated` follows Python convention for "do not import
manually" packages and prevents `mypy --strict` from flagging missing
type stubs.

```python
# AUTO-GENERATED. Do not edit manually.
# Regenerated on every build by tool/generate_feature_registry.py
FEATURE_REGISTRY: list[str] = [
    "chat-stream",
    "voice-stream",
    "game-think",
    # ...
]
```

### Generator script

Pure Python, no external dependencies — runs anywhere `python3` is
available. Drop into `tool/generate_feature_registry.py`.

```python
#!/usr/bin/env python3
"""Scan Python sources for `feature("...")` calls and emit FEATURE_REGISTRY."""
from __future__ import annotations

import argparse
import re
import sys
from pathlib import Path

# Match features.feature("name") and features.feature('name'), atom syntax only.
PATTERN = re.compile(r'''features\.feature\(\s*['"]([a-z][a-z0-9-]*)['"]''')

EXCLUDE_DIRS = {
    "__pycache__", ".venv", "venv", "env", ".tox", "migrations",
    "tests", "test", "build", "dist", ".eggs", ".pytest_cache",
    "_generated", "site-packages",
}


def discover(sources_dir: Path) -> set[str]:
    names: set[str] = set()
    for path in sources_dir.rglob("*.py"):
        if any(part in EXCLUDE_DIRS for part in path.parts):
            continue
        try:
            text = path.read_text(encoding="utf-8")
        except (OSError, UnicodeDecodeError):
            continue
        names.update(PATTERN.findall(text))
    return names


def emit(names: set[str], out_file: Path) -> None:
    out_file.parent.mkdir(parents=True, exist_ok=True)
    sorted_names = sorted(names)
    body = "\n".join(f'    "{n}",' for n in sorted_names)
    out_file.write_text(
        "# AUTO-GENERATED. Do not edit manually.\n"
        "# Regenerated on every build by tool/generate_feature_registry.py\n"
        "FEATURE_REGISTRY: list[str] = [\n"
        f"{body}\n"
        "]\n",
        encoding="utf-8",
    )


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("sources_dir", nargs="?", default="src")
    parser.add_argument(
        "out_file",
        nargs="?",
        default=None,
        help="Defaults to <sources_dir>/<first-package>/_generated/feature_registry.py",
    )
    args = parser.parse_args()

    sources_dir = Path(args.sources_dir).resolve()
    if not sources_dir.is_dir():
        print(f"error: sources dir not found: {sources_dir}", file=sys.stderr)
        return 1

    if args.out_file:
        out_file = Path(args.out_file)
    else:
        # Find the first package (dir with __init__.py) under sources_dir.
        pkg = next((p.parent for p in sources_dir.rglob("__init__.py")), sources_dir)
        out_file = pkg / "_generated" / "feature_registry.py"

    names = discover(sources_dir)
    emit(names, out_file)
    print(f"Generated {out_file} with {len(names)} features")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run manually with `python3 tool/generate_feature_registry.py src` (or
the relevant sources dir).

### Build hook integration

Python doesn't have a single canonical "pre-build" lifecycle the way
npm or Gradle does, so wire the generator into whichever entry point
your team already uses:

**`Makefile` (most portable):**

```makefile
.PHONY: generate-registry dev test build

generate-registry:
	python3 tool/generate_feature_registry.py src

dev: generate-registry
	uvicorn app.main:app --reload

test: generate-registry
	pytest

build: generate-registry
	python3 -m build
```

**`tox.ini`:** add a `commands_pre` step on the relevant envs:

```ini
[testenv]
commands_pre =
    python tool/generate_feature_registry.py src
commands =
    pytest {posargs}
```

**Poetry (`pyproject.toml`):** Poetry's `scripts` table only registers
console-script entry points, not lifecycle hooks. Wire the generator
through a `pre-commit` hook plus a Makefile target:

```toml
[tool.poetry.scripts]
generate-registry = "tool.generate_feature_registry:main"
```

Then invoke `poetry run generate-registry` from your dev/build
scripts.

**Hatch / PDM / setuptools-build:** all support custom build hooks via
their respective plugin systems, but the simplest path is the Makefile
or a one-liner `prebuild.py` invoked from CI before `python -m build`.

**`pre-commit` hook (catches drift in PRs):** add to
`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local
    hooks:
      - id: generate-feature-registry
        name: Regenerate Onelo feature registry
        entry: python3 tool/generate_feature_registry.py src
        language: system
        pass_filenames: false
        files: \.py$
```

Wire it once, forget about it — every commit refreshes the registry.

## Project structure assumptions

Where the `onelo` singleton typically lives, by framework:

| Framework | Init location | How handlers reach it |
|---|---|---|
| FastAPI | Top of `app/main.py` (alongside `app = FastAPI()`) or in `app/dependencies.py` for larger projects. Use `lifespan` context manager to call `onelo.ready()` and `onelo.close()`. | Direct module import: `from app.main import onelo` or `from .dependencies import onelo`. |
| Django | `apps.py` `AppConfig.ready()` method, set on a module-level variable. | `from <app>.apps import onelo`. |
| Flask | Alongside `app = Flask(...)` in `app.py` / `__init__.py`, or attached to the app via `app.extensions["onelo"]` for the application-factory pattern. | Direct module import or `current_app.extensions["onelo"]`. |
| aiohttp | At module level alongside `app = web.Application()`, with `app.on_cleanup.append(...)` to call `onelo.close()` on shutdown. | Direct module import or `request.app["onelo"]` if attached to the app. |

### FastAPI lifespan example (full init)

```python
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI
from onelo import Onelo
from app._generated.feature_registry import FEATURE_REGISTRY

onelo = Onelo(
    publishable_key=os.environ["ONELO_PUBLISHABLE_KEY"],
    api_url=os.environ.get("ONELO_API_URL", "https://app.onelo.tools"),
)

@asynccontextmanager
async def lifespan(app: FastAPI):
    onelo.features.declare(FEATURE_REGISTRY)
    onelo.ready(timeout=2.0)  # block up to 2s for first SSE event
    yield
    onelo.close()

app = FastAPI(lifespan=lifespan)
```

### Django apps.py example

```python
import os
from django.apps import AppConfig
from onelo import Onelo

onelo: Onelo | None = None

class CoreConfig(AppConfig):
    name = "core"

    def ready(self):
        from core._generated.feature_registry import FEATURE_REGISTRY

        global onelo
        onelo = Onelo(
            publishable_key=os.environ["ONELO_PUBLISHABLE_KEY"],
            api_url=os.environ.get("ONELO_API_URL", "https://app.onelo.tools"),
        )
        onelo.features.declare(FEATURE_REGISTRY)
        onelo.ready(timeout=2.0)
```

### Flask app factory example

```python
import os
from flask import Flask
from onelo import Onelo

def create_app() -> Flask:
    app = Flask(__name__)

    from app._generated.feature_registry import FEATURE_REGISTRY
    onelo = Onelo(
        publishable_key=os.environ["ONELO_PUBLISHABLE_KEY"],
        api_url=os.environ.get("ONELO_API_URL", "https://app.onelo.tools"),
    )
    onelo.features.declare(FEATURE_REGISTRY)
    onelo.ready(timeout=2.0)
    app.extensions["onelo"] = onelo

    # register blueprints, etc.
    return app
```

### aiohttp example

```python
import os
from aiohttp import web
from onelo import Onelo
from app._generated.feature_registry import FEATURE_REGISTRY

onelo = Onelo(
    publishable_key=os.environ["ONELO_PUBLISHABLE_KEY"],
    api_url=os.environ.get("ONELO_API_URL", "https://app.onelo.tools"),
)
onelo.features.declare(FEATURE_REGISTRY)
onelo.ready(timeout=2.0)

async def _close_onelo(app: web.Application) -> None:
    onelo.close()

app = web.Application()
app.on_cleanup.append(_close_onelo)
```

### Multi-process / fork-worker safety

Production Python deployments fork workers (uvicorn `--workers 4`,
gunicorn `-w 4`). The Onelo Python SDK auto-handles this via
`os.register_at_fork` — each forked worker spawns its own background
thread and SSE connection on first `feature(...)` access. **No special
configuration needed** — see the "How it works" section in
`packages/onelo-python/README.md` for details.

If your deployment uses `multiprocessing.Process` or `concurrent.futures.ProcessPoolExecutor`
for *application code* (not the WSGI/ASGI worker model), each child
process will likewise get its own background thread the first time it
calls `feature(...)`. The SDK is process-safe; what it doesn't share
is the in-memory cache (each process has its own).
