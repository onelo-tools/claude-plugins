# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=electron field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=electron field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/electron'
import { app, BrowserWindow } from 'electron'

// 1. Register your deep-link protocol (do this before app is ready)
//    Replace 'myapp' with your own scheme — must be unique to your app.
app.setAsDefaultProtocolClient('myapp')

// 2. Initialize the SDK in your Electron main process. ONE instance owns auth,
//    features, consent and paywall — they SHARE the session. Use onelo.auth
//    everywhere: a separate `new OneloElectronAuth(...)` is a DIFFERENT session,
//    so the consent gate's "Sign out" would fire on onelo.auth and your
//    onSessionChange handler (on the other instance) would never run — the app
//    would look like sign-out did nothing.
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  protocol: 'myapp', // must match the scheme above
  // Your app id (same on macOS + Windows). REQUIRED for codesign attestation:
  // the SDK sends it on every request and a code-signed build without it is
  // rejected (bundle_id_mismatch) once enforcement is active. Match your
  // electron-builder appId / CFBundleIdentifier.
  bundleId: 'com.company.app',
})
const auth = onelo.auth // shorthand — the SAME instance the consent gate signs out
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// 3. Auto-show auth on app launch — no button needed.
//
// This is your paywall hard-gate: a user who signs in but never completes a
// purchase never reaches your app — the window stays hidden until they hold
// access. (Swift's OneloAuthView gates its content for you; on web you gate on
// the RESULT of loadAuthView(); Electron can't touch your BrowserWindow, so
// YOU own the gate — this is it.)
//
// ONE gate decides whether the app window may appear. "Allowed in" = signed in
// AND, on a paywalled app, holding active access. Do NOT treat presentAuthWindow()
// returning as proof of entry: closing the store without buying, a declined
// consent gate, or an OAuth hand-off to the system browser all resolve WITHOUT a
// session — so re-check real state, never the return value. (Electron, unlike the
// native SDKs, has no auth wrapper that re-shows login for you; this is it.)
let mainWindow            // created on ready (below)
let gating = false        // re-entrancy guard — never stack two auth windows

async function enforceGate() {
  if (!mainWindow || gating) return
  gating = true
  try {
    await auth.whenReady().catch(() => {})  // config loaded → paywall flag is accurate
    const allowedIn = async () => {
      const session = await auth.getSession()
      if (!session) return false                              // not signed in
      // Only paywalled apps require active access; others enter on a session alone.
      return !auth.isPaywallEnabled() || (await auth.hasActiveAccess())
    }
    if (await allowedIn()) { mainWindow.show(); return }
    mainWindow.hide()
    // null = standalone auth window (no app window behind it). May resolve with
    // NO session (store closed without buying / OAuth punted to the browser) —
    // that's why we re-check below instead of showing unconditionally.
    // presentAuthWindow shows its own graceful "Try again" window on a flow
    // failure (attestation / offline), so it won't throw for those — but a
    // revoked key still throws. Catch it so enforceGate never leaks an unhandled
    // rejection that would leave mainWindow permanently hidden (blank app).
    await auth.presentAuthWindow(null)
    if (await allowedIn()) mainWindow.show()   // still not allowed in → stay hidden
  } catch (err) {
    console.warn('[Onelo] auth gate error — keeping app window hidden:', err)
  } finally {
    gating = false
  }
}

// Re-run the gate on launch, on every sign-in/out (revoke, session expiry, a
// declined consent gate — saveSession/signOut both fire this), and after an
// OAuth deep-link exchange.
auth.onSessionChange(() => { void enforceGate() })

// OAuth finishes in the system browser and returns via your deep-link scheme.
// Exchanging it fires onSessionChange → enforceGate reveals the app. Registered
// at top level so a cold-start deep-link isn't missed.
app.on('open-url', (_event, url) => {
  // A present-but-failed code exchange throws — swallow it so the gate stays put
  // (the user retries); onSessionChange reveals the app only on a real session.
  auth.handleDeepLink(url).catch(() => {})
})

app.whenReady().then(async () => {
  mainWindow = new BrowserWindow({ show: false /* your options */ })
  mainWindow.loadFile('index.html')  // load now; the gate decides WHEN to show
  await enforceGate()
})

// Sign out — the onSessionChange handler above returns the app to sign-in.
await auth.signOut()

// ── If using your own auth system ───────────────────────────
// When you have your own user database, features SDK still needs to know who is
// signed in. Call identify() after your login so per-user/per-plan targeting works.
// Without it, targeted features fall back to "hidden" and you'll see a console warning.
await onelo.identify(currentUser.id)

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// Uses the full Onelo SDK — const onelo = new Onelo({ ... }) — which exposes
// onelo.consent (same instance as features/paywall). When you publish a version
// with "Require in-app acceptance" ON in the dashboard — this works for BOTH
// Terms of Service AND Privacy Policy — signed-in users get a BLOCKING gate they
// must accept to keep using the app; Decline signs them out. "Notify only"
// versions never block.
//
// REQUIRED for a true hard block: register your main window so the gate opens
// MODALLY over the app. Without it the gate can't block the app and the SDK logs
// a warning (Electron can't hide your window for you the way native SDKs do).
onelo.consent.setGateParent(mainWindow)
// The gate then auto-presents on sign-in AND the instant you publish a blocking
// version (pushed live over SSE) — no polling, no restart.
//
// Belt-and-braces (recommended): also gate your OWN post-login UI before showing it.
const pending = await onelo.consent.requiredConsents()
if (pending.some(c => c.blocking)) {
  await onelo.consent.presentGateIfNeeded(mainWindow)   // resolves after Accept
}
//
// Custom UI instead of the hosted gate (new Onelo({ autoPresentConsentGate: false })):
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const p = await onelo.consent.requiredConsents()
//   await onelo.consent.acceptConsent(p[0].versionId)
```
<!-- /onelo:snippet -->
