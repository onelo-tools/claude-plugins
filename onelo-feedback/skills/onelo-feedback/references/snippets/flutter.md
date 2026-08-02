# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Feedback** tab and **/docs** render from. Insert it
as-is; never write an Onelo SDK call from memory and never adapt another
platform's snippet.

## install
<!-- onelo:snippet sdk=feedback lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
# pubspec.yaml:
dependencies:
  onelo:
    git:
      url: https://github.com/onelo-tools/onelo-flutter
      ref: main
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=feedback lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

final onelo = Onelo(
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp',  // required by Onelo(); used by auth deep-links
);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=feedback lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// Anonymous — best for public-facing apps; the report isn't tied to a person
onelo.feedback.open(context);
onelo.feedback.open(context, options: const FeedbackOptions(type: 'bug'));

// Identified — INSIDE your app, pass the signed-in user so you know who reported it
onelo.feedback.open(context, options: FeedbackOptions(type: 'bug', area: 'checkout', userId: currentUserId));
```
<!-- /onelo:snippet -->
