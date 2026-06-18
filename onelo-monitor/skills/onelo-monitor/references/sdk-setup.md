# SDK setup — install, version, init (Swift & Python)

Install / update the Onelo SDK and initialise Monitor. Read this in Phase 0
(present & current) and Phase 5 (init), for the two SDKs this skill supports.

## Contents
- Install the SDK
- Keep the SDK current (version check)
- Initialize Monitor — Swift
- Initialize Monitor — Python

## Install the SDK
The Onelo SDK ships all modules (auth, features, paywall, monitor) in ONE package,
so if auth/features are already wired, the package is present — Monitor just needs
initialising (see below).

| Stack | Install |
|---|---|
| Swift (iOS / macOS) | Swift Package — `https://github.com/onelo-tools/onelo-swift` (branch: `staging`) |
| Python (FastAPI / Django / Flask / Litestar) | `pip install "git+https://github.com/onelo-tools/onelo-python.git@staging"` — extras: `onelo[fastapi]` / `[django]` / `[flask]` / `[litestar]` |

## Keep the SDK current (version check)
The snippets this skill inserts may use APIs added in newer versions —
instrumenting against a stale SDK can insert calls the installed version lacks.

| Stack | Show installed version | Update |
|---|---|---|
| Swift | grep `onelo-swift` in `Package.resolved` | Xcode → File → Packages → Update to Latest, or `swift package update` |
| Python | `pip show onelo` | `pip install -U "git+https://github.com/onelo-tools/onelo-python.git@staging"` |

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
```python
from onelo import Onelo, monitor

onelo = Onelo(publishable_key="{{publishableKey}}", api_url="{{apiUrl}}")
monitor.init(onelo=onelo, environment="production", install_excepthook=True)

# FastAPI / Starlette / Litestar (ASGI):
from onelo.monitor.integrations import OneloMonitorASGIMiddleware
app.add_middleware(OneloMonitorASGIMiddleware)
# Django:  add "onelo.monitor.integrations.django.OneloMonitorMiddleware" to MIDDLEWARE
# Flask:   from onelo.monitor.integrations.flask import install; install(app)
```
