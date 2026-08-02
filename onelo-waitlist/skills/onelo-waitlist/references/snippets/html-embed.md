# Web — HTML embed

Baked into this plugin at publish time from `@onelo/snippets` — the same source
the dashboard **SDK → Waitlist** tab and **/docs** render from. Insert it as-is.

Replace `YOUR_WAITLIST_SLUG` with the real slug (dashboard → SDK → Waitlist shows
the finished URL). It appears **three times** per snippet: the iframe `src`, the
iframe `id`, and the `getElementById` in the resize script — change all three or
the auto-resize silently stops working.

`?lang=en` selects the locale. Drop it or change it to match a translation you
configured in the dashboard.

## init

The hosted waitlist page. Shareable as a plain link, and also the `src` the
iframes below point at.

<!-- onelo:snippet sdk=waitlist lang=web field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
<!-- Hosted waitlist page — shareable as-is, or use as the iframe src -->
https://onelo.tools/waitlist/YOUR_WAITLIST_SLUG
```
<!-- /onelo:snippet -->

## usage

Two embeds — pick ONE. Both auto-resize to their content, and both handle the
signup form **and** the store, because the same URL renders either depending on
Waitlist mode (see `references/waitlist-modes.md`).

- **Full-width** — breaks out of the page column and follows the browser width.
  Use it if you want the multi-column desktop store on wide screens after launch.
- **In-container** — fills its slot; keeps everything inside your page's column.

<!-- onelo:snippet sdk=waitlist lang=web field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
<!-- A) In-container: fills the slot you put it in -->
<iframe
  id="onelo-embed-YOUR_WAITLIST_SLUG"
  src="https://onelo.tools/waitlist/YOUR_WAITLIST_SLUG?lang=en"
  style="width:100%;border:none;min-height:480px;"
  loading="lazy"
></iframe>
<script>
  // Auto-resize the iframe height to its content (no inner scrollbar). Handles
  // both the waitlist form and the store (the waitlist-mode toggle shows either).
  window.addEventListener('message', function (e) {
    if (e.origin !== "https://onelo.tools") return;
    var d = e.data || {};
    if (d.type === 'onelo:resize' && d.height) {
      var f = document.getElementById('onelo-embed-YOUR_WAITLIST_SLUG');
      if (f) f.style.height = d.height + 'px';
    }
  });
</script>

<!-- B) Full-width: follows the browser width -->
<iframe
  id="onelo-embed-YOUR_WAITLIST_SLUG"
  src="https://onelo.tools/waitlist/YOUR_WAITLIST_SLUG?lang=en"
  style="width:100vw;max-width:100vw;margin-left:calc(50% - 50vw);border:none;min-height:480px;"
  loading="lazy"
></iframe>
<script>
  // Auto-resize the iframe height to its content (no inner scrollbar). Handles
  // both the waitlist form and the store (the waitlist-mode toggle shows either).
  window.addEventListener('message', function (e) {
    if (e.origin !== "https://onelo.tools") return;
    var d = e.data || {};
    if (d.type === 'onelo:resize' && d.height) {
      var f = document.getElementById('onelo-embed-YOUR_WAITLIST_SLUG');
      if (f) f.style.height = d.height + 'px';
    }
  });
</script>
```
<!-- /onelo:snippet -->
