# The in-app store (native apps) — dashboard setup, no code

## Contents
- Why there is no snippet
- What to turn on
- What the user then experiences
- How it differs from the website embed
- What to tell the developer to verify

## Why there is no snippet

In a native app — Swift, Android, Flutter, React Native, Electron — you do
**not** embed an iframe and you do **not** call a "show the store" method. The
store is driven by **Onelo Auth**.

`loadAuthView()` (JS/Electron/RN) and `OneloAuthView` (Swift) are not a login
box, they are a router. The backend decides the single next step for this user:
already entitled, needs to sign in, or needs to buy. When the answer is "needs
to buy", the hosted flow chains sign-up straight into the store itself and only
returns a session once the user is actually entitled.

So the integration work is: **wire Onelo Auth** (the `onelo-store` skill's job
ends there — use the `onelo-auth` skill), then flip one dashboard toggle. There
is nothing extra to call.

## What to turn on

Both are required. One without the other produces a confusing half-state.

1. **Paywall → Store → "Require plan on sign-up"** — ON.
   This is the master access gate. It is what makes the native sign-up route
   through the store.
2. **At least one visible store option**, and a **connected Stripe account** —
   otherwise there is nothing to sell and checkout cannot complete.

## What the user then experiences

1. Opens the app → the Onelo sign-in view appears
2. Signs up
3. The same view continues into the store — they pick a plan
4. The Stripe card page hands off to the system browser
5. Back in the app, the call resolves with a session and the app renders

A user without an active plan never reaches the app. That is the point of the
gate.

## How it differs from the website embed

| | Website embed | In-app store |
|---|---|---|
| Surface | public hosted page `/customer/app/<slug>` in an iframe | the Auth view, in-app |
| Code | two iframe snippets | none — Auth drives it |
| Needs "Require plan on sign-up" | **no** (to display) | **yes** |
| Needs a store option | no (renders from active products) | yes |
| Gated by | your Onelo plan's Store capability + Waitlist mode | the access gate above |

**The trap:** with the gate OFF the website store still *renders*, but the buy
link `/buy/…` returns 404. That is not a bug — an app with the paywall off is
free, so there is nothing to buy. If a developer reports "the store shows but I
can't purchase", this is almost always why.

## What to tell the developer to verify

1. **Sign-in routes to the store** for a brand-new account with no plan.
2. **The call resolves `null`** when they close the store without buying — their
   gate must handle it and keep the locked screen. Rendering paid UI right after
   sign-in is a paywall anyone can walk past.
3. **After purchase**, the app renders and the plan shows in the dashboard.
4. **Waitlist mode OFF** — while it is on, both surfaces route to the waitlist
   instead of the store, by design.
