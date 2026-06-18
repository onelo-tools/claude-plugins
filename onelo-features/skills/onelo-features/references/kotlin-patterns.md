# Kotlin / Android Patterns

## Contents
- Kotlin-specific classification rules
- Grep patterns
- Symbol extraction
- Insertion point
- Snippets

## Kotlin-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Android adds two signals on top:

- **`*Activity` and `*Fragment` → screen.** Always. They *are* the unit
  the system navigates to.
- **`*ViewModel` → screen** (per Android architecture: one ViewModel per
  screen). Row-level VMs would be named `*RowViewModel` and drop via the
  generic `Row` suffix rule.
- **`*Dialog` is ambiguous** — full-screen dialogs are screens, confirmation
  pop-ups are atoms. Mark `ambiguous` in Phase 2.5 so the developer
  decides.

### Compose-specific note

For `@Composable fun XxxYyy(...)` matches, the `Yyy` suffix drives
classification via the generic Phase 2.5 list. Composables in a path
containing `/screens/`, `/pages/`, `/tabs/`, or `/flows/` are screens
even without a `*Screen` suffix.

## Grep patterns

```bash
# Jetpack Compose functions (match @Composable annotation + next function line)
grep -rn --include="*.kt" \
  -E "@Composable" \
  . 2>/dev/null
```

For each `@Composable` match at line N, read line N+1 to get the function name.
Line N+1 will be of the form `fun FunctionName(` — extract `FunctionName` as the symbol.

```bash
# Fragments
grep -rn --include="*.kt" \
  -E "^class ([A-Z][A-Za-z0-9]+Fragment)" \
  . 2>/dev/null

# Activities
grep -rn --include="*.kt" \
  -E "^class ([A-Z][A-Za-z0-9]+Activity)" \
  . 2>/dev/null

# ViewModels
grep -rn --include="*.kt" \
  -E "^class ([A-Z][A-Za-z0-9]+ViewModel)" \
  . 2>/dev/null
```

**Skip:** `*Test.kt`, `*Tests.kt`, `androidTest/`, `test/`

## Symbol extraction

- `fun ExportScreen(` (preceded by `@Composable`) → symbol: `ExportScreen`
- `class ExportFragment` → symbol: `ExportFragment`
- `class MainActivity` → symbol: `MainActivity`
- `class DashboardViewModel` → symbol: `DashboardViewModel`

## Insertion point

**Composable function:**
- Insert as first line of the function body `{`

**Fragment:**
- Find `override fun onViewCreated(`
- Insert as first line of the body

**Activity:**
- Find `override fun onCreate(`
- Insert after `super.onCreate(savedInstanceState)`

**ViewModel:**
- Find first `fun ` that isn't `init`
- Insert as first line

## Snippets

> **API note:** Onelo's Android SDK exposes `isVisible()`, `isEnabled()`,
> `isGreyed()` as **functions** (Kotlin convention for boolean checks). Call
> with parentheses.
>
> The snippets assume an `onelo` instance is reachable in scope. Typical
> projects expose it via a singleton or DI graph; see Phase 6 of SKILL.md
> for the init pattern.

### Choosing the right check

Default to **`isEnabled()`** for "show this content or don't" gates. It
matches the dashboard's primary on/off semantics and the Kotlin snippet
shown on `app.onelo.tools`, so docs and codegen stay aligned.

| Check | True when status is | Use for |
|---|---|---|
| `isEnabled()` | `ENABLED`, `NEW`, `BETA` | **Default.** Binary gate: show content vs. hide. Paid features, A/B tests, gradual rollouts. |
| `isVisible()` | anything except `HIDDEN` | When you render a greyed-out / upsell / "coming soon" variant in place of the real content. The status itself is the signal for *which* placeholder to show. |
| `isGreyed()`, `isUpsell()`, `isNew()`, `isBeta()`, `isComingSoon()` | the matching status | Branching on a specific promotional state. |

**Composable:**
```kotlin
val feat = onelo.features.feature("$NAME")
if (!feat.isEnabled()) return
```

**Fragment / Activity:**
```kotlin
val feat = onelo.features.feature("$NAME")
if (!feat.isEnabled()) {
    // On API 33+, prefer the dispatcher; the deprecated onBackPressed() still works
    // back to API 13 if you need it.
    requireActivity().onBackPressedDispatcher.onBackPressed()
    return
}
```

**ViewModel:**
```kotlin
val feat = onelo.features.feature("$NAME")
if (!feat.isEnabled()) return
```

### Dev-mode default

A freshly instrumented `feature("$NAME")` returns `HIDDEN` by default
until the gate is enabled in the dashboard — fail-closed in production.
During development you can flip the default so new gates render until
you toggle them in the dashboard:

```kotlin
val onelo = Onelo(
    publishableKey = "pk_live_...",
    baseURL = "https://app.onelo.tools",
    featureDefaultStatus = if (BuildConfig.DEBUG) FeatureStatus.ENABLED else FeatureStatus.HIDDEN,
)
```

Available in `onelo-android` ≥ 3.19.0.

### Upfront declaration & registry codegen

Lazy discovery via `feature("...")` only registers a name once the code
path actually runs. Screens that aren't in the hot path (debug
fragments, settings deep links, edge-case activities, BuildConfig.DEBUG
overlays) won't show up in the dashboard until someone happens to
navigate to them.

Call `declare(...)` once at startup with every name you instrument so
the dashboard reflects the full registry from the first run:

```kotlin
import com.myapp.generated.FeatureRegistry

class MyApp : Application() {
    lateinit var onelo: Onelo
    override fun onCreate() {
        super.onCreate()
        onelo = Onelo(/* … */)
        onelo.features.declare(FeatureRegistry.ALL)
    }
}
```

Maintain `FeatureRegistry.ALL` with a build-time script that greps your
sources for `feature("...")` calls — zero manual sync, zero risk of the
list drifting from reality.

#### Generated file

Path: `app/src/main/kotlin/<package>/generated/FeatureRegistry.kt`.

```kotlin
// AUTO-GENERATED. Do not edit manually.
package com.myapp.generated

object FeatureRegistry {
    val ALL: List<String> = listOf(
        "analytics-dashboard",
        "export-screen",
        "settings-account",
        // ...
    )
}
```

#### Generator script

Drop into `scripts/generate-feature-registry.sh` and `chmod +x`. Takes
sources dir, package name, and output file (or env vars). Defaults
target a typical single-module Android Gradle project.

```bash
#!/bin/bash
set -euo pipefail

# Usage: generate-feature-registry.sh [<sources_dir>] [<package>] [<output_file>]
SOURCES_DIR="${1:-${SOURCES_DIR:-app/src/main/kotlin}}"
PACKAGE="${2:-${PACKAGE:-com.myapp.generated}}"
PACKAGE_PATH="${PACKAGE//./\/}"
OUT_FILE="${3:-${OUT_FILE:-$SOURCES_DIR/$PACKAGE_PATH/FeatureRegistry.kt}}"

mkdir -p "$(dirname "$OUT_FILE")"

NAMES=$(grep -rhE 'features\.feature\("[a-z][a-z0-9-]*"\)' "$SOURCES_DIR" \
  --include="*.kt" \
  --exclude-dir=generated \
  | grep -oE '"[a-z][a-z0-9-]*"' \
  | sort -u)

{
  echo "// AUTO-GENERATED. Do not edit manually."
  echo "package $PACKAGE"
  echo ""
  echo "object FeatureRegistry {"
  echo "    val ALL: List<String> = listOf("
  while IFS= read -r name; do
    [ -n "$name" ] && echo "        $name,"
  done <<< "$NAMES"
  echo "    )"
  echo "}"
} > "$OUT_FILE"

COUNT=$(echo "$NAMES" | grep -c '^"' || true)
echo "Generated $OUT_FILE with $COUNT features"
```

#### Build hook integration

**Gradle (Kotlin DSL — `app/build.gradle.kts`):** register the script
as an Exec task and wire it as a `preBuild` dependency so it runs on
every build, including IDE syncs.

```kotlin
val generateFeatureRegistry by tasks.registering(Exec::class) {
    workingDir = rootDir
    commandLine(
        "${rootDir}/scripts/generate-feature-registry.sh",
        "${projectDir}/src/main/kotlin",
        "com.myapp.generated"
    )
    // Re-run unconditionally — feature names land in arbitrary .kt files,
    // and Gradle's input-snapshot UP-TO-DATE check would miss new ones.
    outputs.upToDateWhen { false }
}

tasks.named("preBuild").configure { dependsOn(generateFeatureRegistry) }
```

**Groovy DSL (`app/build.gradle`):** the equivalent is a few lines longer
but mechanically identical — register an `Exec` task pointing at the
same script, then `preBuild.dependsOn`.

**Multi-module projects:** repeat the task in every module that calls
`feature(...)`, each writing into its own module's `generated` package
to avoid cross-module coupling. The script's `--exclude-dir=generated`
guard prevents output recursion.

**Pure-Kotlin alternative:** for teams that prefer no bash dependency,
the same logic fits in ~30 lines of Kotlin inside the `buildSrc/` task
class. The bash version stays portable across Linux / macOS CI runners
without JVM bootstrap.

### Trigger detection patterns

#### Grep patterns

```bash
# Activity launches
grep -rnE '(Intent\(.*::class\.java\)|startActivity|startActivityForResult)' \
  --include="*.kt"

# Compose Navigation
grep -rnE '(navController\.navigate|NavHost|composable\(route)' \
  --include="*.kt"

# Fragment transactions
grep -rnE '(supportFragmentManager|FragmentTransaction|\.replace\(R\.id\.)' \
  --include="*.kt"

# Deep links
grep -rnE '(Intent\.ACTION_VIEW|<intent-filter|onNewIntent)' \
  --include="*.kt"
```

#### Linking heuristic

For destination `SettingsActivity` (feature `settings-screen`):
- `Intent(this, SettingsActivity::class.java)` → exact match
- `navController.navigate("settings")` → substring match by route
- `R.id.action_to_settings` → substring match by Nav graph action id

#### Trigger snippet

```kotlin
val feat = onelo.features.feature("$NAME")
if (feat.isVisible) {
    Button(onClick = { /* existing nav */ }) { Text("Settings") }
}
```

For imperative handlers:
```kotlin
val feat = onelo.features.feature("$NAME")
if (!feat.isVisible) return
// existing startActivity / navigate call
```

#### Rich destination states (opt-in)

```kotlin
val feat = onelo.features.feature("$NAME")
when {
    feat.isGreyed -> UpsellScreen(feat)
    feat.isComingSoon -> ComingSoonScreen()
    feat.isEnabled -> SettingsScreen()
    else -> { /* nothing */ }
}
```

### DI: Hilt

For Hilt projects, expose `Onelo` from a `@Module` installed in
`SingletonComponent`, then `@Inject` the module you actually need
(`OneloFeatures`, `OneloAuth`, etc.) at the call site.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object OneloModule {

    @Singleton
    @Provides
    fun provideOnelo(@ApplicationContext context: Context): Onelo =
        Onelo(
            publishableKey = "pk_live_...",
            baseURL = "https://app.onelo.tools",
            featureDefaultStatus = if (BuildConfig.DEBUG) FeatureStatus.ENABLED else FeatureStatus.HIDDEN,
        ).also {
            it.features.declare(FeatureRegistry.all)
        }

    @Provides
    fun provideOneloFeatures(onelo: Onelo): OneloFeatures = onelo.features
}
```

Then in a Composable's enclosing screen-level ViewModel (Hilt doesn't
inject directly into Composables — you reach the SDK via the
ViewModel):

```kotlin
@HiltViewModel
class ExportViewModel @Inject constructor(
    private val features: OneloFeatures,
) : ViewModel() {
    val featureState = features.feature("export-screen")
}
```

If you absolutely need the SDK inside a Composable without a ViewModel
in between, use `hiltViewModel()` to obtain one or `LocalContext.current`
+ `EntryPoint` to reach into the Hilt graph.

### DI: Koin

For Koin projects, register Onelo as a `single` in your top-level
module and inject it via `by inject()` or `koinInject()`:

```kotlin
val appModule = module {
    single {
        Onelo(
            publishableKey = "pk_live_...",
            baseURL = "https://app.onelo.tools",
            featureDefaultStatus = if (BuildConfig.DEBUG) FeatureStatus.ENABLED else FeatureStatus.HIDDEN,
        ).also { it.features.declare(FeatureRegistry.all) }
    }
    single { get<Onelo>().features }
}
```

In your `Application.onCreate()`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}
```

Then from a Composable:

```kotlin
@Composable
fun ExportScreen() {
    val features: OneloFeatures = koinInject()
    val feat = features.feature("export-screen")
    if (!feat.isEnabled()) return
    // ...
}
```

Or from a Fragment / Activity:

```kotlin
class ExportFragment : Fragment() {
    private val features: OneloFeatures by inject()
    // ...
}
```
