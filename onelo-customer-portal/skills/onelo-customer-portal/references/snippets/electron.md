# Electron

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=electron field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-electron#semver:*
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=customer-portal lang=electron field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
const { Onelo } = require('@onelo/electron')

const onelo = new Onelo({
  apiUrl: 'https://api.onelo.tools',
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  protocol: 'myapp',  // deep-link scheme registered in your app
  bundleId: 'com.company.app', // your app id — REQUIRED for codesign attestation (a signed build 403s without it)
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=electron field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```ts
// Open the hosted customer portal (manage subscription, cancel, receipts…).
// The user must be signed in. Resolves when they close the portal window.
// Optionally pass the parent BrowserWindow for modal centering: open(mainWindow)
try {
  await onelo.customerPortal.open()
} catch (err) {
  if (err.code === 'not_authenticated') {
    // User signed out — route them to sign-in
  } else {
    console.error('Portal error:', err)
  }
}

// Alternative: open in the user's real system browser (real domain + their
// saved cards / password manager). Then wire the deep-link back so hard
// account events (deletion / revoke) clear the local session instantly:
//   await onelo.customerPortal.openInBrowser()
//   app.on('open-url', (_e, url) => onelo.customerPortal.handlePortalCallback(url))               // macOS
//   app.on('second-instance', (_e, argv) => onelo.customerPortal.handlePortalCallback(argv.at(-1)))  // win/linux
```
<!-- /onelo:snippet -->
