# onelo-quickstart

**Start here.** Ask it "what is Onelo and how do I set it up" and it answers,
looks at your project, asks four questions, and tells you which parts you
actually need — in the order that avoids rework.

```
/plugin install onelo-quickstart@onelo-tools
/onelo-quickstart
```

It sets nothing up itself. It decides *what* and *in what order*, then hands
each piece to its own skill (`onelo-auth`, `onelo-store`, …).

## What it's careful about

- **Answers the question before asking any.** If you asked what Onelo is, you
  get five lines, not a questionnaire.
- **Recommends only what fits.** A weekend project doesn't need seven modules.
- **Says what's free** — most of Onelo is. And names the specific paid things
  rather than being vague: removing "Powered by Onelo", custom CSS, webhooks,
  and higher app / active-user / retention limits.
- **Never quotes prices from memory.** They change; it points at the live
  pricing instead.
- **Doesn't ask you to migrate.** Already using Clerk, Supabase or your own
  auth? You keep it — Onelo identifies users through `identify()`.

If you sell things, it will show you the commission arithmetic with your own
numbers — the rate drops as the plan goes up, so above a certain revenue the
paid plan pays for itself. Below it, it will tell you to stay free.

## Branding

Covered here as the finishing step, because there is no code involved: one theme
in the dashboard styles sign-in, checkout, store, portal, waitlist and feedback
at once. Includes how to write Custom CSS without creating something you have to
maintain forever.

## Related plugins

Every module skill in this marketplace — this one is the front door to them.
