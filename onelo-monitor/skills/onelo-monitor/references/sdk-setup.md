# SDK setup — install, version, init (Swift, Python & JS)

Install / update the Onelo SDK and initialise Monitor. Read this in Phase 0
(present & current) and Phase 5 (init), for the SDKs this skill supports.

## Contents
- Install the SDK
- Keep the SDK current (version check)
- Initialize Monitor — Swift
- Initialize Monitor — Python
- Initialize Monitor — JavaScript / TypeScript

## Install the SDK
The Onelo SDK ships all modules (auth, features, paywall, monitor) in ONE package,
so if auth/features are already wired, the package is present — Monitor just needs
initialising (see below).

| Stack | Install |
|---|---|
| Swift (iOS / macOS) | Swift Package — `https://github.com/onelo-tools/onelo-swift` (branch: `staging`) |
| Python (FastAPI / Django / Flask / Litestar) | `pip install "git+https://github.com/onelo-tools/onelo-python.git@staging"` — extras: `onelo[fastapi]` / `[django]` / `[flask]` / `[litestar]` |
| JS / TS (vanilla / React / Vue / Next.js / Node) | `npm install github:onelo-tools/onelo-js#staging` (prod: drop `#staging`). Package `@onelo/js`. |

## Keep the SDK current (version check)
The snippets this skill inserts may use APIs added in newer versions —
instrumenting against a stale SDK can insert calls the installed version lacks.

| Stack | Show installed version | Update |
|---|---|---|
| Swift | grep `onelo-swift` in `Package.resolved` | Xcode → File → Packages → Update to Latest, or `swift package update` |
| Python | `pip show onelo` | `pip install -U "git+https://github.com/onelo-tools/onelo-python.git@staging"` |
| JS | `npm ls @onelo/js` / `version` in `node_modules/@onelo/js/package.json` | `npm install github:onelo-tools/onelo-js#staging` again (re-pulls the branch HEAD) |

**How to know the latest version:** the SDKs aren't on a package registry, so read
the git tags directly:
```bash
git ls-remote --tags https://github.com/onelo-tools/onelo-python   # → highest v0.5.0aN-staging
git ls-remote --tags https://github.com/onelo-tools/onelo-swift
```
Take the highest `*-staging` tag and compare to the installed version. If behind
(or you can't determine it), update with the command above.

**Minimum versions for APIs this skill inserts:**
- Python `monitor.track()` (context manager / decorator) → **onelo-python ≥ 0.5.0a23**.
  Older installs lack it; update before instrumenting or the inserted code breaks.

If you can't confirm the version, update anyway.

## Initialize Monitor — Swift
Initialise once for the app lifetime (SwiftUI: `@StateObject` on `@main App`;
AppKit/UIKit: in AppDelegate as a singleton). Crash capture (NSException +
MetricKit) installs automatically at init.
```swift
import OneloSwift

@main
struct MyApp: App {
  @StateObject private var onelo = Onelo(
    publishableKey: "{{publishableKey}}",
    callbackScheme: "myapp",
    baseURL: URL(string: "{{apiUrl}}")!
  )
  var body: some Scene { WindowGroup { ContentView().environmentObject(onelo) } }
}
// Lifecycle: macOS → onelo.monitor.destroy() in applicationWillTerminate;
// iOS → also onelo.monitor.flush() on willResignActive / ScenePhase .background.
```

## Initialize Monitor — Python
Call `monitor.init()` ONCE at startup, before any request handler. Wire the
framework integration + `install_excepthook=True` for global crash capture
(covers uncaught errors across threads + the asyncio loop).

**Credential — Python is a BACKEND SDK, so it authenticates with the SERVER
SECRET key (`onelo_sk_live_*`), NOT a publishable key.** A publishable key is a
public app *identifier* for client apps (protected by Origin / attestation /
client_secret); a backend is a trusted server and uses the secret key. Create it
in the Onelo dashboard → API Keys → Secret keys and read it from the environment
(`ONELO_SECRET_KEY`) — never hard-code a secret in source. (Functionally the
ingest endpoint accepts either key by prefix, but the secret key is the correct
server-side credential.)
```python
import os
from onelo import Onelo, monitor

onelo = Onelo(secret_key=os.environ["ONELO_SECRET_KEY"], api_url="{{apiUrl}}")
# Monitor is a single event stream — no test/live split (that's a Features-only
# concept). `environment` is just an OPTIONAL free-form deploy label/tag; omit it
# and the SDK reads ONELO_ENVIRONMENT / ENVIRONMENT only if you set them.
monitor.init(onelo=onelo, install_excepthook=True)

# FastAPI / Starlette / Litestar (ASGI):
from onelo.monitor.integrations import OneloMonitorASGIMiddleware
app.add_middleware(OneloMonitorASGIMiddleware)
# Django:  add "onelo.monitor.integrations.django.OneloMonitorMiddleware" to MIDDLEWARE
# Flask:   from onelo.monitor.integrations.flask import install; install(app)
```

## Initialize Monitor — JavaScript / TypeScript
Create ONE `Onelo` instance at module level (never inside a component / hook /
request handler — re-creating loses the buffer + resets the session id). For SSR
(Next.js / Remix) create it once at server startup, not per request.

**Credential — JS is a CLIENT SDK, so it uses the PUBLISHABLE key
(`onelo_pk_live_*`).** Unlike the Python backend SDK (secret key), a publishable
key is a public app identifier — safe to ship in a browser bundle because it's
protected by Origin / attestation / client_secret, and it can't do privileged
operations. (Even in a Next.js app the publishable key is the right one — it ends
up in the client bundle, which is fine.)
```ts
import { Onelo } from '@onelo/js'

export const onelo = new Onelo({
  publishableKey: '{{publishableKey}}',   // onelo_pk_live_* — public app identifier
  apiUrl: '{{apiUrl}}',
  // Optional → surface as Release / App build / Environment columns:
  // appVersion: '1.2.0', appBuild: '420', environment: 'production',
})
// Crash capture: AUTOMATIC in the browser (window 'error' + 'unhandledrejection').
// SSR / Node: there is NO global handler — track() server operations explicitly,
//   or forward process.on('uncaughtException'/'unhandledRejection') into
//   onelo.monitor.event('unhandled', { ok: false, error: String(e) }).
// Teardown: onelo.monitor.destroy() on graceful Node shutdown; an SPA needs none.
```
`onelo.monitor` exposes `track(name, fn, {meta})`, `event(name, {ok,error,meta})`,
`setUserId(id|null)`, `flush()`, `destroy()`. There is **no `capture()`** — record
a caught error with `event({ ok: false, error })`. See [js.md](js.md).
