# onelo-roadmap

Show what you're building. [Onelo](https://onelo.tools) hosts a public roadmap
page; you link it.

```
/plugin install onelo-roadmap@onelo-tools
/onelo-roadmap
```

## No SDK code

The roadmap is a hosted public page at
`/roadmap/<teamSlug>/<appSlug>` — no snippet, no API call, no enablement toggle.
Anyone with the link can open it without signing in.

The skill hands you the URL, proposes where to put it (footer, help menu, end of
a release note, or an in-page iframe), and makes sure you know the one thing
developers always trip over:

**Only items you mark Public appear.** Everything else stays hidden. A roadmap
that looks empty is almost always a roadmap with nothing marked Public yet — not
a broken one.

## Related plugins

- **onelo-feedback** — where feature requests come from before you triage them
  onto the roadmap.
- **onelo-features** — ship a roadmap item behind a flag and roll it out
  gradually.
