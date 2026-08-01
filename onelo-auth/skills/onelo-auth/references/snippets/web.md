# Web — JavaScript / TypeScript

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=npm field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=npm field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'

// Initialize once (e.g. in a module-level singleton)
const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=npm field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Sign in — pick ONE presentation. loadAuthView() auto-routes
// waitlist / paywall-store automatically when those are enabled for your app.

// A) Embedded (the default): an in-page iframe modal. First add your site's
//    origin to Allowed Origins in the dashboard (on a test key, localhost
//    works already).
const session = await onelo.loadAuthView()

// B) Popup: a top-level window, works on any origin — like an OAuth popup.
//    Nothing to configure.
//    const session = await onelo.loadAuthView({ mode: 'popup' })

// Read the current session + react to changes
const current = await onelo.auth.getSession()
const unsub = onelo.auth.onAuthStateChange((s) => {
  console.log(s ? 'signed in' : 'signed out', s?.user)
})

// ── Paywall hard-gate (apps that sell access) ───────────────
// loadAuthView() (above) auto-routes a signed-in user with NO plan to the store
// and resolves with a session ONLY once they're entitled — if they close the
// store without buying, it resolves null. So gate your app on the RESULT; never
// render paid UI just because sign-in happened. (Unlike Electron you don't manage
// a window — your framework re-renders on state — but this is the same entitlement
// the SDK enforces server-side on every call.)
const entitled = !onelo.auth.paywallEnabled || (await onelo.auth.hasActiveAccess())
if (session && entitled) {
  // → render your app
} else {
  // Not entitled: loadAuthView() already showed the store. Keep your locked
  // screen; call onelo.loadAuthView() again to re-open it.
}

// Sign out (also revokes the session server-side)
await onelo.auth.signOut()

// ── Using your own auth system? ─────────────────────────────
// Call identify() after your login so per-user/per-plan feature targeting applies.
await onelo.identify(currentUser.id)

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// When you publish a version with "Require in-app acceptance" ON in the
// dashboard (works for BOTH Terms of Service AND Privacy Policy), signed-in
// users get a BLOCKING full-screen gate they must accept — or Sign out — before
// continuing. Auto-presents on sign-in and the moment you publish (over SSE).
// "Notify only" versions never block. Nothing to wire — it's on by default.
//
// Return to sign-in on ANY sign-out (the gate's Sign out, a session revoke, or
// an expired session): re-render your app to the signed-out state here.
onelo.auth.onAuthStateChange((s) => { if (!s) { /* show your login screen */ } })
//
// Custom UI instead of the built-in gate (new Onelo({ autoPresentConsentGate: false })):
//   onelo.consent.onConsentRequired(() => { /* re-read + render your own banner */ })
//   const pending = await onelo.consent.requiredConsents()
//   await onelo.consent.acceptConsent(pending[0].versionId)
```
<!-- /onelo:snippet -->
