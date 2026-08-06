# Onelo Monitor — Electron, React Native, Android/Kotlin, Flutter

Per-platform signals, primitives, grep patterns and fix syntax for the four SDKs
that share this file. The language-agnostic audit rules (A–G, including A2) live
in SKILL.md.

All four SDKs expose the same three primitives — `track` / `event` / `capture` —
verified against `packages/onelo-electron`, `packages/onelo-react-native`,
`packages/onelo-android`, `packages/onelo-flutter`. Only the syntax and the
crash-capture story differ.

## Contents
- Detect signals
- Primitives per platform
- Grep patterns
- Coverage scan patterns
- Crash capture — what is automatic, what is NOT
- Insertion rules
- Snippet source

## Detect signals
| Platform | SDK signal | Extra |
|---|---|---|
| Electron | `from '@onelo/electron'`, `new Onelo({ publishableKey, apiUrl, bundleId })` | main process only — `electron` in `package.json`, a `main.ts`/`main.js` entry |
| React Native | `from '@onelo/react-native'`, `new Onelo({ … })` | `react-native` in `package.json`, `App.tsx` |
| Android / Kotlin | `com.onelo.android.Onelo`, `OneloConfig(...)` | `build.gradle.kts`, `Application` subclass |
| Flutter | `package:onelo/onelo.dart`, `Onelo(publishableKey: …)` | `pubspec.yaml`, `void main()` |

## Primitives per platform
All: **`track` measures duration, emits ok on return / error on throw, and
re-throws.** Preserve the assignment and the surrounding error handling.

**Electron / React Native** (identical TS shape, same as `@onelo/js` plus a real
`capture`):
```ts
const r = await onelo.monitor.track('checkout', () => pay(), { meta: { plan } })
onelo.monitor.event('tab_viewed', { ok: true, meta: { tab: 'export' } })
onelo.monitor.capture(err, { featureName: 'import', meta: { source: 'csv' } })
```

**Android / Kotlin** — `track` is a **suspend fun**, call it inside a coroutine
scope (`lifecycleScope` / `viewModelScope`); `event` and `capture` are regular:
```kotlin
val r = onelo.monitor.track("checkout", meta = mapOf("plan" to plan)) { pay() }
onelo.monitor.event("tab_viewed", ok = true, meta = mapOf("tab" to "export"))
onelo.monitor.capture(e, featureName = "import", meta = mapOf("source" to "csv"))
```

**Flutter / Dart** — the operation is the 2nd positional arg; `event` takes a
`MonitorEventOptions`:
```dart
final r = await onelo.monitor.track('checkout', () => pay(), meta: {'plan': plan});
onelo.monitor.event('tab_viewed', MonitorEventOptions(ok: true, meta: {'tab': 'export'}));
onelo.monitor.capture(e, featureName: 'import', stack: stack, meta: {'source': 'csv'});
```

## Grep patterns
Call sites + feature names + init:
```bash
# Electron / React Native
grep -rnE "\.monitor\.(track|event|capture)\(" --include=*.ts --include=*.tsx --include=*.js --include=*.jsx .
grep -rnE "new Onelo\(" --include=*.ts --include=*.tsx .
# Android / Kotlin
grep -rnE "\.monitor\.(track|event|capture)\(" --include=*.kt .
grep -rnE "OneloConfig\(" --include=*.kt .
# Flutter / Dart
grep -rnE "\.monitor\.(track|event|capture)\(" --include=*.dart .
grep -rnE "Onelo\(\s*publishableKey" --include=*.dart .
```

## Coverage scan patterns (Phase 4 step 2 — run AFTER the user-facing feature list)
```bash
# operations worth track()
grep -rnE "async function|await |fetch\(|axios|ipcMain\.handle|new Worker\(" --include=*.ts --include=*.tsx .   # Electron / RN
grep -rnE "suspend fun|withContext\(|Retrofit|OkHttp|\.enqueue\(" --include=*.kt .                              # Android
grep -rnE "Future<|await |http\.|Dio\(|rootBundle|File\(" --include=*.dart .                                     # Flutter
# error sites
grep -rnE "\} ?catch ?\(|\.catch\(|throw " --include=*.ts --include=*.tsx .
grep -rnE "catch \(|throw " --include=*.kt .
grep -rnE "catch \(|on [A-Z][A-Za-z]* catch|throw " --include=*.dart .
# background work where a failure vanishes
grep -rnE "void [a-zA-Z]+\(|\.then\(|setTimeout\(|setInterval\(" --include=*.ts --include=*.tsx .
grep -rnE "GlobalScope\.launch|lifecycleScope\.launch|viewModelScope\.launch|Thread\(|WorkManager" --include=*.kt .
grep -rnE "unawaited\(|scheduleMicrotask\(|Timer\(|compute\(|Isolate\." --include=*.dart .
```

## Crash capture — what is automatic, what is NOT
| Platform | Automatic | NOT automatic — you must instrument |
|---|---|---|
| Electron | unhandled **exceptions** → `global_error` | unhandled **promise rejections** are deliberately NOT hooked (a listener would suppress Node's crash-on-rejection). Wrap risky async work in `track()`. **Renderer code is not covered at all** — the SDK runs in the main process only; send from renderer via an IPC handler. |
| React Native | unhandled JS errors via `ErrorUtils` → `global_error`, chaining the existing handler (red-box still runs) | native-side crashes; anything inside a swallowed `.catch` |
| Android | see the SDK's init; assume manual coverage for coroutine failures | a `launch { }` whose exception nobody handles — wrap the body in `track()` |
| Flutter | `FlutterError.onError` + `PlatformDispatcher.instance.onError` hooked at init, chaining any existing handler | if you set your OWN `FlutterError.onError` AFTER creating Onelo it REPLACES the hook — call `onelo.monitor.registerGlobalHandlers()` again (safe no-op if unchanged) |

## Insertion rules
- **track()** wraps the existing call, body inside the callback; it re-throws, so
  never change the surrounding `try/catch`. Keep the `await` and the assignment.
  Android: the call site must already be in a coroutine — if it isn't, that is a
  code change the developer must approve, not something to slip in.
- **event()** on the line where the instantaneous thing happens.
- **capture()** inside the existing `catch`; if the code re-threw before, keep the
  throw. Flutter: pass `stack:` from `catch (e, stack)` — without it you lose the
  trace.
- **ONE instance**, created where the snippet says (Electron: main process;
  RN: module level; Android: `Application`, never an Activity; Flutter: `main()`
  before `runApp`). Re-creating resets the session id and drops the buffer.
- **Rule A2 applies especially here** — mobile/desktop flows that return `null`
  on both cancel and breakage (auth sheets, file pickers, permission prompts,
  deep links) are the most common false green. Throw inside the callback.

## Snippet source
Exact `install` / `init` / `usage` for each platform lives in its snippet file —
see the platform table in SKILL.md. Those are baked from `@onelo/snippets` at
publish time. Take the syntax from there and replace the feature name; the
examples above are for judging NAMES and PRIMITIVES, not for copying verbatim.
