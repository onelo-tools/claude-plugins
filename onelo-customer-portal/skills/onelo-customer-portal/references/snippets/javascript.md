# JavaScript / TypeScript

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=js field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=customer-portal lang=js field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  apiUrl: 'https://api.onelo.tools',
  publishableKey: 'onelo_pk_live_YOUR_KEY',
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=js field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// Open the hosted customer portal (manage subscription, cancel, receipts…).
// The user must be signed in. Resolves when they close the portal.
await onelo.customerPortal.open()
```
<!-- /onelo:snippet -->
