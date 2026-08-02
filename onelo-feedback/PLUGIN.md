# onelo-feedback

In-app bug reports and feature requests, hosted by [Onelo](https://onelo.tools).

```
/plugin install onelo-feedback@onelo-tools
/onelo-feedback
```

You wire an entry point — "Report a bug", "Send feedback" — and one call opens
Onelo's hosted form in a modal. The report lands in your dashboard.

## What makes a report actually useful

The skill pushes three pieces of context, because a report without them is
usually unactionable:

- **`type`** — bug / feature request / general, pre-selected so the user doesn't
  classify their own report.
- **`area`** — where they were (`'checkout'`, `'settings'`). This is what makes
  reports triageable instead of "something is broken somewhere".
- **`userId`** — who reported it, when they're signed in. It travels as a header,
  never in the URL, so it never lands in an access log.

**The active features are attached automatically.** You don't pass them: the SDK
sends which flags that user had on. That is often the difference between "cannot
reproduce" and a one-minute fix.

## Platforms

JavaScript / TypeScript, Electron, React Native, Flutter, Android (Kotlin) and
Swift (iOS + macOS). One file per platform; the skill opens only the one it
needs. There is no backend snippet — feedback is user-facing only.

Snippets are baked in at publish time from `@onelo/snippets`, so what the skill
inserts matches the dashboard exactly.

## Related plugins

- **onelo-roadmap** — turn accepted requests into a public roadmap page.
- **onelo-monitor** — catches the errors users never report.
- **onelo-features** — the feature state that gets attached to each report.
