# React Native

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=customer-portal lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/react-native'

const onelo = new Onelo({
  apiUrl: 'https://api.onelo.tools',
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  bundleId: 'com.company.app', // app id / package name for X-Bundle-Id — iOS auto-derives, ANDROID needs it (else 403 under enforcement)
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// 1. Mount <CustomerPortalModal> once in your root component (alongside
//    <AuthModal> if you use auth), so it can render over your UI:
import { CustomerPortalModal } from '@onelo/react-native'

function App() {
  return (
    <>
      <YourNavigator />
      <CustomerPortalModal onelo={onelo} />
    </>
  )
}

// 2. Open the portal from any screen — the modal appears fullscreen.
//    The user MUST be signed in: open() throws OneloError('not_authenticated')
//    otherwise, so guard it and route signed-out users to sign-in.
try {
  await onelo.customerPortal.open()
} catch (err) {
  if (err.code === 'not_authenticated') {
    // User signed out — route them to sign-in (e.g. re-open OneloAuthGate)
  } else {
    console.error('Portal error:', err)
  }
}
// Note: React Native has no system-browser variant — the hosted portal always
// renders in-app via <CustomerPortalModal> (a WebView). Hard account events
// (deletion / revoke) inside the portal clear the local session automatically.
```
<!-- /onelo:snippet -->
