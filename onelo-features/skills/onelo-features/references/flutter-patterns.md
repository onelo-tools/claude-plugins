# Flutter / Dart Patterns

## Flutter-specific classification rules

Generic atom-suffix and atom-path filtering happens in SKILL.md Phase 2.5.
Flutter adds one strong heuristic on top:

- **`Scaffold` rule (override).** When a widget survives Phase 2.5 as
  `ambiguous`, peek at its `build()` method:
  - Returns / contains a `Scaffold(...)` → it's a `screen`. Keep.
  - Returns `Container` / `Padding` / `Row` / `Column` / `SizedBox` /
    `Stack` of primitives → `atom-likely`. Drop.

  This is the most reliable signal in Flutter because anyone painting a
  `Scaffold` is taking over the whole viewport.

### Extra Flutter-only atom suffixes to drop

On top of the generic Phase 2.5 list, Flutter idioms add:
- `Tile` — Material's `ListTile`/`SwitchTile`/`RadioListTile` convention.
  Always a list cell, never a screen.
- `Sliver` — sliver primitives that compose into a `CustomScrollView`.

For StatefulWidget pairs (`Foo` + `_FooState`), classify on the public
widget name (`Foo`); the state class inherits the same classification.

## Grep patterns

```bash
# StatelessWidget
grep -rn --include="*.dart" \
  -E "^class ([A-Z][A-Za-z0-9]+) extends StatelessWidget" \
  lib/ 2>/dev/null

# StatefulWidget
grep -rn --include="*.dart" \
  -E "^class ([A-Z][A-Za-z0-9]+) extends StatefulWidget" \
  lib/ 2>/dev/null

# State class (for StatefulWidget — find build method)
grep -rn --include="*.dart" \
  -E "^class _([A-Z][A-Za-z0-9]+)State extends State" \
  lib/ 2>/dev/null
```

**Skip:** `*_test.dart`, `test/`

## Symbol extraction

- `class ExportScreen extends StatelessWidget` → symbol: `ExportScreen`
- `class AnalyticsDashboard extends StatefulWidget` → symbol: `AnalyticsDashboard`

For StatefulWidget, instrument the `_XState` class's `build()` method, use the widget class name for the feature name.

## Insertion point

**StatelessWidget:**
- Find `Widget build(BuildContext context) {`
- Insert as first line of the build method body

**StatefulWidget (via State class):**
- Find `Widget build(BuildContext context) {` in the `_XState` class
- Insert as first line of the build method body

## Snippets

> **API note:** Onelo's Flutter SDK exposes `isVisible`, `isEnabled`, `isGreyed`
> as **Dart getters**. Access without parentheses (`feat.isEnabled`).
>
> The snippets assume an `onelo` instance is reachable in scope — typically
> a top-level `final onelo = Onelo(...)` in `main.dart`, or surfaced through
> Provider/Riverpod. See Phase 6 of SKILL.md for the init pattern.

### Choosing the right check

Default to **`isEnabled`** for "show this content or don't" gates. It
matches the dashboard's primary on/off semantics and the Flutter snippet
shown on `app.onelo.tools`, so docs and codegen stay aligned.

| Check | True when status is | Use for |
|---|---|---|
| `isEnabled` | `enabled`, `new`, `beta` | **Default.** Binary gate: show content vs. hide. Paid features, A/B tests, gradual rollouts. |
| `isVisible` | anything except `hidden` | When you render a greyed-out / upsell / "coming soon" variant in place of the real content. The status itself is the signal for *which* placeholder to show. |
| `isGreyed`, `isUpsell`, `isNew`, `isBeta`, `isComingSoon` | the matching status | Branching on a specific promotional state. |

**StatelessWidget / StatefulWidget:**
```dart
final feat = onelo.features.feature('$NAME');
if (!feat.isEnabled) return const SizedBox.shrink();
```

`SizedBox.shrink()` is the canonical "render nothing" widget. Inside a
`Column`/`Row` it collapses to zero size; if the surrounding layout
allocates space anyway (e.g. `mainAxisSize: MainAxisSize.max` with
spacing), wrap with `Visibility(visible: false, child: ...)` instead so
the slot is fully removed.

**With greyed/upsell variants:**
```dart
final feat = onelo.features.feature('$NAME');
if (!feat.isVisible) return const SizedBox.shrink();
// if (feat.isGreyed) return const _LockedVariant();
// if (feat.isUpsell) return const _UpgradePrompt();
```

### Dev-mode default

A freshly instrumented `feature('$NAME')` returns `hidden` by default
until the gate is enabled in the dashboard — fail-closed in production.
During development you can flip the default so new gates render until
you toggle them in the dashboard:

```dart
import 'package:flutter/foundation.dart';
import 'package:onelo/onelo.dart';

final onelo = Onelo(
  publishableKey: 'pk_live_...',
  baseUrl: 'https://app.onelo.tools',
  featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
);
```

Available in `onelo` (Dart) ≥ 3.19.0.

### Upfront declaration & registry codegen

Lazy discovery via `feature('...')` only registers a name once the code
path actually runs. Screens that aren't in the hot path (debug pages,
deep-linked settings, edge-case flows, kDebugMode-only widgets) won't
show up in the dashboard until someone happens to navigate to them.

Call `declare(...)` once at startup with every name you instrument so
the dashboard reflects the full registry from the first run:

```dart
import 'generated/feature_registry.dart';

void main() {
  onelo.features.declare(featureRegistry);
  runApp(MyApp());
}
```

Maintain `featureRegistry` with a build-time script that scans your
sources for `feature('...')` calls — zero manual sync, zero risk of the
list drifting from reality.

#### Generated file

Path: `lib/generated/feature_registry.dart`.

```dart
// AUTO-GENERATED. Do not edit manually.
// Regenerated on every build by tool/generate_feature_registry.dart

const featureRegistry = <String>[
  'analytics-dashboard',
  'export-screen',
  'settings-account',
  // ...
];
```

#### Generator script

Pure Dart — no bash dependency, runs on every platform Flutter
supports. Drop into `tool/generate_feature_registry.dart`.

```dart
// tool/generate_feature_registry.dart
import 'dart:io';

void main(List<String> args) {
  final sourcesDir = args.isNotEmpty ? args[0] : 'lib';
  final outFile = args.length > 1
      ? args[1]
      : '$sourcesDir/generated/feature_registry.dart';

  // Match feature('name') and feature("name"), atom syntax only.
  final pattern = RegExp(r'''features\.feature\(['"]([a-z][a-z0-9-]*)['"]''');
  final names = <String>{};

  final dir = Directory(sourcesDir);
  for (final entity in dir.listSync(recursive: true)) {
    if (entity is! File || !entity.path.endsWith('.dart')) continue;
    if (entity.path.contains('${Platform.pathSeparator}generated${Platform.pathSeparator}')) continue;

    for (final match in pattern.allMatches(entity.readAsStringSync())) {
      names.add(match.group(1)!);
    }
  }

  final sorted = names.toList()..sort();
  final out = File(outFile);
  out.parent.createSync(recursive: true);
  out.writeAsStringSync('''
// AUTO-GENERATED. Do not edit manually.
// Regenerated on every build by tool/generate_feature_registry.dart

const featureRegistry = <String>[
${sorted.map((n) => "  '$n',").join('\n')}
];
''');

  stdout.writeln('Generated $outFile with ${sorted.length} features');
}
```

Run manually with `dart run tool/generate_feature_registry.dart`.

#### Build hook integration

**Wrapper script (`tool/build.sh`):** simplest pattern — wrap the
flutter command in a script that regenerates first.

```bash
#!/bin/bash
set -euo pipefail
dart run tool/generate_feature_registry.dart
flutter build "$@"
```

Use `./tool/build.sh apk` (or `ios`, `web`) instead of bare
`flutter build`. CI configs invoke the wrapper the same way.

**`build_runner` integration (larger projects):** if you already use
`build_runner` for `freezed` / `json_serializable`, promote the
generator into a custom builder under `tool/builder/`. For most
projects the wrapper script is enough — adding `build_runner` just for
this would be over-engineering.

**`pubspec.yaml` aliases:** Dart doesn't have npm-style pre-build hooks,
but you can document the contract by adding the regen as a script
section in your README and refusing to merge PRs where
`feature_registry.dart` is out of sync (CI check: re-run the generator,
diff against committed file, fail if non-empty).

**Hot reload note:** the registry is read once at `main()` startup, so
calls added during a hot-reload session won't appear in the registry
until the next full restart + regen. This matches Flutter's general
hot-reload semantics for top-level constants and rarely matters in
practice (the lazy `feature()` call still registers the name on first
hit).

### Trigger detection patterns

#### Grep patterns

```bash
# Navigator (legacy)
grep -rnE '(Navigator\.(push|pushNamed|pushReplacement)|MaterialPageRoute)' \
  --include="*.dart"

# go_router
grep -rnE '(context\.(go|push|pushNamed|replace)|GoRoute)' \
  --include="*.dart"

# Deep links
grep -rnE '(uni_links|app_links|onGenerateRoute|Linking\.)' \
  --include="*.dart"
```

#### Linking heuristic

For destination `SettingsScreen` (feature `settings-screen`):
- `Navigator.push(MaterialPageRoute(builder: (_) => SettingsScreen()))` → symbol match
- `context.go('/settings')` → substring match by route
- `GoRoute(path: '/settings', builder: ...)` → match by path

#### Trigger snippet

```dart
final feat = onelo.features.feature('$NAME');
if (feat.isVisible)
  IconButton(icon: Icon(Icons.settings), onPressed: openSettings),
```

For imperative handlers:
```dart
final feat = onelo.features.feature('$NAME');
if (!feat.isVisible) return;
Navigator.push(context, MaterialPageRoute(builder: (_) => SettingsScreen()));
```

#### Rich destination states (opt-in)

```dart
final feat = onelo.features.feature('$NAME');
if (feat.isGreyed) return UpsellPage(feature: feat);
if (feat.isComingSoon) return ComingSoonPage();
if (!feat.isEnabled) return SizedBox.shrink();
return SettingsScreen();
```

### DI: Riverpod 3 (recommended default)

Riverpod is the default state management choice for new Flutter
projects in 2026. Expose Onelo as a `Provider` and read it via `ref`:

```dart
// providers/onelo.dart
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:onelo/onelo.dart';

final oneloProvider = Provider<Onelo>((ref) {
  final onelo = Onelo(
    publishableKey: 'pk_live_...',
    baseUrl: 'https://app.onelo.tools',
    featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
  );
  onelo.features.declare(FeatureRegistry.all);
  ref.onDispose(onelo.dispose);
  return onelo;
});

final featuresProvider = Provider<OneloFeatures>(
  (ref) => ref.watch(oneloProvider).features,
);
```

```dart
// main.dart
void main() => runApp(const ProviderScope(child: MyApp()));
```

Then in a `ConsumerWidget` (or `Consumer`/`HookConsumer`):

```dart
class ExportScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final feat = ref.watch(featuresProvider).feature('export-screen');
    if (!feat.isEnabled) return const SizedBox.shrink();
    // ...
  }
}
```

### DI: Provider (legacy / small projects)

For projects already using `package:provider`, expose Onelo via
`Provider.value` at the root and read it with `Provider.of<Onelo>` or
`context.read<Onelo>()`:

```dart
final onelo = Onelo(
  publishableKey: 'pk_live_...',
  baseUrl: 'https://app.onelo.tools',
  featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
);
onelo.features.declare(FeatureRegistry.all);

void main() {
  runApp(
    Provider<Onelo>.value(
      value: onelo,
      child: const MyApp(),
    ),
  );
}
```

```dart
// In a widget
@override
Widget build(BuildContext context) {
  final feat = context.read<Onelo>().features.feature('export-screen');
  if (!feat.isEnabled) return const SizedBox.shrink();
  // ...
}
```

### DI: BLoC / flutter_bloc

For BLoC projects, treat Onelo as a repository and expose it via
`RepositoryProvider`:

```dart
void main() {
  final onelo = Onelo(
    publishableKey: 'pk_live_...',
    baseUrl: 'https://app.onelo.tools',
    featureDefaultStatus: kDebugMode ? FeatureStatus.enabled : FeatureStatus.hidden,
  );
  onelo.features.declare(FeatureRegistry.all);

  runApp(
    RepositoryProvider<Onelo>.value(
      value: onelo,
      child: const MyApp(),
    ),
  );
}
```

```dart
// In a widget
@override
Widget build(BuildContext context) {
  final feat = context.read<Onelo>().features.feature('export-screen');
  if (!feat.isEnabled) return const SizedBox.shrink();
  // ...
}
```

For BLoCs that need feature-flag awareness (e.g. emit a different
state based on plan), pass `OneloFeatures` into the BLoC's constructor
and resolve `feature(...)` inside the relevant event handlers — keep
the side-effecting check out of the build method when possible.

### MaterialApp vs CupertinoApp

The instrumentation snippet (`SizedBox.shrink()` for hidden, optional
greyed/upsell branches) is style-agnostic and works under both
`MaterialApp` and `CupertinoApp`. If your app renders Cupertino-style
UI in some flows, make sure your "greyed" / "upsell" placeholders use
`Cupertino*` widgets too — `OneloFeatureBadge` (when added) follows
the ambient theme.
