# Troubleshooting

Common pitfalls Onelo developers hit during their first integration. If you (or
the developer you're helping) are debugging unexpected behavior, check these first.

## Contents
- Fail-closed default is by design
- A gated button flashes then disappears then reappears on login / cold start
- macOS App Attest hang on dev builds
- Keychain errSecDuplicateItem after rebuilds
- Hosted-auth WebView opens then disappears
- TCC permission re-prompts on every dev build
- body re-renders forever after instrumenting a SwiftUI view

## Fail-closed default is by design

A freshly instrumented `feature("...")` returns the SDK's default status (usually
`hidden` / `disabled`) until you enable it in the dashboard. This is intentional —
it prevents code that ships before the dashboard is configured from leaking
unfinished UI to real users.

If you want new gates to render their content while you build them locally, pass
`featureDefaultStatus: .enabled` (or the language equivalent) in the SDK init,
gated by your debug flag. Each language reference file shows the exact syntax
under "Dev-mode default".

## A gated button flashes then disappears then reappears on login / cold start

This looks like a caching bug in Onelo. It usually isn't — the cache is
per-user (feature status depends on the signed-in identity/plan), so on a
fresh load or right after login the SDK doesn't yet know WHICH user's cached
snapshot to restore until `identify()` (or Onelo Auth's own session resolve)
finishes its network round-trip. If the gated component reads `feature(...)`
before that settles, and treats "not yet known" the same as `hidden` — the
almost-universal bug — the button vanishes for the resolve window, then pops
back in once the real per-user state lands. Users read this as "the button
disappeared."

Fix: don't read features before `onelo.features.ready(timeoutMs)` resolves
(or, if you can't block first paint, track a separate loading flag and render
a skeleton — never fold "unknown yet" into the `hidden` branch). Every SDK
(JS/React/Electron/React Native, Swift, Android/Kotlin, Flutter) exposes
`ready()` for exactly this; see "Avoid the first-paint flicker" in the
matching `*-patterns.md` reference file for the exact call shape.

## macOS App Attest hang on dev builds

Older SDK versions (< 3.18.0) could hang on `DCAppAttestService.attestKey` for
Developer ID-signed macOS apps that ship without an embedded provisioning profile.
The SDK now adds a 5-second timeout and an entitlement check; bump to ≥ 3.18.0 if
you see the symptom (login UI never appears, attestation pegs CPU forever).

## Keychain errSecDuplicateItem after rebuilds

Ad-hoc-signed dev builds get a fresh `cdhash` on every `swift build`, which
invalidates the ACL on previously-stored keychain items. Older SDKs (< 3.17.1)
failed `SecItemAdd` because `SecItemDelete` couldn't reach the existing entry. The
SDK now falls back to `SecItemUpdate` on duplicates; bump to ≥ 3.17.1.

## Hosted-auth WebView opens then disappears

If the WKWebView's content process crashes (sandbox kill, OOM, system pressure),
the window goes blank with no signal back to the user. The SDK now recovers
automatically — silent reload on first crash, error message on the second; bump
to ≥ 3.18.2.

## TCC permission re-prompts on every dev build

Each ad-hoc rebuild generates a new `cdhash`, and macOS treats that as a distinct
app for TCC purposes — Camera, Microphone, Files & Folders, etc. permissions get
re-prompted every launch. Workarounds: use a stable Developer ID for daily dev
builds, or deploy to TestFlight for testers. This is a macOS / TCC concern, not
specific to Onelo.

## body re-renders forever after instrumenting a SwiftUI view

Older SDKs (< 3.18.1) made `OneloFeatures` properties trigger view invalidation
when the SDK touched its internal discovery set, producing an infinite render loop
in any view that called `feature(...)` from `body`. The SDK now marks all internal
state as `@ObservationIgnored`; bump to ≥ 3.18.1.
