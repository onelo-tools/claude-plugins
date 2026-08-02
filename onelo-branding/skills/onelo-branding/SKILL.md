---
name: onelo-branding
description: Brands the Onelo-hosted pages — sign-in, checkout, store, customer portal, waitlist, feedback and roadmap — from one theme in the dashboard, and writes Custom CSS for requirements the controls cannot express. Use when a developer wants Onelo's pages to match their product, asks about logos, colours, fonts or custom CSS, or wants to remove the "Powered by Onelo" footer.
allowed-tools: Bash Glob Grep Read Write Edit
---

# Onelo Branding

One theme, every hosted page. Set the logo and brand colour once and sign-in,
checkout, the store, the customer portal, the waitlist, the feedback form and
the roadmap all follow.

**There is no code to insert.** Everything below is dashboard configuration —
except Custom CSS, which is where you actually write something.

## Run checklist — copy into your FIRST reply and tick off IN ORDER

```
- [ ] 1 · Establish the target: what should these pages look like?
- [ ] 2 · Map it to controls — as much as possible WITHOUT custom CSS
- [ ] 3 · Custom CSS only for what's left, and only if plan allows
- [ ] 4 · Review across EVERY surface, both themes
```

---

## The model (say this first)

Three facts change how a developer approaches this:

**1. One theme covers everything.** The theme is per app and applies to all
surfaces. There is no per-page styling — a change to the button radius changes
it on checkout, the portal and the feedback form at once. That is a feature, not
a limitation: it is why the set looks coherent without effort.

**2. You cannot style these pages from your own CSS.** They are cross-origin.
The dashboard is the only route in, and Custom CSS is its escape hatch.

**3. The defaults are deliberate.** A developer who sets only the logo and brand
colour already gets a coherent result. Say this — it stops people from
disappearing into a settings panel for an hour.

---

## Phase 1 — Establish the target

Ask for something concrete. "Match our brand" is not actionable; a hex value, a
font name or a screenshot is.

Useful questions, in order:

1. **Brand colour** — the one they'd use for a primary button.
2. **Light or dark ground?** This drives page and card backgrounds.
3. **A logo** — and whether it needs a frame (a light logo on a light page
   usually does).
4. **A font** they already use, if it is in the list.
5. **Anything specific** they will notice if it's wrong — pill buttons, square
   corners, no shadows.

If they have a live site, look at it and propose values rather than asking for
five hex codes. Read the CSS, pull the primary colour and font, and offer them.

---

## Phase 2 — Map it to the controls

Work top-down; the first two groups do most of the work.

| Group | Controls |
|---|---|
| **Logo** | logo image, frame, frame fill, frame thickness |
| **Colors** | brand colour, page background, card background, text, borders |
| **Buttons** | background, text, border width and colour, radius |
| **Inputs** | background, text, border colour, field radius, border width |
| **Card** | border width, radius |
| **Typography** | heading and body font (curated list) and sizes |
| **Shadow** | per element — card, buttons, inputs — offset, blur, opacity |

Give the developer a values list to paste into the dashboard, not prose:

```
Colors    brand            #4F46E5
          page background  #0B0B0F
          card background  #15151B
Buttons   radius           999px      (pill, as requested)
Typography heading font    Plus Jakarta Sans
          body font        Inter
```

**Anything expressible here does NOT go in Custom CSS.** A value in a field is
maintained by Onelo; a rule is maintained by the developer forever.

---

## Phase 3 — Custom CSS (only for what's left)

`custom_css` is **plan-gated**. If the app is on a plan without it, say so plainly
and stop — do not offer workarounds.

Read [references/custom-css.md](references/custom-css.md) before writing a single
rule. It covers the stable hooks to target, both-theme handling, and one
behaviour that surprises everyone:

> Onelo forwards generic `input` / `textarea` / `.onelo-input` styles into
> Stripe's card fields, because CSS cannot otherwise reach them. So a generic
> input rule you wrote for the feedback form **will also restyle checkout**.
> Surface-specific hooks are deliberately not bridged.

Propose the CSS, explain what each rule is for, and wait for approval before it
goes in.

---

## Phase 4 — Review across every surface

This is where branding work actually fails: a rule tuned on checkout wrecks the
waitlist, and nobody looks until a customer does.

Check all seven, in both light and dark:

sign-in · store / checkout · customer portal · waitlist · feedback form ·
roadmap · the card

Preview pages exist for the waitlist, roadmap and paywall products — use them
rather than making real purchases.

Report what you checked. "Looks good" without naming the surfaces is not a
review.

---

## "Powered by Onelo"

Removing the footer is a **paid** feature (`remove_branding`). On the free plan
it stays, and it is enforced server-side — client-side attempts to hide it do
not work.

State it as a fact and move on. Do not propose CSS that tries to hide it: it
will not work, and suggesting it wastes the developer's time and your
credibility.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Checkout card fields don't match the other inputs | a surface-specific hook was used where a generic `input` rule was needed — only generic selectors bridge into Stripe. |
| Card fields changed colour unexpectedly | the opposite: a generic `input` rule intended for one surface bridged into Stripe. |
| A rule stopped working after an update | it targeted a hashed/generated class. Re-target a `data-onelo-*` hook or `.onelo-*` class. |
| Text unreadable for some users | a hard-coded colour that only works on one ground. Check both themes. |
| Custom CSS field is disabled | plan-gated — the app's plan doesn't include it. |
| Logo invisible on the page | light logo on a light background — enable the logo frame and set its fill. |
| One surface looks wrong, the rest fine | expected: a surface-scoped rule. Check whether it was meant to be global. |
