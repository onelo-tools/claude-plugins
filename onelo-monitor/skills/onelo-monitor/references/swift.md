# Onelo Monitor — Swift (iOS & macOS)

Per-language signals, primitives, grep patterns and fix syntax. The
language-agnostic audit rules (A–G) live in SKILL.md.

## Contents
- Detect signals
- Primitives (track / event / capture) and when to use each
- Grep patterns (call sites, feature names, init)
- Insertion rules
- iOS vs macOS (flush lifecycle + crash capture — the only platform sub-split)
- Snippet fetch + fallback
- Good vs bad examples

## Detect signals
- **SDK:** `import OneloSwift`, an `Onelo(publishableKey: …)` init, and use of
  `onelo.monitor`. Package: `onelo-swift` via Swift Package Manager.
- **Platform** (drives the iOS-vs-macOS subsection only — NOT a separate file):
  ```bash
  grep -rlnE "import AppKit|NSApplicationDelegate|MenuBarExtra" --include=*.swift .   # macOS
  grep -rlnE "import UIKit|UIApplicationDelegate|UIViewController" --include=*.swift . # iOS
  ```
  AppKit only → macOS. UIKit only / pure SwiftUI → iOS. Both → multiplatform
  (apply both lifecycle notes).

## Primitives — pick the right one
| Want to record | Use | Emits |
|---|---|---|
| an operation (has duration / can fail) | `track(_:meta:_:)` — sync or async | `source=track`, `ok+duration` on success / `error+duration` on throw; **re-throws** |
| an instantaneous thing (click, toggle, view) | `event(_:options:)` | `source=event` |
| an error you already caught | `capture(_:featureName:meta:)` | `source=event`, `ok=false` + stack + breadcrumbs + flags |

Decision: awaited / throwing / network / disk → `track()`. No duration, just a
marker → `event()`. Inside a `catch` for an error you handle yourself →
`capture()`. `MonitorEventOptions(ok:durationMs:error:meta:)`.

## Grep patterns
Call sites:
```bash
grep -rnE "\.monitor\.(track|event|capture)\(" --include=*.swift .
```
Instantaneous events (candidates for Rule C / Rule F — inspect for nearby `await`
or `ok: false`):
```bash
grep -rnE "\.monitor\.event\(" --include=*.swift .
```
Feature names (first string arg):
```bash
grep -rhoE "\.monitor\.(track|event|capture)\(\s*\"[a-zA-Z0-9_]+\"" --include=*.swift . \
  | grep -oE "\"[a-zA-Z0-9_]+\""
```
Init present:
```bash
grep -rnE "Onelo\(\s*publishableKey" --include=*.swift .
```

## Coverage scan patterns (Phase 4)
Operations worth `track()` (awaited / throwing / I/O):
```bash
grep -rnE "func .*(async|throws)|try await|URLSession|\.dataTask\(|FileManager|Data\(contentsOf:" --include=*.swift .
```
Error sites (where failures should be captured):
```bash
grep -rnE "\} catch|try\?|try!|throw " --include=*.swift .
```
Background threads / tasks where a thrown error vanishes:
```bash
grep -rnE "Task\s*\{|Task\.detached|DispatchQueue\.[A-Za-z]+\.async|withTaskGroup|\.async\s*\{" --include=*.swift .
```
Already instrumented — skip (a monitor call within ~3 lines):
```bash
grep -rnE "\.monitor\.(track|event|capture)\(" --include=*.swift .
```
**Quality note:** an unguarded `Task { try await … }` swallows thrown errors —
they never surface anywhere. Wrap the body in `track()` (or `capture()` in its
`catch`) so background failures aren't invisible.

## Insertion rules
- **track()** wraps an existing call — keep the body, wrap it:
  - sync: `let x = try onelo.monitor.track("name", meta: […]) { try doWork() }`
  - async: `let x = try await onelo.monitor.track("name") { try await doWork() }`
  - `track` is `@discardableResult` and returns the closure's result — preserve
    the assignment. It re-throws, so existing `do/catch` keeps working; never
    change the surrounding error handling.
- **event()** goes on the line where the instantaneous thing happens.
- **capture()** goes inside the existing `catch`.
- Never wrap a closure that isn't the operation itself.

## iOS vs macOS  (the only platform sub-split)
Crash capture (NSException + MetricKit), HTTP auto-trace, breadcrumbs, and the
event payload are identical. Two things differ:

- **Flush lifecycle:**
  - macOS: call `onelo.monitor.destroy()` in `applicationWillTerminate`.
  - iOS: `applicationWillTerminate` is unreliable — also `onelo.monitor.flush()`
    on `UIApplication.willResignActiveNotification` or ScenePhase `.background`,
    or events can be lost when the app is suspended.
- **MetricKit crashes** are delivered on the NEXT launch, on REAL devices only
  (never the simulator), up to ~24h later. Don't expect simulator crash rows;
  don't treat their absence as "no crashes".

## Snippet fetch (Phase 4)
```bash
curl -sf "${ONELO_API_BASE:-https://app.onelo.tools}/api/snippets?sdk=monitor&lang=swift"
```
Use the `usage` field and replace the feature name. Fallback if the fetch fails:
```swift
// operation (async)
let data = try await onelo.monitor.track("pdf_upload", meta: ["size": file.size]) {
    try await upload(file)
}
// instantaneous
onelo.monitor.event("tab_viewed", options: .init(ok: true, meta: ["tab": "export"]))
// already-caught error
onelo.monitor.capture(error, featureName: "checkout")
```

## Good vs bad (examples pattern)
- `event("backend_error", .init(ok: false))` → `try onelo.monitor.track("backend_call") { try call() }`  *(Rules A+B)*
- `event("permission_denied", .init(ok: false))` → `event("permission_check", .init(ok: false, error: "denied"))`  *(Rule B)*
- `event("export_started"); …; event("export_completed")` → `track("export") { … }`  *(Rule E)*
- `track("ai_response") { … }` with no meta → add `meta: ["model": "gpt-4"]`  *(Rule D)*
