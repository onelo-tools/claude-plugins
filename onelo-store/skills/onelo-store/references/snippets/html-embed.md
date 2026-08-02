# Web — HTML embed

Baked into this plugin at publish time from `@onelo/snippets` — the same source
the dashboard **SDK → Store** tab and **/docs** render from. Insert it as-is.

Replace `YOUR_APP_SLUG` with the app's real slug (dashboard → SDK → Store shows
the finished URL). The slug appears in three places per snippet: the iframe
`src`, the `id`, and the `getElementById` in the resize script — change all
three or the auto-resize silently stops working.

## init

The hosted store page. Shareable as a plain link, and it is also the `src` the
iframes below point at.

<!-- onelo:snippet sdk=store lang=web field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
<!-- Hosted store page — shareable as-is, or use as the iframe src -->
https://onelo.tools/customer/app/YOUR_APP_SLUG
```
<!-- /onelo:snippet -->

## usage

Two embeds — pick ONE. Both auto-resize their height to the content, so the
iframe never gets its own scrollbar.

- **Full-width** — breaks out of the page column and follows the browser width.
  Use it for the multi-column desktop store on wide screens.
- **In-container** — fills whatever slot you put it in. Use it to keep the store
  inside your page's existing column.

<!-- onelo:snippet sdk=store lang=web field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
<!-- A) In-container: fills the slot you put it in -->
<iframe
  id="onelo-embed-store-YOUR_APP_SLUG"
  src="https://onelo.tools/customer/app/YOUR_APP_SLUG"
  style="width:100%;border:none;min-height:480px;"
  loading="lazy"
></iframe>
<script>
  // Auto-resize the iframe height to its content (no inner scrollbar).
  window.addEventListener('message', function (e) {
    if (e.origin !== "https://onelo.tools") return;
    var d = e.data || {};
    if (d.type === 'onelo:resize' && d.height) {
      var f = document.getElementById('onelo-embed-store-YOUR_APP_SLUG');
      if (f) f.style.height = d.height + 'px';
    }
  });
</script>

<!-- B) Full-width: follows the browser width -->
<iframe
  id="onelo-embed-store-YOUR_APP_SLUG"
  src="https://onelo.tools/customer/app/YOUR_APP_SLUG"
  style="width:100vw;max-width:100vw;margin-left:calc(50% - 50vw);border:none;min-height:480px;"
  loading="lazy"
></iframe>
<script>
  // Auto-resize the iframe height to its content (no inner scrollbar).
  window.addEventListener('message', function (e) {
    if (e.origin !== "https://onelo.tools") return;
    var d = e.data || {};
    if (d.type === 'onelo:resize' && d.height) {
      var f = document.getElementById('onelo-embed-store-YOUR_APP_SLUG');
      if (f) f.style.height = d.height + 'px';
    }
  });
</script>
```
<!-- /onelo:snippet -->
