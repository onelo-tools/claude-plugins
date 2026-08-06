# Node.js — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=node field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=node field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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

## usage
<!-- onelo:snippet sdk=auth lang=node field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
