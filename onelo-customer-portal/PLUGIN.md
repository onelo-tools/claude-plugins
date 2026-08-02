# onelo-customer-portal

Wires [Onelo](https://onelo.tools)'s hosted Customer Portal into your app —
the page where a subscriber cancels, changes plan, requests a refund within your
window, updates their payment method and downloads invoices.

```
/plugin install onelo-customer-portal@onelo-tools
/onelo-customer-portal
```

Onelo hosts and brands the whole portal. You build one "Manage subscription"
entry point; the skill finds the right place for it and inserts the official
snippet for your platform.

## Platforms

Ships on all six client platforms — JavaScript / TypeScript, Electron, React
Native, Flutter, Android (Kotlin) and Swift (iOS + macOS). One file per platform;
the skill opens only the one you need.

There is no backend snippet — the portal is a user-facing surface only.

## Three things the skill makes sure you handle

- **It needs a signed-in user.** Opening it without a session throws, so the
  entry point sits behind sign-in and the error path routes back there.
- **It resolves on close, not on change.** Plan changes arrive over SSE; your
  feature gates update themselves.
- **It can end the session.** Account deletion or a revoking refund signs the
  user out — your app has to re-render to its signed-out state.

## Snippet source

Snippets are **baked into this plugin at publish time** from `@onelo/snippets`,
the same source the dashboard SDK tab and /docs render from. Nothing is fetched
at runtime, so what the skill inserts cannot drift from the dashboard.

## Related plugins

- **onelo-auth** — required; the portal is for signed-in users.
- **onelo-store** — sells the plans the portal then manages.
- **onelo-features** — gate features by plan; use `openUpgrade()` on a blocked
  feature instead of opening the portal directly.
