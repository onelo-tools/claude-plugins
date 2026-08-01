# Python — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install 'onelo[fastapi] @ git+https://github.com/onelo-tools/onelo-python.git'
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=python field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
import os
from fastapi import FastAPI, Depends
from onelo import Onelo
from onelo.fastapi import RequireUser, OneloUser

onelo = Onelo(secret_key=os.environ["ONELO_SECRET_KEY"])  # onelo_sk_live_…  (NEVER commit)
require_user = RequireUser(onelo)

app = FastAPI()
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=python field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
@app.get("/me")
async def me(user: OneloUser = Depends(require_user)):
    return {"id": user.id, "email": user.email}

# SSE / EventSource endpoints can't set an Authorization header — accept the
# token from ?token= instead (opt-in; a token in a URL can leak via logs):
sse_user = RequireUser(onelo, accept_query_token=True)

@app.get("/stream")
async def stream(user: OneloUser = Depends(sse_user)):
    ...   # EventSource("/stream?token=" + accessToken)

# OptionalUser injects user=None instead of 401 when the token is missing/invalid.
# A 503 on an Onelo outage still propagates — never silently anonymous.
#   from onelo.fastapi import OptionalUser

# Same secret_key client, other frameworks:
#   Flask    → from onelo.flask import require_user           (decorator)
#   Django   → from onelo.django import OneloAuthenticationFactory   (DRF class)
#   Litestar → from onelo.litestar import OneloGuardFactory
#   ASGI (any) → onelo.asgi.OneloAsgiMiddleware  — sets scope["onelo_user"];
#     on an Onelo outage ALSO sets scope["onelo_auth_unavailable"]=True → 503.
#   WSGI (any) → onelo.wsgi.OneloWsgiMiddleware  — sets environ["onelo.user"];
#     on outage ALSO sets environ["onelo.auth_unavailable"]=True → 503 (not anon).
#
#   No framework, or one we don't wrap (aiohttp, Sanic, websockets, a worker)?
#   The core is framework-agnostic — FastAPI above is just the example, never a
#   requirement. Verify a token directly:
#       from onelo.auth import verify_token        # async  (verify_token_sync for sync)
#       user = await verify_token(onelo, token)    # → OneloUser, or a typed error
#
#   Install per stack: onelo[fastapi] · onelo[flask] · onelo[litestar] · or plain
#   onelo for just the core (verify_token) + the ASGI/WSGI middleware.
#
# Error → status: OneloAuthInvalidToken → 401 · OneloAuthForbidden → 403 ·
#   OneloAuthRateLimited / OneloAuthUnavailable → 503 (a 429 is not retried).
```
<!-- /onelo:snippet -->
