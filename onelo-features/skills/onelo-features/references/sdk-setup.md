# SDK setup (post-instrumentation)

How to install / keep current / wire the Onelo SDK. Read this in Phase 0 (install
+ version check) and Phase 6 (init + setup, once instrumentation is done).

## Contents
- Install the SDK
- Keep the SDK current (version check)
- Initialize once at app startup (per-language init)
- Declare known features upfront
- Keeping the registry in sync (build hook)
- Identify the user
- Real-time delivery
- What you'll see in the dashboard

## Install the SDK

Onelo SDKs ship with all modules — auth, features, paywall, monitor — in
one package, so a single init covers everything you instrumented.

**Every SDK installs from GitHub (`onelo-tools/*`) — NOT the npm registry or
pub.dev.** Use the `staging` ref while testing; drop it (or use the default
branch) for production.

| Stack | Install (from GitHub) |
|---|---|
| JavaScript / TypeScript | `npm install github:onelo-tools/onelo-js` (staging: append `#staging`) |
| Swift (iOS / macOS) | Xcode → Add Package Dependencies → `https://github.com/onelo-tools/onelo-swift` (branch `staging`) |
| Android (Kotlin) | JitPack (builds from GitHub): in `settings.gradle` add `maven { url "https://jitpack.io" }`, then `implementation("com.github.onelo-tools:onelo-android:+")` |
| Flutter | `pubspec.yaml` git dep: `onelo_flutter:` → `git: { url: https://github.com/onelo-tools/onelo-flutter, ref: staging }` |
| Python (FastAPI / Django / Flask / aiohttp) | `pip install "git+https://github.com/onelo-tools/onelo-python.git@staging"` |
| Go (net/http / Gin / Echo / Fiber / Chi / gRPC) | `go get github.com/onelo-tools/onelo-go@staging` |

## Keep the SDK current (version check)

Before instrumenting, make sure the installed SDK is recent — the snippets this
skill inserts may use APIs added in newer versions. Show the installed version,
and if it's behind (or absent), install/update from the ref in the table above.

| Stack | Show installed version | Update to latest |
|---|---|---|
| JavaScript / TypeScript | `npm ls @onelo/js` (shows resolved github commit) | re-run `npm install github:onelo-tools/onelo-js#staging` (the github ref fetches newest) |
| Swift (iOS / macOS) | grep `onelo-swift` in `Package.resolved` | Xcode → File → Packages → Update to Latest, or `swift package update` |
| Kotlin (Android) | the version in `build.gradle(.kts)` | bump the JitPack coordinate (`com.github.onelo-tools:onelo-android`) to the latest tag |
| Flutter | `flutter pub deps \| grep onelo_flutter` (or `pubspec.lock`) | `flutter pub upgrade onelo_flutter` |
| Python | `pip show onelo` | `pip install -U "git+https://github.com/onelo-tools/onelo-python.git@staging"` |
| Go | `go list -m github.com/onelo-tools/onelo-go` | `go get -u github.com/onelo-tools/onelo-go@staging` |

If you can't confirm the version, update anyway — instrumenting against a stale
SDK can insert calls the installed version doesn't have.

## Initialize once at app startup

The instance is what every inserted `onelo.features.feature(...)` call reaches for.

> **`baseURL` matters.** Copy the URL from the dashboard SDK page
> (`app.onelo.tools` → your app → SDK), which renders the right one for
> production (`https://app.onelo.tools`) or staging
> (`https://st.backend.onelo.tools`). Hardcoding production while you're
> on a staging tenant (or vice versa) silently fails — requests land on
> the wrong backend with no useful error.

### JavaScript / TypeScript
```ts
import { Onelo } from '@onelo/js'
import { FEATURE_REGISTRY } from './generated/feature-registry'

export const onelo = new Onelo({
  publishableKey: 'pk_live_...',
  baseURL: 'https://app.onelo.tools',
  // featureDefaultStatus: 'enabled',  // optional — see "Dev-mode default" in troubleshooting.md
})

// Register every instrumented feature upfront — see "Declare" below for why.
onelo.features.declare([...FEATURE_REGISTRY])
```

### Swift (SwiftUI App entry)
```swift
import OneloSwift

#if DEBUG
let onelo = Onelo(
    publishableKey: "pk_live_...",
    callbackScheme: "myapp",
    baseURL: URL(string: "https://app.onelo.tools")!,
    featureDefaultStatus: .enabled  // dev: new gates render until enabled in dashboard
)
#else
let onelo = Onelo(
    publishableKey: "pk_live_...",
    callbackScheme: "myapp",
    baseURL: URL(string: "https://app.onelo.tools")!
)
#endif

// Register every instrumented feature upfront — see "Declare" below.
onelo.features.declare(FeatureRegistry.all)
```

For AppKit projects without a SwiftUI root, hold the instance on the app
delegate (`AppDelegate.shared.onelo`) and reach for it from views as
`AppDelegate.shared.onelo.features.feature(...)`. Onelo's modules are
not `ObservableObject`, so they don't slot naturally into `@StateObject`
or `@EnvironmentObject` — a singleton-on-delegate pattern is idiomatic
for mixed AppKit/SwiftUI codebases.

### Kotlin (Android `Application` class)
```kotlin
import com.myapp.generated.FeatureRegistry

class MyApp : Application() {
    lateinit var onelo: Onelo
    override fun onCreate() {
        super.onCreate()
        onelo = Onelo(
            publishableKey = "pk_live_...",
            baseURL = "https://app.onelo.tools",
            // featureDefaultStatus = if (BuildConfig.DEBUG) FeatureStatus.ENABLED else FeatureStatus.HIDDEN,
        )
        // Register every instrumented feature upfront — see "Declare" below.
        onelo.features.declare(FeatureRegistry.ALL)
    }
}
```
Reach via `(applicationContext as MyApp).onelo.features.feature("...")`,
or expose through Hilt/Koin if you use DI.

### Flutter (`main.dart`)
```dart
import 'package:flutter/foundation.dart';
import 'package:onelo_flutter/onelo_flutter.dart';
import 'generated/feature_registry.dart';

final onelo = Onelo(
  publishableKey: 'pk_live_...',
  baseUrl: 'https://app.onelo.tools',
  // featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
);

void main() {
  // Register every instrumented feature upfront — see "Declare" below.
  onelo.features.declare(featureRegistry);
  runApp(MyApp());
}
```
For Provider/Riverpod/BLoC projects, expose the instance through your DI of choice and read it from widgets accordingly.

## Declare known features upfront (already wired above)

The init samples above each call `features.declare(...)` right after
construction. This explains *why*, and how the registry constant is kept current.

**Why declare:** lazy discovery via `feature("...")` only registers a name once
that code path actually runs. For non-trivial apps this leaves rarely-visited
gates — debug screens, admin-only handlers, error boundaries, A/B variants no
user is bucketed into — **invisible in the dashboard** until someone triggers
them. `declare(...)` batch-pings every known name on init, so they all appear
from the first run regardless of runtime traversal.

**Where the list comes from:** Phase 5b writes a generated registry file
(`FeatureRegistry.swift` / `feature-registry.ts` / `FeatureRegistry.kt` /
`feature_registry.dart`) plus a generator script. The init samples import the
registry from that file — nothing to type by hand. Triggers (Phase 2.5 Rule 0)
appear under their destination's feature name.

## Keeping the registry in sync (build hook)

The registry file is generated, not hand-maintained. Wire the generator into your
build pipeline so it refreshes as new `feature("...")` calls are added. The exact
snippet to drop into your build config lives in the language reference file's
"Build hook integration" subsection:

- Swift / Xcode → `swift-macos-patterns.md` / `swift-ios-patterns.md`
- JS / TS / Next.js / Vite → `js-patterns.md`
- Kotlin / Android Gradle → `kotlin-patterns.md`
- Flutter / Dart → `flutter-patterns.md`
- Python → `python-patterns.md`
- Go → `go-patterns.md`

Wire it once — every build refreshes the registry from current source, so the
dashboard list stays in lock-step with what the codebase instruments.

## Identify the user (only if you handle auth yourself)

If you use Auth0, Firebase, or your own backend, call `onelo.identify(userId)`
after login. Without it, per-user and per-plan targeting can't apply and the SDK
logs a one-time warning. If you use **Onelo Auth**, identify happens
automatically — skip this.

## Real-time delivery

When you flip a feature in the dashboard and click Deploy, every connected SDK
instance receives the update over a long-lived SSE connection within
milliseconds. Nothing to poll, refresh, or cache-bust manually.

## What you'll see in the dashboard

Newly instrumented features appear in **Dashboard → Features → Registry** on first
run, tagged **"JUST ADDED"** for the first 24 hours. Configure visibility
(`active` / `hidden` / `greyed` / `upsell` / `targeted`) per feature, then click
Deploy to push the changes to live clients.
