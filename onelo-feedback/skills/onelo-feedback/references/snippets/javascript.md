# JavaScript / TypeScript

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=js field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=js field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=js field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// Anonymous — best for public-facing apps; the report isn't tied to a person
await onelo.feedback.open()
await onelo.feedback.open({ type: 'bug' })

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
await onelo.feedback.open({ type: 'bug', area: 'checkout', userId: currentUser.id })
```
<!-- /onelo:snippet -->
