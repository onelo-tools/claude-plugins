# React Native

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview react-native-get-random-values
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// PKCE needs a cryptographically-secure RNG. React Native has no built-in Web
// Crypto, so import this polyfill as the FIRST line of your app ENTRY (index.js),
// before anything else — it backs crypto.getRandomValues with the platform CSPRNG
// (iOS SecRandomCopyBytes / Android SecureRandom). Without it, sign-in throws.
import 'react-native-get-random-values'
import { Onelo } from '@onelo/react-native'

// ONE instance owns auth, features, consent, store and paywall — they SHARE the
// session. Always use onelo.auth: a second `new Onelo(...)` is a DIFFERENT
// session, so the consent gate's "Sign out" and your auth listener would run on
// different instances and the app would look like sign-out did nothing.
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp', // pick a scheme UNIQUE to your app + register it natively (see usage below)
  // Your app id / package name, sent as X-Bundle-Id for bundle attestation.
  // iOS auto-derives it from the native module, but ANDROID needs it here —
  // without it, requests are rejected (bundle_id_mismatch) once enforcement is on.
  bundleId: 'com.company.app',
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { OneloAuthGate, ConsentGateModal } from '@onelo/react-native'

// ── REQUIRED NATIVE SETUP: register your deep-link scheme ────
// Social sign-in (paid plans; and store upgrades / some Stripe redirects) hands off to the
// SYSTEM BROWSER, which returns to your app via the callbackScheme you set above.
// Register the SAME scheme natively or the browser can't reopen the app and sign-in
// never finishes (you'd be stuck on a "Return to the app" page):
//   • Android — android/app/src/main/AndroidManifest.xml, inside your <activity>:
//       <intent-filter>
//         <action android:name="android.intent.action.VIEW" />
//         <category android:name="android.intent.category.DEFAULT" />
//         <category android:name="android.intent.category.BROWSABLE" />
//         <data android:scheme="myapp" />
//       </intent-filter>
//   • iOS — ios/<App>/Info.plist → URL Types → URL Schemes: myapp
// Plain in-app WebView sign-in works without this; the system-browser return needs it.

// ── Flagship: declarative auth + paywall gate ───────────────
// OneloAuthGate is the React Native analogue of Swift's OneloAuthView. Wrap your
// app in it and it drives the CENTRALLY-HOSTED sign-in page automatically — no
// button, no custom form, no branded pre-auth screen (just a neutral background
// while it resolves). It reveals children ONLY when the user is signed in AND
// (the app has no paywall OR they hold active access). Mount <ConsentGateModal>
// near the root so a blocking Terms/Privacy update can draw over the app.
function App() {
  return (
    <>
      <OneloAuthGate onelo={onelo}>
        {/* Your app — shown only when signed in AND (no paywall OR entitled) */}
        <RootNavigator />
      </OneloAuthGate>
      <ConsentGateModal onelo={onelo} />
    </>
  )
}

// ── Paywall hard-gate (apps that sell access) ───────────────
// This is the entitlement wall. OneloAuthGate reveals children ONLY when signed
// in AND (no paywall OR entitled). For a paywalled app it routes a
// signed-in-but-unentitled user to the hosted STORE and keeps children hidden
// until they buy — closing the store WITHOUT purchasing leaves them on the
// sign-in / store surface, never inside your app. This is the SAME entitlement
// the backend enforces server-side on every call. If you build your OWN gate
// instead of OneloAuthGate, check both yourself — never render paid UI just
// because sign-in happened:
const paywalled = onelo.auth.isPaywallEnabled()
const entitled = !paywalled || (await onelo.auth.hasActiveAccess())
if (entitled) {
  // → render your app
} else {
  // Not entitled — send them to the hosted store (see the Store tab):
  //   await onelo.store.open()
}

// ── Sign out / return to login ──────────────────────────────
// signOut() clears the session (and revokes it server-side); OneloAuthGate
// re-shows the hosted sign-in page automatically. Subscribe to auth changes to
// react to REMOTE revokes too (refund, ban, account-deletion cron) — the session
// becomes null and OneloAuthGate returns to sign-in with no code from you:
const unsub = onelo.auth.onAuthStateChange((session) => {
  if (!session) { /* signed out — OneloAuthGate shows sign-in again */ }
})
await onelo.auth.signOut()

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// <ConsentGateModal onelo={onelo} /> (mounted above) auto-presents a BLOCKING
// full-screen gate — the hosted document in a WebView — when you publish a
// version with "Require in-app acceptance" ON (works for BOTH Terms of Service
// AND Privacy Policy). The user must Accept & Continue, or Sign out. It checks on
// sign-in, on every app-foreground, and the instant you publish (over SSE) — no
// polling, no restart. "Notify only" versions never block. Nothing else to wire.
//
// Custom UI instead of the hosted gate (opt out of auto-present):
//   const onelo = new Onelo({ ..., autoPresentConsentGate: false })
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const pending = await onelo.consent.requiredConsents()
//   const mustAccept = pending.filter(c => c.blocking)   // blocking, past effective
//   await onelo.consent.acceptConsent(pending[0].versionId)

// ── If using your own auth system ───────────────────────────
// When you have your own user database, call identify() after your login so the
// features SDK can apply per-user/per-plan targeting. Without it, targeted
// features fall back to "hidden" and you'll see a console warning at runtime.
await onelo.identify(currentUser.id)

// ── Manual mount (lower-level, without OneloAuthGate) ───────
// Prefer OneloAuthGate above. If you must drive the hosted flow yourself, mount
// <AuthModal> and open it with loadAuthView() (auto-routes waitlist / paywall
// store); useModalState() re-renders when the modal opens/closes:
//   import { AuthModal, useModalState } from '@onelo/react-native'
//   const modal = useModalState(onelo.auth)
//   <AuthModal visible={modal.visible} hostedUrl={modal.url}
//              onResult={modal.onResult ?? (() => {})} />
//   const session = await onelo.auth.loadAuthView()
```
<!-- /onelo:snippet -->
