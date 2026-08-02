# onelo-waitlist

Collect signups before launch with [Onelo](https://onelo.tools) — then sell from
the same place, without touching the code again.

```
/plugin install onelo-waitlist@onelo-tools
/onelo-waitlist
```

## The point

The waitlist URL and your store are the **same address**. `/waitlist/<slug>`
decides at request time whether to show the signup form or the store.

So you embed the waitlist once on your pre-launch page, and at launch that same
embed starts selling — no code change, no redeploy, no second snippet. That is
why a pre-launch page should use the waitlist embed rather than the store one.

## Launch, three ways

In **Waitlist → Settings**:

- **Waitlist mode** — ON collects signups and requires an invite token to reach
  the plans.
- **Auto-redirect to store** (default ON) — what happens when Waitlist mode goes
  off: jump to the store, or stay an independent page with a branded "closed"
  state.
- **Switch to store on** — an optional timestamp; the page flips itself at that
  moment, so you can launch while nobody is watching.

The skill asks which behaviour you want, sets your expectations, and makes you
test **both** phases before calling it done.

## What it inserts

Two iframe variants (full-width, in-container), baked into this plugin at
publish time from `@onelo/snippets` — the same source the dashboard SDK tab and
/docs render from. Nothing is fetched at runtime.

A custom-form path exists for developers who insist, with an honest list of what
they give up: consent handling, localisation, duplicate detection and the launch
switch.

## Related plugins

- **onelo-store** — the plans the page switches to.
- **onelo-auth** — in native apps, sign-in does the waitlist routing for you.
