# React Native

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=features lang=reactnative field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
npm install github:onelo-tools/onelo-react-native react-native-keychain react-native-webview
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=features lang=reactnative field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
import { Onelo } from '@onelo/react-native'
import { useEffect, useState } from 'react'

const onelo = new Onelo({
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  bundleId: 'com.company.app', // app id / package name for X-Bundle-Id — iOS auto-derives, ANDROID needs it (else bundle_id_mismatch 403 under enforcement)
  // Feature environment: 'test' | 'live' — which features snapshot this app READS
  // (drafts/QA vs production) and whether DISCOVERY is active, INDEPENDENT of your
  // key. React Native has no process.env, so this config field is the ONLY switch
  // (there is no env-var fallback). Set 'test' while developing — your in-progress
  // features show and auto-register in the dashboard Registry; on 'live' (or unset)
  // the registry is read-only. Set the SAME value in your backend.
  featureEnvironment: 'test',
  // Status for a slug not yet in the snapshot. Fail-closed 'hidden' in prod; set
  // 'enabled' in dev to preview new gates before toggling them in the dashboard.
  // featureDefaultStatus: 'hidden',
  // Foreground force-refreshes the snapshot (on by default); false = poll + SSE only.
  // autoLifecycleRefresh: true,
})

// Read a feature's state and STAY IN SYNC. FeatureState is a synchronous snapshot,
// and this hook re-renders automatically the instant the SDK cache changes — an
// admin's Deploy (pushed over the shared SSE connection), a foreground/identity
// resync, or invalidateCache(). subscribe() is purely local (no extra network, DB,
// or SSE) and returns an unsubscribe fn, which useEffect runs on unmount.
export function useFeature(name: string) {
  const [state, setState] = useState(() => onelo.features.feature(name))

  useEffect(() => {
    setState(onelo.features.feature(name))
    return onelo.features.subscribe(() => setState(onelo.features.feature(name)))
  }, [name])

  return state
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=reactnative field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```ts
// If you use your own auth system, call identify() after login so per-user /
// per-plan targeting applies. (Skip if using Onelo Auth — automatic.)
await onelo.identify(currentUser.id)

// Feature states are available immediately — no init() needed. Updates push in
// real-time the moment an admin clicks Deploy in the dashboard; the SDK also
// resyncs on app-foreground and when the signed-in user's plan changes.

// ⚠️ Render by STATUS, not a bare isEnabled — 'if (isEnabled)' HIDES the greyed
// (locked), upsell (upgrade) and coming-soon states you may have configured.
function ExportButton() {
  const feature = useFeature('advanced-export')
  // 'hidden' or 'disabled' → render NOTHING.
  if (!feature.isVisible || feature.isDisabled) return null

  // A plan-gated tile (greyed / upsell / coming_soon) is tappable-to-UPGRADE ONLY
  // when the backend says so: upgradeCta is the dashboard "Tapping the feature opens
  // the upgrade flow" toggle, and requiredPlan is the plan to open. This ONE gate
  // covers all three gated states — do NOT special-case isGreyed()/isUpsell().
  const canUpgrade = feature.upgradeCta && feature.requiredPlan != null

  // enabled / new / beta → usable.
  if (feature.isEnabled) return <Button onPress={runExport}>Export</Button>

  // gated + upgrade CTA → tap opens the hosted upgrade flow (subscriber → "Change
  // plan" with the plan pinned; non-subscriber → store). badgeLabel is ready-made:
  // '🔒' (greyed) · 'Available in Pro' (upsell) · 'Coming Soon'.
  if (canUpgrade) {
    return (
      <Button onPress={() => onelo.openUpgrade(feature.requiredPlan!)}>
        Export {feature.badgeLabel}
      </Button>
    )
  }

  // gated but informational (no upgrade CTA) → visible, tap DISABLED.
  return <Button disabled>Export {feature.badgeLabel}</Button>
}

// Escape hatch — clears the local cache. You should rarely need this;
// deploys reach the SDK automatically.
onelo.features.invalidateCache()

// OPEN THE UPGRADE FLOW — onelo.openUpgrade(plan) opens the hosted upgrade in the
// system browser: an active subscriber gets "Change plan" (target pinned), a
// non-subscriber gets the store. The backend decides — you just pass the plan.
// The tappable-to-upgrade gate is feature.upgradeCta && feature.requiredPlan != null
// (covers greyed, upsell AND coming_soon). Never gate on isGreyed()/isUpsell() alone.
//
// State helpers on onelo.features.feature(name):
// .isEnabled   → enabled | new | beta — the feature is USABLE (the ONE interactivity gate).
// .isVisible   → false ONLY for 'hidden' → when false, render nothing. ALSO hide 'disabled' via .isDisabled.
// .isDisabled  → 'disabled' — a killed feature; render nothing (isVisible is still true for it).
// .isNew / .isBeta → USABLE; cosmetic badge only (feature works).
// .isGreyed     → 'greyed'      — VISIBLE with 🔒; blocked. Tappable → upgrade when upgradeCta.
// .isComingSoon → 'coming_soon' — VISIBLE; blocked. Tappable → upgrade when upgradeCta (a plan-gated
//                 coming_soon CAN carry an upgrade CTA — don't assume it's inert).
// .isUpsell     → 'upsell'      — VISIBLE with "Available in <plan>"; blocked. Tap → onelo.openUpgrade.
// .upgradeCta   → dashboard "tap opens upgrade" toggle. PAIR WITH .requiredPlan → the tappable gate.
// .badgeLabel  → ready-made label ('New', '🔒', 'Available in Pro', 'Coming Soon', …) or null.
// .requiredPlan / .requiredPlanLabel → plan that unlocks it (slug / human label).
// .upgradeHint → { requiredPlan, currentStatus } when plan-gated — for upgrade-prompt UI.
// .status      → 'enabled' | 'new' | 'beta' | 'greyed' | 'upsell' | 'coming_soon' | 'hidden' | 'disabled'
//
// RENDER?/INTERACTIVE?  enabled·new·beta = show + interactive · greyed·coming_soon·upsell = show +
//   (tap→upgrade when upgradeCta, else blocked) · hidden·disabled = don't render
```
<!-- /onelo:snippet -->
