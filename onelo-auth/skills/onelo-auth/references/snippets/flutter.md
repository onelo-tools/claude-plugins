# Flutter — Dart

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=flutter field=install -->
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
<!-- onelo:snippet sdk=auth lang=flutter field=init -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
import 'package:onelo/onelo.dart';

// Email/password hosted sign-in works out of the box — the in-app WebView
// captures the callback. You ONLY need to register the callback scheme for
// SOCIAL sign-in (signInWithOAuth — paid plans), which opens a system browser:
//   iOS → Info.plist: add "myapp" under CFBundleURLTypes → CFBundleURLSchemes
//   Android → AndroidManifest.xml, register flutter_web_auth_2's CallbackActivity
//     (a plain MainActivity intent-filter is NOT enough — OAuth will hang):
//     <activity android:name="com.linusu.flutter_web_auth_2.CallbackActivity"
//               android:exported="true">
//       <intent-filter>
//         <action android:name="android.intent.action.VIEW"/>
//         <category android:name="android.intent.category.DEFAULT"/>
//         <category android:name="android.intent.category.BROWSABLE"/>
//         <data android:scheme="myapp"/>
//       </intent-filter>
//     </activity>

// Initialize once (e.g. in main.dart)
final onelo = Onelo(
  publishableKey: 'onelo_pk_live_YOUR_KEY',
  apiUrl: 'https://api.onelo.tools',
  callbackScheme: 'myapp',
);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=flutter field=usage -->
<!-- baked from @onelo/snippets@0.19.10 — do not edit by hand -->
```dart
// Wrap your app — shows hosted sign-in automatically when not signed in
void main() {
  runApp(
    MaterialApp(
      home: OneloAuthView(
        auth: onelo.auth,
        child: MyHomePage(),
      ),
    ),
  );
}

// Access current user
final user = onelo.auth.currentSession?.user;

// Sign out (revokes server-side too, not just locally)
await onelo.auth.signOut();

// ── Paid access (entitlement) ───────────────────────────────
// hasActiveAccess reflects the user's paid entitlement (false when signed out).
if (onelo.auth.hasActiveAccess) { /* unlock paid features */ }
// Re-check after a purchase / cancellation:
await onelo.auth.revalidateEntitlement();

// ── Automatic remote logout ─────────────────────────────────
// A server-side revoke (ban, account deletion, refund lapse) clears the session
// in REAL TIME (SSE, with a 13-min heartbeat fallback) — zero code. onelo.auth is
// a ChangeNotifier: route to sign-in when currentSession becomes null;
// onelo.auth.isUserRevoked is true after a remote revoke (show a toast).

// If using Onelo Auth — features identify automatically after sign-in.
// If using your own auth system — call identify() manually after login:
await onelo.identify(currentUser.id);

// ── Legal-consent gate (Terms / Privacy updates) ────────────
// Nest OneloConsentGate INSIDE OneloAuthView.child, so a declined consent →
// sign-out falls back to the sign-in screen. It hard-blocks the app whenever a
// BLOCKING legal version is pending (publish a version with "Require in-app
// acceptance" — typically Terms; Privacy only if you turn it on). The user must
// Accept (on the hosted page) or Sign out — no dismiss. A live publish
// auto-presents in REAL TIME via SSE (no polling, no manual trigger), and only
// ONE gate ever shows even if several are mounted.
// MaterialApp(
//   home: OneloAuthView(
//     auth: onelo.auth,
//     child: OneloConsentGate(
//       consent: onelo.consent,
//       child: HomeScreen(),
//     ),
//   ),
// );
//
// Advanced / build your own UI (optional) — onelo.consent is a ChangeNotifier:
//   final pending = await onelo.consent.requiredConsents(); // [] if none due
//   if (onelo.consent.hasBlockingConsent) { /* render your own blocking screen */ }
//   await onelo.consent.acceptConsent(onelo.consent.pendingBlockingConsent!.versionId);
```
<!-- /onelo:snippet -->
