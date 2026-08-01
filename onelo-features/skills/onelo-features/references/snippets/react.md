# React

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=react field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-js
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=react field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/js'
import { createContext, useContext, useEffect, useState } from 'react'

// Create the Onelo instance once (outside the component tree)
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

export function useFeature(name: string) {
  const [state, setState] = useState(() => onelo.features.feature(name))

  useEffect(() => {
    setState(onelo.features.feature(name))
    // Live-update: subscribe() fires on every Deploy (SSE), plan change or identity
    // swap. Re-read feature() so the component re-renders. Returns the unsubscribe.
    return onelo.features.subscribe(() => setState(onelo.features.feature(name)))
  }, [name])

  return state
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=react field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call await onelo.identify(userId) after login
// so per-user/per-plan targeting can apply. (Skip if using Onelo Auth — automatic.)

// Usage in a component. Gate INTERACTIVITY on isEnabled; a blocked tile is
// tappable-to-UPGRADE only when feature.upgradeCta (dashboard "Tapping the feature
// opens the upgrade flow") + feature.requiredPlan — covers greyed, upsell AND
// coming_soon. onelo.openUpgrade() routes to the working web surface: subscriber →
// Customer Portal (Change plan); otherwise → Store. Needs a signed-in Onelo Auth user.
function ExportButton() {
  const feature = useFeature('advanced-export')
  if (!feature.isVisible || feature.isDisabled) return null   // hidden / disabled → nothing
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null
  return (
    <button
      disabled={!feature.isEnabled && !canUpgrade}            // blocked + no upgrade → inert
      onClick={feature.isEnabled ? runExport : () => onelo.openUpgrade()}
    >
      {feature.badgeLabel ? `Export • ${feature.badgeLabel}` : 'Export'}
    </button>
  )
}
```
<!-- /onelo:snippet -->
