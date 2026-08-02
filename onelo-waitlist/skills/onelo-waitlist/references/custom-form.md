# Building your own signup form

Use this only when the developer explicitly wants their own form. The hosted
embed is the default and it already handles validation, consent checkboxes,
localisation, branding, duplicate detection and the launch switch — all of which
you would otherwise rebuild.

> **This is an SDK method, not a dashboard snippet.** The dashboard ships no
> code sample for it, so there is nothing baked in here. The signature below is
> read from the `@onelo/js` source; if it ever disagrees with the installed SDK,
> the SDK wins.

## The call

```ts
const result = await onelo.waitlist.join(slug, email)
```

- `slug` — the waitlist's slug. Pass `undefined` to use the app's default
  waitlist.
- `email` — the address to enrol.

Returns:

```ts
{
  success: boolean       // false on any failure, including network errors
  position?: number      // their place in the queue, when the backend returns one
  alreadyJoined: boolean // this email was already on the list
}
```

## Behaviour you must handle

**It never throws.** A network failure resolves as `{ success: false,
alreadyJoined: false }` rather than rejecting. So an empty `catch` will not save
you — you have to check `success` and show a real message, otherwise a failed
signup looks identical to a successful one.

**`alreadyJoined` is not an error.** Show a friendly "you're already on the
list" state, not a failure. People re-submit constantly.

**`position` is optional.** Do not render "You're #undefined" — only show the
position when it is present.

## What you still owe the user

The hosted page gives these for free; a custom form does not:

- **Consent checkboxes** where you need them (GDPR). Collecting an email without
  the consent record your dashboard expects leaves you with a list you cannot
  legally use.
- **Email validation** beyond the browser's, and a disposable-domain policy if
  that matters to you.
- **The launch switch.** Your own form has no idea Waitlist mode turned off — it
  will happily keep collecting signups for a launched product unless you build
  that check yourself. The hosted page handles it by construction.

Say this out loud when a developer asks for a custom form. It is usually enough
for them to choose the embed.
