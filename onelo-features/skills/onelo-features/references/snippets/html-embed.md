# Web — HTML embed and JavaScript / TypeScript

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

> One snippet covers Web — HTML embed and Web — JavaScript / TypeScript — the SDK code is identical.

## install
<!-- onelo:snippet sdk=features lang=web field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=web field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
import { Onelo } from '@onelo/js'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  // Feature environment: 'test' | 'live' — the features discovery/targeting env,
  // INDEPENDENT of monitor's `environment` field and NOT decided by your key.
  // Discovery (features auto-registering in your dashboard Registry) works ONLY
  // in 'test'; on 'live' the registry is read-only. Browser has no env var, so
  // this config field is the switch. 'test' while developing; remove / 'live'
  // for production.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
})
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=web field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```html
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// REACTIVITY — feature states are available immediately (no init() needed) and
// update in real-time the moment an admin clicks Deploy. subscribe() fires on every
// change (a Deploy over SSE, a plan change, an identity swap) — re-read feature()
// and re-render (React: wrap in useSyncExternalStore; Vue: bump a ref). Returns an
// unsubscribe fn; call it on teardown.
const unsubscribe = onelo.features.subscribe(() => { /* re-read + re-render */ })
const feature = onelo.features.feature('advanced-export')

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell and coming-soon states. And a GATED tile is tappable-to-UPGRADE
// only when the backend says so: feature.upgradeCta (the dashboard "Tapping the
// feature opens the upgrade flow" toggle) + feature.requiredPlan — covers greyed,
// upsell AND coming_soon. onelo.openUpgrade() routes to the working web surface:
// subscriber → Customer Portal (Change plan); otherwise → Store. Signed-in user.
if (!feature.isVisible || feature.isDisabled) {
  // 'hidden' or 'disabled' → render NOTHING
} else {
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null
  if (feature.isEnabled) {
    // enabled / new / beta → usable; run the feature on click
  } else if (canUpgrade) {
    // greyed / upsell / coming_soon + upgrade CTA → on click, open the upgrade flow:
    //   await onelo.openUpgrade()   // subscriber → Customer Portal · else → Store
    // Render feature.badgeLabel: '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  } else {
    // gated but informational (no upgrade CTA) → render feature.badgeLabel, disable the click.
  }
}

// Escape hatch — clears the local cache. You should rarely need this;
// deploys reach the SDK automatically.
onelo.features.invalidateCache()

// State helpers on feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing.
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .isUpsell     → 'upsell'      — VISIBLE; blocked. Tappable → upgrade when upgradeCta.
// .upgradeCta   → the dashboard "tap opens the upgrade flow" toggle; gate onelo.openUpgrade on it.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', …) or null
// .requiredPlan / .requiredPlanLabel / .upgradeHint → plan-gate CTA data
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show + BLOCKED · hidden·disabled = don't render
```
<!-- /onelo:snippet -->
