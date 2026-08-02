# onelo-store

Adds the [Onelo](https://onelo.tools) Store to your product. Onelo hosts the
plans, checkout, tax and receipts — you never build a pricing page or touch a
payment.

```
/plugin install onelo-store@onelo-tools
/onelo-store
```

## Two surfaces, one skill

**Website embed** — a hosted store page you drop into a marketing site as an
iframe. Two variants: full-width (follows the browser width, for the
multi-column desktop store) and in-container (fills its slot). Both auto-resize
to their content.

**In-app store** — for Swift, Android, Flutter, React Native and Electron apps.
No iframe and no code: the store appears inside the Onelo sign-in view. Turn on
**Paywall → Store → "Require plan on sign-up"**, and Onelo Auth routes any user
without a plan through the store before they reach the app.

The skill asks which surface you want, then either inserts the snippet or walks
you through the dashboard setup.

## Snippet source

The iframe snippets are **baked into this plugin at publish time** from
`@onelo/snippets` — the same source the dashboard SDK tab and /docs render from,
with the production URL already substituted. Nothing is fetched at runtime, so
the code the skill inserts cannot drift from what you see in the dashboard.

## Pre-launch? Use the waitlist instead

If you have not launched yet, do **not** embed the store — embed the
**waitlist**. The waitlist URL and the store are the same address, so one embed
shows the signup form now and starts showing your store the moment you flip
Waitlist mode off (or hit the scheduled time). No second snippet, no redeploy.

The skill asks about this before anything else and hands you to
**onelo-waitlist** when that is the situation.

## Related plugins

- **onelo-waitlist** — the pre-launch version of this page; its embed becomes this store at launch.
- **onelo-auth** — required for the in-app store; optional for the website embed.
- **onelo-features** — gate features by the plan a user bought.
- **onelo-monitor** — error and performance tracking.
