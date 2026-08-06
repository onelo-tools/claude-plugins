# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=electron field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=electron field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/electron'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // your app id — REQUIRED for codesign attestation (a signed build 403s without it)
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// Anonymous — best for public-facing apps; the report isn't tied to a person
onelo.feedback.open()
onelo.feedback.open({ type: 'bug' })

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
onelo.feedback.open({ type: 'bug', area: 'checkout', userId: currentUser.id })
```
<!-- /onelo:snippet -->
