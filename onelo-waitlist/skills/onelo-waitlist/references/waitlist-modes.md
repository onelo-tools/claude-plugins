# Waitlist mode, the launch timer, and turning into the store

## Contents
- The one idea that matters
- The three dashboard controls
- What renders, in every combination
- Choosing between the two launch behaviours
- What this means for the code you write

## The one idea that matters

**The waitlist URL and the store are the same address.** `/waitlist/<slug>`
decides at request time whether to show the signup form or send the visitor to
the store.

So you embed the waitlist iframe **once**, before launch, and at launch the same
embed starts showing your store — with no code change, no redeploy, and no
second snippet. That is the whole point of using the waitlist embed instead of
the store embed on a pre-launch page.

## The three dashboard controls

All in **Waitlist → Settings**.

| Control | What it does | Default |
|---|---|---|
| **Waitlist mode** | ON → the hosted store requires an invite token to reach the plans, and the waitlist page shows the signup form. | — |
| **Auto-redirect to store** | Governs what happens when Waitlist mode is OFF. ON → the waitlist page redirects to the store. OFF → it stays an independent page and shows a branded "closed" state. | **ON** |
| **Switch to store on** | Optional timestamp. The waitlist form auto-switches to the store at that moment. Blank = you switch manually. Only offered while Waitlist mode is ON *and* Auto-redirect is ON. | blank |

"Auto-redirect to store" sits **above** Waitlist mode in the UI and stays
editable in both states, because it describes the *after* — not the *now*.

## What renders, in every combination

| Waitlist mode | Timer passed? | Auto-redirect | `/waitlist/<slug>` shows |
|---|---|---|---|
| ON | no timer, or not yet | — | the **signup form** |
| ON | yes, timer elapsed | ON | the **store** |
| ON | yes, timer elapsed | OFF | branded **"closed"** state |
| OFF | — | ON | the **store** |
| OFF | — | OFF | branded **"closed"** state |

The timer is evaluated per request against the current time: no timestamp means
the waitlist stays active indefinitely, and an unparseable one is treated the
same way (fail-open to "still collecting signups") rather than closing early.

## Choosing between the two launch behaviours

**Auto-redirect ON (the default)** — you want the pre-launch page to become the
sales page. One embed on your marketing site covers the whole lifecycle. This is
what most products want.

**Auto-redirect OFF** — you want the waitlist to be its own thing: a permanent
page that closes gracefully, while the store lives at its own URL. Pick this when
the waitlist is for a *different* thing than the store sells (a beta of one
feature, a regional launch), so hijacking it to the store would confuse people.

## What this means for the code you write

- **Do not detect launch in your code.** No feature flag, no date check, no
  swapping the iframe `src`. The hosted page already decides.
- **Do not embed the store separately on the same page** "for after launch" —
  you would end up with two stores once the waitlist flips.
- The resize script in the snippet handles **both** renderings, so a page that
  worked for the form keeps working for the store.
- In native apps there is no embed at all: `loadAuthView()` routes users to the
  waitlist while Waitlist mode is on, and to the store afterwards. Same idea,
  same zero code changes.
