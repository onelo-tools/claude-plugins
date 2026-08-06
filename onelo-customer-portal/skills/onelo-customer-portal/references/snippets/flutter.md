# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK → Customer Portal** tab and **/docs** render from.
Insert it as-is; never write an Onelo SDK call from memory and never adapt
another platform's snippet.

## install
<!-- onelo:snippet sdk=customer-portal lang=flutter field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
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
<!-- onelo:snippet sdk=customer-portal lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

final onelo = Onelo(
  apiUrl: 'https://api.onelo.tools',
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  callbackScheme: 'myapp',  // deep-link scheme registered in AndroidManifest / Info.plist
);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=customer-portal lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```dart
// Push OneloCustomerPortalView to present the hosted portal IN-APP (WebView).
// The widget fetches the portal URL, renders the page, and can be CLOSED at any
// time via the ✕ in the top-right (or the hosted "Done"). Only the PAYMENT step
// ("Change plan" → card) hands off to the system browser; the portal management
// screen itself stays in your app. If the user schedules account deletion, the SDK
// clears the session for you before dismissing.

// The portal needs a signed-in session — guard first, else route to sign-in.
if (onelo.auth.currentSession == null) {
  // e.g. show your sign-in screen
  return;
}

Navigator.push(
  context,
  MaterialPageRoute(
    builder: (_) => OneloCustomerPortalView(
      portal: onelo.customerPortal,
      onDismiss: () => Navigator.pop(context), // ✕ / Done / account-deletion
    ),
  ),
);
```
<!-- /onelo:snippet -->
