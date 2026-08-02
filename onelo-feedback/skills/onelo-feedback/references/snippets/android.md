# Android — Kotlin

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=android field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// build.gradle.kts (app/module)
// mavenCentral() is standard in Android projects — add it to settings.gradle.kts
// dependencyResolutionManagement { repositories { mavenCentral() } } if missing.
implementation("tools.onelo:onelo-android:1.+")
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=android field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
import com.onelo.android.Onelo
import com.onelo.android.OneloConfig
import com.onelo.android.FeedbackOptions

class MainActivity : AppCompatActivity() {
  private lateinit var onelo: Onelo

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    onelo = Onelo(
      config = OneloConfig(
        apiUrl = "https://api.onelo.tools",
        publishableKey = "onelo_pk_live_YOUR_KEY",
      ),
      context = applicationContext,
    )
  }
}
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=android field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```kotlin
// Anonymous — best for public-facing apps; the report isn't tied to a person
onelo.feedback.open(this)
onelo.feedback.open(this, FeedbackOptions(type = "bug"))

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
onelo.feedback.open(this, FeedbackOptions(type = "bug", area = "checkout", userId = currentUser.id))
```
<!-- /onelo:snippet -->
