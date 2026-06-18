# Troubleshooting

Common pitfalls Onelo developers hit during their first integration. If you (or
the developer you're helping) are debugging unexpected behavior, check these first.

## Contents
- Fail-closed default is by design
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
