# Custom CSS — writing it without creating a maintenance problem

## Contents
- Before you write anything
- What to target
- The Stripe bridge — the trap nobody expects
- Both themes, always
- Keep it small
- Review checklist

## Before you write anything

Custom CSS is **plan-gated**. It is also the only part of branding where an
agent writes code rather than pointing at a toggle — so the temptation to reach
for it is real. Resist it.

Ask one question first: **can a dashboard control already do this?** Colours,
radii, border widths, fonts, sizes and shadows all have fields. A value set in a
field is maintained by Onelo; a rule you write is maintained by the developer,
forever, against a page they do not control.

Only write CSS for a **named, concrete** requirement the controls cannot express.
"Make it look nicer" is not one. "Our buttons are pill-shaped with a 2px inset
border" is.

## What to target

The hosted pages expose deliberate, stable hooks. Use these — never a hashed or
generated class name, which will change at the next release and take your rule
with it, silently.

**Structural (present on every surface):**

| Hook | What it is |
|---|---|
| `[data-onelo-root]` | the page root — scope everything under it |
| `.onelo-page` | page shell |
| `.onelo-card` / `[data-onelo-card-divider]` | the card and its divider |
| `.onelo-button` | buttons |
| `.onelo-input` | text inputs |
| `.onelo-footer` / `[data-onelo-powered-by]` | the footer (see the gate below) |

**Surface-scoped (use these when you mean ONE surface):**

`.onelo-auth` · `.onelo-portal` · `.onelo-features` · `.onelo-roadmap` ·
`[data-onelo-product-card]` · `[data-onelo-store-name]` ·
`[data-onelo-store-price]` · `[data-onelo-store-feature]` ·
`[data-onelo-store-badge]` · `[data-onelo-feedback-input]` ·
`[data-onelo-feedback-chip]` · `[data-onelo-feedback-submit]` ·
`[data-onelo-roadmap-item]`

If you cannot find a hook for what the developer wants, say so. Inventing a
selector against inspected markup produces a rule that breaks without warning.

## The Stripe bridge — the trap nobody expects

Stripe renders the card fields inside its own iframe, so CSS cannot reach them.
Onelo works around this: it **scans your Custom CSS for generic input selectors
and forwards those styles to Stripe**.

Bridged: `input`, `textarea`, `.onelo-input`.
Not bridged (deliberately): surface-specific hooks like
`[data-onelo-feedback-input]`.

Two consequences, and both bite:

**1. A generic rule you meant for one surface will restyle the card fields.**

```css
/* WRONG — this also lands on the Stripe card inputs */
input { border: 2px solid orange; }

/* RIGHT — stays on the feedback form */
[data-onelo-feedback-input] { border: 2px solid orange; }
```

**2. To style the card fields on purpose, use a generic selector** — that is the
only route in.

Rule of thumb: **generic selector = "every input everywhere, including
checkout". Surface hook = "just here".** Choose deliberately, every time.

## Both themes, always

The hosted pages honour a light and a dark ground. A hard-coded colour that only
works on one leaves half the users with unreadable text.

Prefer the theme's own custom properties over literal colours, and if you must
hard-code, check both. This is the single most common way custom CSS ships
broken — it looked right on the author's machine.

## Keep it small

Every rule is a bet that the page's structure stays the same. A long stylesheet
against a page you do not control is a standing liability at each Onelo release.

Write the minimum that expresses the requirement, and say out loud what the
maintenance cost is. A developer who understands that ten rules means ten things
to re-check usually asks for three.

## Review checklist

Before calling it done:

- [ ] every rule maps to a stated requirement — nothing "while we're here"
- [ ] no hashed or generated class names in any selector
- [ ] generic `input` / `textarea` rules are intentional (they reach Stripe)
- [ ] readable on both light and dark grounds
- [ ] checked on **every** surface, not just the one it was written for
- [ ] the footer was not the target (it cannot be hidden — see SKILL.md)
