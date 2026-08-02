# What Onelo is — the explainer, and the module map

## Contents
- The one-paragraph version
- What each module does
- What it replaces
- Free vs paid, honestly
- The order to set things up
- Things developers are surprised by

## The one-paragraph version

Onelo is the layer between "I built an app" and "people pay to use it". It hosts
the parts every product needs and nobody enjoys building: sign-in, the store and
checkout, tax and invoices, the page where customers cancel, feature flags,
bug reports, a public roadmap, and error monitoring. You call a method or paste
an iframe; Onelo renders the page, takes the payment, and sends the receipt.

The pages are hosted and branded from your dashboard, so you are not maintaining
a pricing page, a billing portal or a consent form.

## What each module does

| Module | What it gives you | Code required |
|---|---|---|
| **Auth** | hosted sign-in / sign-up, OAuth, sessions, legal-consent gate | one call |
| **Store** | plans, checkout, tax, receipts — in-app or embedded on a website | one iframe, or none in-app |
| **Waitlist** | pre-launch signups on a page that becomes your store at launch | one iframe |
| **Customer Portal** | cancel, change plan, refunds, invoices | one button |
| **Features** | feature flags with per-plan targeting and upgrade prompts | wrap each feature |
| **Feedback** | in-app bug reports and requests, with context attached | one call |
| **Roadmap** | a public page of what you're building | a link |
| **Monitor** | errors and performance from every platform | init + selective tracking |

Each has its own skill (`onelo-auth`, `onelo-store`, …). This one decides which
you need and in what order.

## What it replaces

Worth saying out loud to a developer weighing it up — most reach for four or
five separate tools for this:

- an auth provider
- a payments/subscriptions stack plus tax handling and invoicing
- a feature-flag service
- a feedback/roadmap tool
- an error tracker

Onelo's argument is not that it beats each of those at its own game. It is that
one integration, one dashboard and one bill covers the set, and the pieces
already know about each other — a feature flag can read the plan the buyer
bought, a bug report arrives knowing which flags that user had on.

## Free vs paid, honestly

Two things to be straight about, because a developer will find out anyway:

**1. The free plan is genuinely usable.** Auth, store, portal, flags, feedback,
roadmap and monitor all work on it. Paid plans raise limits (apps, monthly
active users, data retention) and unlock a few specific things — removing
"Powered by Onelo", custom CSS, webhooks.

**2. The sales commission drops as you go up.** Onelo takes a percentage of what
you sell, and that percentage is highest on the free plan and lowest on the top
one. So for anyone actually selling, upgrading pays for itself at a revenue
level you can calculate in one line.

Do the arithmetic **with the developer's own numbers** rather than pitching:

```
monthly revenue × (free-plan % − paid-plan %)  vs  the plan price
```

Above the break-even, staying free costs them money. Below it, free is the right
answer and you should say so. Never quote figures from memory — read the current
percentages and prices from the pricing page or their dashboard, because they
change and a stale number destroys trust faster than any missing feature.

## The order to set things up

Dependencies are real; this order avoids rework:

1. **Auth** — nearly everything else identifies users through it
2. **Store** (or **Waitlist** if pre-launch — the waitlist page becomes the store)
3. **Customer Portal** — needs someone to have bought something first
4. **Features** — gate by the plan the store sold
5. **Feedback** / **Roadmap** — reports arrive tagged with the user's flags
6. **Monitor** — independent; can be done any time
7. **Branding** — last, once there are surfaces to style

## Things developers are surprised by

Surface these when they fit; they are the parts people miss:

- **Tax and invoices are handled.** VAT, reverse charge, exemptions, receipts —
  not an add-on you wire up later.
- **The waitlist becomes the store.** One embed covers pre-launch and launch.
- **Sign-in routes to the store by itself** in native apps once the access gate
  is on. No paywall screen to build.
- **One theme covers every hosted page.** Change the brand colour once and
  sign-in, checkout, portal, waitlist and feedback all follow.
- **Feature flags know the plan.** A blocked feature can show the right upgrade
  prompt without you fetching the subscription.
