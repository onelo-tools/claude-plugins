# onelo-branding

Make [Onelo](https://onelo.tools)'s hosted pages look like your product.

```
/plugin install onelo-branding@onelo-tools
/onelo-branding
```

## One theme, seven surfaces

Sign-in, checkout, the store, the customer portal, the waitlist, the feedback
form and the roadmap all render from **one theme** on your app. Set the logo and
brand colour once and the set looks coherent — you are not styling each page.

There is no code. Everything is dashboard configuration, except Custom CSS.

## What the skill actually does for you

- **Proposes values instead of asking for hex codes.** Point it at your live
  site and it reads your colour and font and offers them.
- **Keeps things out of Custom CSS.** Anything a control expresses stays in a
  control — a field is maintained by Onelo, a rule is maintained by you, forever.
- **Writes Custom CSS properly** when it is genuinely needed: stable
  `data-onelo-*` hooks instead of generated class names, both light and dark
  grounds, minimum surface area.
- **Warns you about the Stripe bridge.** Generic `input` styles are forwarded
  into Stripe's card fields, because CSS cannot otherwise reach them — so a rule
  you wrote for the feedback form can restyle checkout. Surface-specific hooks
  are not bridged.
- **Reviews all seven surfaces**, in both themes. That is where branding work
  usually fails.

## Paid parts, stated plainly

**Custom CSS** and **removing "Powered by Onelo"** are plan-gated. The footer is
enforced server-side, so the skill will not offer you CSS that pretends to hide
it.

## Related plugins

- **onelo-quickstart** — the front door; it hands you here as the finishing step.
