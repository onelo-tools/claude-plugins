# Onelo Monitor — JavaScript / TypeScript (web + SSR/Node)

Per-language signals, primitives, grep patterns and fix syntax. The
language-agnostic audit rules (A–G) live in SKILL.md.

## Contents
- Detect signals
- Primitives (track / event) — and why there is NO `capture()` in JS
- Grep patterns
- Coverage scan patterns
- Insertion rules
- Browser vs SSR/Node (the platform sub-split — crash capture differs!)
- Snippet fetch + fallback
- Good vs bad examples

## Detect signals
- **SDK:** `import { Onelo } from '@onelo/js'`, a `new Onelo({ publishableKey, apiUrl })`
  init, and use of `onelo.monitor`. Package: `@onelo/js`
  (`github:onelo-tools/onelo-js`). The monitor lives at `onelo.monitor`
  (class `OneloMonitor`).
- **Runtime context** (drives the Browser-vs-SSR subsection only — NOT a separate
  file): is the instance created/used on the **client** (browser bundle, React
  components, `'use client'`) or on the **server** (Next.js route handlers / server
  components / `getServerSideProps`, Remix loaders, Express/Fastify, plain Node)?
  ```bash
  grep -rlnE "next/|getServerSideProps|app/api/|route\.(ts|js)|express\(|fastify\(" --include=*.{ts,tsx,js,jsx} .   # server / SSR present
  grep -rlnE "'use client'|window\.|document\.|useEffect\(" --include=*.{ts,tsx,js,jsx} .                            # client present
  ```
  Both present (e.g. Next.js) → apply BOTH lifecycle notes.

## Primitives — pick the right one
| Want to record | Use | Emits |
|---|---|---|
| an operation (has duration / can fail) | `monitor.track(name, fn, { meta })` — `fn` is sync or async | `source=track`, `ok+duration` on success / `error+duration` on throw; **re-throws** |
| an instantaneous thing (click, toggle, view) | `monitor.event(name, { ok, durationMs?, error?, meta? })` | `source=event` |
| an error you already caught | **No `capture()` in JS** → `monitor.event(name, { ok: false, error })` | `source=event`, `ok=false` |

**There is NO `capture()` / `captureException()` in `@onelo/js`** (unlike Swift /
Python). For an error you catch yourself, record it manually with
`event(name, { ok: false, error: msg })`. Truly *unhandled* errors are
auto-captured ONLY in the browser (see the platform sub-split below).

`track()` shapes:
```ts
// async
const data = await onelo.monitor.track('pdf_upload', () => upload(file), { meta: { size } })
// sync
const out = onelo.monitor.track('parse', () => parse(input), { meta: { bytes } })
```
`track` returns the callback's result and re-throws on error — keep the `await`
and the assignment; never change the surrounding `try/catch`.

## Grep patterns
Call sites:
```bash
grep -rnE "\.monitor\.(track|event)\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
Instantaneous events (candidates for Rule C / Rule F — inspect for nearby `await`
or `ok: false`):
```bash
grep -rnE "\.monitor\.event\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
Feature names (first string arg):
```bash
grep -rhoE "\.monitor\.(track|event)\(\s*['\"][a-zA-Z0-9_]+" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx . \
  | grep -oE "['\"][a-zA-Z0-9_]+"
```
Init present (and the ONE-instance check — more than one `new Onelo(` is a smell):
```bash
grep -rnE "new Onelo\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```

## Coverage scan patterns (Phase 4)
Operations worth `track()` (awaited / network / DB / disk / subprocess):
```bash
grep -rnE "async function|async \(|await |fetch\(|axios|\.(get|post|put|delete)\(|prisma\.|drizzle|knex|\bfs\.|child_process|exec\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
Error sites (where failures should be recorded — JS uses `event({ok:false})`):
```bash
grep -rnE "\} ?catch ?\(|\.catch\(|throw " --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
Background work where a rejection vanishes (fire-and-forget — the JS analogue of
an unguarded `Task {}`):
```bash
grep -rnE "void [a-zA-Z]+\(|\.then\(|setTimeout\(|setInterval\(|queueMicrotask\(|new Worker\(|\.on\(['\"]" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
Already instrumented — skip (a `monitor.` call within ~3 lines):
```bash
grep -rnE "\.monitor\.(track|event)\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
```
**Quality note:** a floating promise (`somethingAsync()` with no `await` / `.catch`)
swallows its rejection. In the browser the global `unhandledrejection` handler
catches it (as feature `unhandled`); **on the server there is NO such handler** —
wrap the body in `track()` or add `.catch(e => onelo.monitor.event('name', {ok:false, error:String(e)}))`.

## Insertion rules
- **track()** wraps an existing call — keep the body inside the callback:
  `const x = await onelo.monitor.track('name', () => doWork(), { meta })`. It
  returns the result and re-throws, so existing `try/catch` keeps working — never
  change the surrounding error handling. Preserve the `await` and assignment.
- **event()** goes on the line where the instantaneous thing happens.
- **caught error**: inside the existing `catch`, add
  `onelo.monitor.event('name', { ok: false, error: e instanceof Error ? e.message : String(e) })`
  (there is no `capture()`); if the code re-threw before, keep the `throw`.
- **ONE instance, module level** — never `new Onelo(...)` inside a component, hook
  or request handler (re-creating loses the buffer + resets the session id).
  Export it and import everywhere. For SSR create it ONCE at server startup.
- Never wrap a callback that isn't the operation itself.

## Browser vs SSR/Node  (the only platform sub-split — and it matters a LOT)
The SDK registers global crash handlers **only in the browser**
(`typeof window !== 'undefined'` → `window.addEventListener('error' / 'unhandledrejection')`,
emitting feature `unhandled`, `ok=false`).

- **Browser / client:** unhandled errors + promise rejections are auto-captured —
  no setup. Crash capture is "on".
- **SSR / Node (Next.js route handlers & server components, Remix loaders,
  Express, plain Node):** **NO global handler is installed.** An uncaught error or
  a floating promise rejection on the server is INVISIBLE to Onelo. This is the
  #1 coverage gap for server-side JS — you must `track()` server operations (or
  record caught errors via `event({ok:false})`) explicitly. Optionally wire
  `process.on('uncaughtException')` / `process.on('unhandledRejection')` yourself
  to forward into `monitor.event('unhandled', {ok:false, error})`.

**Lifecycle / session:**
- Browser: one `sessionId` per page load (resets on reload). SPA needs no teardown.
- Node/SSR: create the instance ONCE per process at startup; `sessionId` is then
  per-process. On graceful shutdown call `onelo.monitor.destroy()` (clears the
  15s timer + final flush) so buffered events aren't lost.

**Auto-attached (do NOT add manually):** `meta.sdk = {name:'@onelo/js', version}`,
`meta.app = {version, build, bundleId}` (from `appVersion`/`appBuild` in config),
`environment` (per-event value wins over the config default), `userId` (from
`setUserId()` / Onelo Auth), `sessionId`, `platform: 'js'`. Buffer cap 200, flush
every 15s, immediate flush on `ok:false`.

## Snippet fetch (Phase 4)
```bash
curl -sf "${ONELO_API_BASE:-https://app.onelo.tools}/api/snippets?sdk=monitor&lang=js"
```
Use the `usage` field and replace the feature name. Fallback if the fetch fails:
```ts
// operation (async)
const data = await onelo.monitor.track('pdf_upload', () => upload(file), { meta: { size: file.size } })
// instantaneous
onelo.monitor.event('tab_viewed', { ok: true, meta: { tab: 'export' } })
// already-caught error (NO capture() in JS — use event)
try { await pay() } catch (e) {
  onelo.monitor.event('checkout', { ok: false, error: e instanceof Error ? e.message : String(e) })
  throw e
}
```

## Good vs bad (examples pattern)
- `event('backend_error', { ok: false })` → `await onelo.monitor.track('backend_call', () => call())`  *(Rules A+B)*
- `event('permission_denied', { ok: false })` → `event('permission_check', { ok: false, error: 'denied' })`  *(Rule B)*
- `event('export_started'); …; event('export_completed')` → `track('export', () => …)`  *(Rule E)*
- `track('ai_response', fn)` with no meta → add `{ meta: { model: 'gpt-4' } }`  *(Rule D)*
- floating `sendEmail(user)` on the server (rejection vanishes) → `await onelo.monitor.track('email_send', () => sendEmail(user))` or `.catch(e => onelo.monitor.event('email_send', {ok:false, error:String(e)}))`  *(server coverage)*
