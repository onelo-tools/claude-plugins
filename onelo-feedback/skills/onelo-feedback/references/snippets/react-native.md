# React Native

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/react-native'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // app id / package name for X-Bundle-Id — iOS auto-derives, ANDROID needs it (else 403 under enforcement)
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { FeedbackModal } from '@onelo/react-native'

// REQUIRED: mount <FeedbackModal> ONCE near your app root. open() only flips
// internal state; without this component mounted (subscribed) NOTHING shows —
// same requirement as Swift's .feedbackSheet.
<FeedbackModal feedback={onelo.feedback} />

// Then trigger open() from anywhere (a button, menu, …):
// Anonymous — best for public-facing apps; the report isn't tied to a person
onelo.feedback.open()
onelo.feedback.open({ type: 'bug' })

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
onelo.feedback.open({ type: 'bug', area: 'checkout', userId: currentUser.id })
```
<!-- /onelo:snippet -->
