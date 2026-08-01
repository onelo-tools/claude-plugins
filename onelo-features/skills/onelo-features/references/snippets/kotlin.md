# Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## init
<!-- onelo:snippet sdk=features lang=kotlin field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts: implementation("tools.onelo:onelo-kotlin:1.+")  (from mavenCentral)

import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val config = OneloConfig(
    publishableKey = "onelo_pk_live_YOUR_KEY",
    apiUrl = "https://api.onelo.tools",
)
val onelo = Onelo(config, this)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=features lang=kotlin field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// onelo-kotlin is a BACKEND / JVM SDK — isEnabled() gates your server logic.
// It is true for ENABLED/NEW/BETA and false for GREYED/COMING_SOON/UPSELL/HIDDEN/
// DISABLED, so ONE check denies every non-usable state.
val f = onelo.features.feature("advanced-export")
if (!f.isEnabled()) {
    // not available for this user → deny / 403
    return
}
// serve the feature

// If you render UI from Kotlin (e.g. Compose Desktop), gate interactivity on
// isEnabled() too — NOT on isGreyed() alone (that leaves coming_soon clickable):
//   button.enabled = f.isEnabled() || f.isUpsell()   // greyed + coming_soon stay disabled

// Status → meaning (the rule that prevents the coming_soon trap):
// .isEnabled()    → ENABLED, NEW, BETA — USABLE (the gate).
// .isVisible()    → false ONLY for HIDDEN → don't surface the item at all.
// .isNew() / .isBeta()  → usable; cosmetic badge only.
// .isGreyed()     → GREYED      — NOT usable (show locked / deny).
// .isComingSoon() → COMING_SOON — NOT usable, EXACTLY like GREYED. NOT a badge.
// .isUpsell()     → UPSELL      — NOT usable; route to upgrade.
// .status         → ENABLED | NEW | BETA | COMING_SOON | GREYED | UPSELL | HIDDEN | DISABLED
```
<!-- /onelo:snippet -->
