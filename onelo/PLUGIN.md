# onelo

**Everything Onelo, in one install.**

```
/plugin marketplace add onelo-tools/claude-plugins
/plugin marketplace update onelo-tools      # already added it before? do this or the install 404s
/plugin install onelo@onelo-tools
/reload-plugins
```

`marketplace add` does nothing if you already have the marketplace, so an old
catalogue sticks around and the install fails with "not found". The refresh is
the fix.

That's it. This is a *bundle*: it ships no skills of its own and declares every
Onelo plugin as a dependency, so installing it pulls all of them in. Claude Code
lists what was added at the end of the install.

Then run `/onelo-quickstart` — it works out which parts your product actually
needs and walks you through them in order.

## What you get

| Plugin | What it does |
|---|---|
| onelo-quickstart | Start here — explains Onelo and picks what you need |
| onelo-auth | Hosted sign-in |
| onelo-store | Sell plans — website embed or in-app |
| onelo-waitlist | Pre-launch signups on a page that becomes your store |
| onelo-customer-portal | Cancel, change plan, refunds, invoices |
| onelo-features | Feature flags and per-plan gating |
| onelo-feedback | In-app bug reports and requests |
| onelo-roadmap | Public roadmap page |
| onelo-monitor | Errors and performance |
| onelo-branding | Brand every hosted page from one theme |

## Prefer to pick your own?

Install any of them individually — the bundle is a convenience, not a
requirement:

```
/plugin install onelo-auth@onelo-tools
```

## Updating

```
/plugin marketplace update onelo-tools
/reload-plugins
```

When a new Onelo plugin ships, it is added to this bundle's dependencies, so
updating the bundle brings it in.
