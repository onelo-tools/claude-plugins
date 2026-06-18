# JS / TS / React / Next.js Patterns

## JS-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
JavaScript / Next.js add three signals on top:

- **Routing convention beats naming.** Any of these paths is a `screen`
  *regardless of the component's symbol name*:
  - `app/<route>/page.tsx`, `app/<route>/page.jsx` (Next.js App Router)
  - `app/<route>/route.ts`, `app/<route>/route.js` (Next.js route handler)
  - `pages/<route>.tsx`, `pages/<route>.jsx` (Next.js Pages Router)
  - Express/Fastify/Hono/Hapi route registration calls
    (`.get('/x', ...)`, `app.post(...)`, `router.get(...)`)

- **JSX shape disambiguates ambiguous components.** When the symbol survives
  Phase 2.5 as `ambiguous`, peek at what the component returns:
  - Returns JSX that imports other components from `/components/` →
    `screen` (it's composing a page).
  - Returns mostly raw `<div>`/`<span>`/`<button>` and Tailwind primitives →
    `atom-likely` (drop).

- **`*Provider` and `*Context` → drop.** These wrap the tree and never
  represent a user-visible feature.

## Detection (sub-language)

After detecting `package.json`, further distinguish:
- Check for `next.config.js` or `next.config.ts` or `next.config.mjs` → **Next.js**
- Check for `.tsx` or `.jsx` files anywhere in `src/` → **React**
- Otherwise → **Node.js**

## Grep patterns

Run each pattern. Collect file + line number + captured symbol name.

```bash
# React / Next.js components (function declaration)
grep -rn --include="*.tsx" --include="*.jsx" \
  -E "^export (default )?function ([A-Z][A-Za-z0-9]+)" \
  src/ app/ pages/ components/ 2>/dev/null

# React / Next.js components (arrow function)
grep -rn --include="*.tsx" --include="*.jsx" \
  -E "^export const ([A-Z][A-Za-z0-9]+) = \(" \
  src/ app/ pages/ components/ 2>/dev/null

# Next.js route handlers
grep -rn --include="*.ts" --include="*.tsx" \
  -E "^export (async )?function (GET|POST|PUT|DELETE|PATCH)" \
  src/ app/ 2>/dev/null

# Express / Node.js routes
grep -rn --include="*.ts" --include="*.js" \
  -E "\.(get|post|put|delete|patch)\(['\"]/" \
  src/ routes/ api/ 2>/dev/null
```

**Skip these paths:** `node_modules/`, `.next/`, `dist/`, `build/`, `out/`, `coverage/`, `*.test.ts`, `*.spec.ts`, `*.test.tsx`, `*.spec.tsx`, `*.stories.tsx`

## Symbol extraction

From each grep match, extract the symbol name:
- `export function ExportButton(` → symbol: `ExportButton`
- `export const AnalyticsPage = (` → symbol: `AnalyticsPage`
- `export async function GET(` → symbol: `GET` — prefix with route path if determinable from file path (e.g. `app/api/reports/route.ts` → symbol: `reports-api`, skip kebab conversion)
- `.get('/users',` → symbol from route string: `/users` → `users-list`

## Insertion point

**React/Next.js component:**
- Read the file
- Find the function body opening `{`
- Scan forward past any `use` hook lines (`useState`, `useEffect`, `useCallback`, `useMemo`, `useRef`, `useContext`, `useRouter`, `usePathname`, `useSearchParams`, `useParams`)
- Insert BEFORE the first non-hook line (or before `return` if no non-hook lines)

**Next.js route handler:**
- Insert as first line of the function body after `{`

**Express route:**
- Insert as first line of the callback function body

## Snippets

> **API note:** Onelo's TypeScript SDK exposes `isVisible`, `isEnabled`, `isGreyed`,
> `isUpsell` etc. as **getters**, not methods. Access them without parentheses
> (e.g. `feat.isEnabled`, never `feat.isEnabled()`). Calling them as functions
> produces a TypeScript "expression is not callable" error and a runtime
> `TypeError` at the same line.

### Choosing the right check

Default to **`isEnabled`** for "show this content or don't" gates. It
matches the dashboard's primary on/off semantics and the JS snippet
shown on `app.onelo.tools`, so docs and codegen stay aligned.

| Check | True when status is | Use for |
|---|---|---|
| `isEnabled` | `enabled`, `new`, `beta` | **Default.** Binary gate: show content vs. hide. Paid features, A/B tests, gradual rollouts. |
| `isVisible` | anything except `hidden` | When you render a greyed-out / upsell / "coming soon" variant in place of the real content. The status itself is the signal for *which* placeholder to show. |
| `isGreyed`, `isUpsell`, `isNew`, `isBeta`, `isComingSoon` | the matching status | Branching on a specific promotional state — pairs with the `<OneloFeatureBadge>` component when you have one. |

**React component (replaces `null` return for hidden features):**
```tsx
const feat = onelo.features.feature('$NAME')
if (!feat.isEnabled) return null
```

**React component with greyed/upsell variants:**
```tsx
const feat = onelo.features.feature('$NAME')
if (!feat.isVisible) return null
// if (feat.isGreyed) { /* render locked/disabled UI */ }
// if (feat.isUpsell) { /* render upgrade prompt */ }
```

**Next.js route handler:**
```ts
const feat = onelo.features.feature('$NAME')
if (!feat.isEnabled) return NextResponse.json({ error: 'Feature not available' }, { status: 404 })
```

**Express route:**
```js
const feat = onelo.features.feature('$NAME')
if (!feat.isEnabled) { res.status(404).json({ error: 'Feature not available' }); return }
```

Use the **React component** snippet for `.tsx`/`.jsx` files.
Use the **Next.js route** snippet for `route.ts` files.
Use the **Express route** snippet for `.js`/`.ts` files containing `.get(`/`.post(` etc.

> The snippets assume an `onelo` instance is reachable in scope (typically
> imported from a shared module — see Phase 6 of SKILL.md for the init pattern).
> If your codebase aliases `const features = onelo.features` at module level,
> the bare `features.feature(...)` form also works.

### Dev-mode default

A freshly instrumented `feature("$NAME")` returns `hidden` by default
until the gate is enabled in the dashboard — fail-closed in production.
During development you can flip the default so new gates render until
you toggle them in the dashboard:

```ts
import { Onelo } from '@onelo/js'

export const onelo = new Onelo({
  publishableKey: 'pk_live_...',
  baseURL: 'https://app.onelo.tools',
  featureDefaultStatus: process.env.NODE_ENV === 'development' ? 'enabled' : 'hidden',
})
```

Available in `@onelo/js` ≥ 3.19.0.

### Upfront declaration & registry codegen

Lazy discovery via `feature("...")` only registers a name once the code
path actually runs. Routes that aren't in your hot path (admin handlers,
debug pages, monitoring endpoints, error boundaries) won't show up in
the dashboard until someone happens to hit them.

Call `declare(...)` once at startup with every name you instrument so
the dashboard reflects the full registry from the first run:

```ts
import { onelo } from './onelo'
import { FEATURE_REGISTRY } from './generated/feature-registry'

onelo.features.declare([...FEATURE_REGISTRY])
```

Maintain `FEATURE_REGISTRY` with a build-time script that greps your
sources for `feature("...")` calls — zero manual sync, zero risk of the
list drifting from reality.

#### Generated file

Path: `src/generated/feature-registry.ts`.

```ts
// AUTO-GENERATED. Do not edit manually.
// Regenerated on every build by scripts/generate-feature-registry.sh

export const FEATURE_REGISTRY = [
  'analytics-dashboard',
  'export-button',
  'reports-api',
  // ...
] as const
```

The `as const` widens to a string-literal union, so consumers can
narrow types if they want — e.g. `Extract<typeof FEATURE_REGISTRY[number], 'export-button'>`.

#### Generator script

Drop into `scripts/generate-feature-registry.sh` and `chmod +x`. Reads
two arguments (or env vars): sources directory and output file. Both
have sensible defaults so the typical case is a one-line invocation.

```bash
#!/bin/bash
set -euo pipefail

# Usage: generate-feature-registry.sh [<sources_dir>] [<output_file>]
SOURCES_DIR="${1:-${SOURCES_DIR:-src}}"
OUT_FILE="${2:-${OUT_FILE:-$SOURCES_DIR/generated/feature-registry.ts}}"

mkdir -p "$(dirname "$OUT_FILE")"

# Match both "..." and '...' quoting styles
NAMES=$(grep -rhE 'features\.feature\(["'\''][a-z][a-z0-9-]*["'\'']' "$SOURCES_DIR" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
  --exclude-dir=generated --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=dist \
  | grep -oE '["'\''][a-z][a-z0-9-]*["'\'']' \
  | tr -d "'\"" \
  | sort -u)

{
  echo "// AUTO-GENERATED. Do not edit manually."
  echo "// Regenerated on every build by scripts/generate-feature-registry.sh"
  echo ""
  echo "export const FEATURE_REGISTRY = ["
  while IFS= read -r name; do
    [ -n "$name" ] && echo "  '$name',"
  done <<< "$NAMES"
  echo "] as const"
} > "$OUT_FILE"

COUNT=$(echo "$NAMES" | grep -c . || true)
echo "Generated $OUT_FILE with $COUNT features"
```

The `--exclude-dir` flags keep the script from feeding on its own output
or any built artifacts. The atom regex `[a-z][a-z0-9-]*` matches the
SDK's `feature()` name validation.

#### Build hook integration

**`package.json` (npm / pnpm / yarn):** wire as `prebuild` and `predev`
so the registry stays fresh in both production builds and dev server
restarts.

```json
{
  "scripts": {
    "generate:registry": "./scripts/generate-feature-registry.sh src",
    "prebuild": "npm run generate:registry",
    "predev": "npm run generate:registry",
    "build": "next build",
    "dev": "next dev"
  }
}
```

**Vite / Webpack / Turbopack** projects can keep the same `prebuild`
hook. If you want re-generation on every file change in dev (rare
trade-off — most teams accept the dev-server cost on restart only), add
a watcher via `chokidar-cli`:

```json
"watch:registry": "chokidar 'src/**/*.{ts,tsx}' -i 'src/generated/**' -c 'npm run generate:registry'"
```

and run it in parallel with the dev server (`concurrently npm:dev npm:watch:registry`).

**Monorepos:** invoke the script per-package in your build orchestrator
(Turborepo, Nx, Lerna). Each package gets its own `src/generated/`
output and its own `prebuild` hook.

### Trigger detection patterns

#### Grep patterns

```bash
# React Router / Next.js routing
grep -rnE '(<Link\s|useRouter\(\)|router\.push|<Navigate\s|redirect\()' \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=dist

# Imperative navigation hooks
grep -rnE '(useNavigate|history\.push|window\.location|router\.replace)' \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next

# Lazy / dynamic imports for code-split routes
grep -rnE 'import\(["'\''][^"'\'']*["'\'']\)' \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules --exclude-dir=.next
```

#### Linking heuristic

For destination `AnalyticsPage` (feature `analytics-dashboard`):
- `<Link href="/analytics">` → match by URL segment
- `router.push('/analytics/dashboard')` → substring match
- `import('./pages/Analytics')` → match by module path

#### Trigger snippet

```ts
const feat = onelo.features.feature('$NAME')
if (!feat.isVisible) return null     // in components

// or inline:
{feat.isVisible && <Link href="/analytics">Analytics</Link>}
```

#### Rich destination states (opt-in)

```tsx
const feat = onelo.features.feature('$NAME')
if (feat.isGreyed) return <UpsellPanel feature={feat} />
if (feat.isComingSoon) return <ComingSoonPanel />
if (!feat.isEnabled) return null
return <RealContent />
```

### Next.js: 'use client' boundaries

The Onelo SDK uses browser-only APIs (`fetch` for SSE, `localStorage` /
`sessionStorage`, `EventSource`). It can only run inside Client
Components. Server Components that import the SDK will fail to render
with an "expected a Client Component" error.

The instrumentation snippet inserted by the plugin assumes the file
**is already a Client Component**. The pattern works as-is for files
that already start with `'use client'`. For files that don't, add the
directive at the top:

```tsx
'use client'

import { onelo } from '@/lib/onelo'

export function ExportButton() {
  const feat = onelo.features.feature('export-button')
  if (!feat.isEnabled) return null
  // ...
}
```

For pages that should remain server-rendered (`page.tsx` doing data
fetching), instrument **the inner Client Component**, not the page
itself. Example:

```tsx
// app/analytics/page.tsx — Server Component
import { AnalyticsClient } from './analytics-client'

export default async function AnalyticsPage() {
  const data = await fetchAnalytics()  // server-side
  return <AnalyticsClient initialData={data} />
}

// app/analytics/analytics-client.tsx — Client Component
'use client'
import { onelo } from '@/lib/onelo'
export function AnalyticsClient({ initialData }) {
  const feat = onelo.features.feature('analytics-dashboard')
  if (!feat.isEnabled) return null
  // ...
}
```

For instrumented files that mix server/client logic, the plugin marks
them in the proposal table — review in Phase 3 and split before
accepting if needed.

### Next.js: route handlers and Edge runtime

`route.ts` files run on the server, but the **runtime** matters for the
SDK:

- **Default (Node.js runtime)** — works. Onelo's server-side feature
  resolution uses `fetch` and standard request headers. No special
  setup needed.
- **`export const runtime = 'edge'`** — works for *one-shot* feature
  checks (call `feature("...")` synchronously, return a response, done).
  Avoid using the SDK's long-lived SSE stream from Edge handlers:
  Vercel resets the runtime context between invocations, so any
  persistent connection or in-memory cache is lost. SSE works for
  streaming a single response within the request duration limit (10s
  on Hobby, 60s on Pro), not for keeping a multi-client connection
  alive across requests.

If your handler does:

```ts
export const runtime = 'edge'

export async function GET() {
  const feat = onelo.features.feature('reports-api')
  if (!feat.isEnabled) return NextResponse.json({ error: '...' }, { status: 404 })
  // ...
}
```

— that's fine. Each invocation reads the feature state and exits. If
you start seeing stale feature state (admin clicked Deploy, route still
serves the old gate), drop `runtime = 'edge'` and use the default
Node.js runtime; that's where SSE-backed real-time updates work
correctly.
