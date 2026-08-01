# Onelo Auth — backend verification snippets

Verifying an Onelo session **on the developer's own server**, baked into this
file when the plugin was published — straight from `@onelo/snippets`.

This is a different job from signing in. The client snippets
([snippets-client.md](snippets-client.md)) obtain a session; these verify the
token that session produced, so your API can trust who is calling it.

An app with its own backend usually needs **both**. Ask which the developer
wants — don't assume one implies the other.

Substitute `onelo_pk_live_YOUR_KEY` / `https://api.onelo.tools` with real values. The backend
also needs the **secret** key, which is never placed in client code.

---

## PYTHON

### install
<!-- onelo:snippet sdk=auth lang=python field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```python
pip install 'onelo[fastapi] @ git+https://github.com/onelo-tools/onelo-python.git'
```
<!-- /onelo:snippet -->

### init
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

### usage
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

---

## NODE

### install
<!-- onelo:snippet sdk=auth lang=node field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=node field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/node'
import { requireUser } from '@onelo/node/express'

// Authenticate with your SERVER SECRET key (onelo_sk_live_…) — NOT a
// publishable key. Read it from the environment; never hard-code a secret.
const onelo = new Onelo({
  secretKey: process.env.ONELO_SECRET_KEY!,   // onelo_sk_live_…  (NEVER commit)
  apiUrl: 'https://api.onelo.tools',
})
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=node field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// The frontend sends the user's ACCESS TOKEN (returned by the Onelo SDK after
// sign-in — NOT your secret key) as: Authorization: Bearer <token>.
// requireUser verifies it and attaches req.oneloUser.
app.get('/me', requireUser(onelo), (req, res) => {
  // requireUser guarantees req.oneloUser is set here — assert non-null.
  res.json({ id: req.oneloUser!.id, email: req.oneloUser!.email })
})

// Post-verify gates + SSE support (all optional):
//   requireUser(onelo, { acceptQueryToken: true })   // allow ?token= for EventSource/SSE
//   requireUser(onelo, { identifyMonitor: false })   // opt out of auto monitor.setUser (on by default when a monitor scope is active)
//   optionalUser(onelo)   // sets req.oneloUser = null instead of 401 (a 503 outage still surfaces)

// ── Other frameworks — one adapter import, same onelo client ──
//   Next.js  → import { withUser } from '@onelo/node/next'
//              export const GET = withUser(onelo, async (req, user) =>
//                Response.json({ id: user.id }))
//   Fastify  → import { requireUser } from '@onelo/node/fastify'
//              fastify.get('/me', { preHandler: requireUser(onelo) },
//                async (req) => ({ id: req.oneloUser!.id }))
//   Hono     → import { requireUser } from '@onelo/node/hono'
//              app.use('/me', requireUser(onelo))
//              app.get('/me', (c) => c.json({ id: c.get('oneloUser').id }))
//   Any other (Koa, Nest, raw http, WebSocket, queue worker):
//              import { verifyToken } from '@onelo/node'
//              const user = await verifyToken(onelo, token)   // throws typed errors
//
// Error → status: OneloAuthInvalidToken / missing → 401 · OneloAuthForbidden /
//   failed gate → 403 · OneloAuthRateLimited (429) / OneloAuthUnavailable (5xx) → 503.
```
<!-- /onelo:snippet -->

---

## PHP

### install
<!-- onelo:snippet sdk=auth lang=php field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

### init
<!-- onelo:snippet sdk=auth lang=php field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
<?php
use Onelo\Onelo;

// Authenticate with your SERVER SECRET key (onelo_sk_live_…) — NOT a publishable
// key. Read it from the environment; never hard-code a secret in source.
$onelo = new Onelo(
    secretKey: $_ENV['ONELO_SECRET_KEY'],   // onelo_sk_live_…  (NEVER commit)
    apiUrl: 'https://api.onelo.tools',
);
```
<!-- /onelo:snippet -->

### usage
<!-- onelo:snippet sdk=auth lang=php field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```php
// ── Laravel ── bind $onelo in a service provider, then guard a route.
// The frontend sends the user's ACCESS TOKEN as: Authorization: Bearer <token>.
use Onelo\Laravel\OneloAuthenticate;

Route::get('/me', function (Illuminate\Http\Request $request) {
    $user = $request->attributes->get('oneloUser');   // Onelo\Auth\OneloUser
    return ['id' => $user->id, 'email' => $user->email];
})->middleware(OneloAuthenticate::class);

// Post-verify gates + SSE support (all optional):
//   new OneloAuthenticate($onelo, options: [
//     'acceptQueryToken'     => true,          // allow ?token= for EventSource/SSE
//     'identifyMonitor'      => false,         // opt out of auto monitor->setUser (on by default when the monitor scope is request-isolated)
//   ]);
//   OneloAuthenticate::optional($onelo)  // oneloUser = null instead of 401 (503 outage still surfaces)

// ── PSR-15 (Slim, Mezzio, Symfony) — one middleware, same $onelo client ──
//   use Onelo\Psr15\OneloAuthMiddleware;
//   $app->add(new OneloAuthMiddleware($onelo, $responseFactory));  // $responseFactory = your PSR-17 factory
//   // then in the handler: $request->getAttribute('oneloUser')

// ── Any other framework / a queue worker / no framework ──
//   The core is framework-agnostic — verify a token directly:
//   use Onelo\Auth\Verify;
//   $user = Verify::verifyToken($onelo, $token);   // → OneloUser, or a typed exception
//
// Error → status: OneloAuthInvalidToken / missing → 401 · OneloAuthForbidden /
//   failed gate → 403 · OneloAuthRateLimited (429) / OneloAuthUnavailable (5xx) → 503.
```
<!-- /onelo:snippet -->
