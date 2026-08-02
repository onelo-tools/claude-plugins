# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=customer-portal lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig

val onelo = Onelo(
    config = OneloConfig(
        apiUrl = "https://api.onelo.tools",
        publishableKey = "onelo_pk_live_YOUR_KEY",
    ),
    context = applicationContext,
)
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// 1. Register the launcher in Activity / Fragment onCreate (before any UI):
val portalLauncher = onelo.customerPortal.registerLauncher(this, auth = onelo.auth) { result ->
    if (result.event != null) {
        // Hard account event (deletion / refund-revoke) — the SDK has ALREADY cleared
        // the local session; navigate the user to sign-in. result.event is the reason.
    }
    // result.dismissed == true when the user pressed Back without completing
}

// 2. Open the portal (from a coroutine — suspends during the network call):
lifecycleScope.launch {
    onelo.customerPortal.openCustomerPortal(
        auth = onelo.auth,
        launcher = portalLauncher,
    )
}

// 3. "Change plan" opens the Stripe card in the SYSTEM browser.
// The portal renders in-app; when the user picks a new plan and confirms, only the
// Stripe card opens in the system browser (real domain, saved cards, 3-D Secure).
// It deep-links back — the portal dismisses and the session refreshes. This uses the
// SAME handleRedirect wiring as the store (declare ONCE on your deep-link Activity,
// same <scheme>://callback intent-filter you already use for OAuth / store):
//   override fun onCreate(b: Bundle?) { super.onCreate(b); onelo.handleRedirect(intent?.data) }
//   override fun onNewIntent(i: Intent) { super.onNewIntent(i); onelo.handleRedirect(i.data) }
```
<!-- /onelo:snippet -->
