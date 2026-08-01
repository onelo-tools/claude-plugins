# Node.js — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=node field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-node
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=node field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Onelo Features SDK — Node.js (server-side / backend)
//
// 🔑 KEY — your live SECRET key (onelo_sk_live_*). Server credential for the
//    SSE stream, features, auth verification, monitor. NOT a publishable key.
//    KEEP IT THE SAME across dev/staging/prod. Dashboard → API Keys → Secret keys.
//
// 🌐 FEATURE ENVIRONMENT — "test" or "live". Selects which feature snapshot the
//    server reads AND whether feature DISCOVERY is active. In dev/staging set
//    "test" (reads the Test snapshot, registers the slugs your code checks into
//    the dashboard registry); in prod set "live" (or omit → live). Set the SAME
//    value in your client app so app + backend resolve the same snapshot.
//      ONELO_FEATURE_ENVIRONMENT=test   (staging)   /   =live or unset (prod)
import { Onelo } from '@onelo/node'

const onelo = new Onelo({
  secretKey: process.env.ONELO_SECRET_KEY!,   // onelo_sk_live_*
  apiUrl: 'https://api.onelo.tools',
  featureEnvironment: process.env.ONELO_FEATURE_ENVIRONMENT as 'test' | 'live' | undefined,
  // Optional: strategy 'auto' | 'sse' | 'polling'; appVersion; pollInterval.
})

// Optional: register known features so they appear in the dashboard registry
// without waiting for code paths to run.
onelo.features.declare(['advanced-export', 'face-stream'])

// Recommended: block until the first snapshot lands so the very first request
// sees real state (not the 'hidden' fail-closed default). Returns a Promise<boolean>.
await onelo.features.ready(2000)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=node field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Global flag (no per-user targeting) — the common case. Synchronous read:
app.get('/export', (req, res) => {
  if (!onelo.features.feature('advanced-export').isEnabled) return res.status(404).end()
  // ...run the feature
})

// Per-user / per-plan gating — resolve for THIS request's user. STATELESS and
// multi-user-safe (no global identity, no race). Cached per-user ~30s; fail-closed
// (a network error resolves every feature to hidden, never throws):
app.get('/reports', async (req, res) => {
  const uf = await onelo.features.forUser(req.oneloUser!.id)
  const feat = uf.feature('advanced-reports')
  if (feat.isEnabled) return res.json(await buildReports())
  // Server-rendered upsell — tell the user WHICH plan unlocks it:
  if (feat.upgradeHint) return res.json({ locked: `Available in ${feat.upgradeHint}`, cta: feat.upgradeCta })
  return res.status(404).end()
})

// After YOUR backend processes a plan change (your Stripe webhook / a grant),
// drop the cached snapshot so the NEXT forUser() is fresh immediately:
// onelo.features.invalidateUser(userId)   // pass nothing to clear every user

// ⚠️ onelo.identify(userId) sets ONE process-global identity (full SSE reconnect
// per switch) — only for single-user processes (worker / CLI). In a multi-user
// backend use forUser() instead; the SDK warns if a plan-gated feature (upsell/
// greyed) is read through the global feature() path.

// Feature property checks: .isEnabled · .isVisible (fail-closed for unknown status)
// · .isGreyed · .isNew · .isBeta · .isComingSoon · .isUpsell · .isKnown · .status
// Upsell metadata: .upgradeHint ("Pro") · .requiredPlan · .requiredPlanLabel · .upgradeCta · .reason
```
<!-- /onelo:snippet -->
