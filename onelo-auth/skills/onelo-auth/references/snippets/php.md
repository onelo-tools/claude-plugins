# PHP — backend

Baked into this plugin at publish time from `@onelo/snippets` — the same
source the dashboard **SDK** tab and **/docs** render from. Insert it as-is;
never write an Onelo SDK call from memory and never adapt another platform's
snippet.

## install
<!-- onelo:snippet sdk=auth lang=php field=install -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
# onelo-php installs from GitHub via Composer (not on Packagist).
# Register the repository once, then require:
composer config repositories.onelo vcs https://github.com/onelo-tools/onelo-php
composer require onelo/onelo-php
```
<!-- /onelo:snippet -->

## init
<!-- onelo:snippet sdk=auth lang=php field=init -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
<?php
use Onelo\Onelo;

// Authenticate with your SERVER SECRET key (onelo_sk_live_…) — NOT a publishable
// key. Read it from the environment; never hard-code a secret in source.
$onelo = new Onelo(
    secretKey: $_ENV['ONELO_SECRET_KEY'],   // onelo_sk_live_…  (NEVER commit)
    apiUrl: 'https://api.onelo.tools',
);
```
<!-- /onelo:snippet -->

## usage
<!-- onelo:snippet sdk=auth lang=php field=usage -->
<!-- baked from @onelo/snippets@0.21.0 — do not edit by hand -->
```php
// ── Laravel ── bind $onelo in a service provider, then guard a route.
// The frontend sends the user's ACCESS TOKEN as: Authorization: Bearer <token>.
use Onelo\Laravel\OneloAuthenticate;

Route::get('/me', function (Illuminate\Http\Request $request) {
    $user = $request->attributes->get('oneloUser');   // Onelo\Auth\OneloUser
    return ['id' => $user->id, 'email' => $user->email];
})->middleware(OneloAuthenticate::class);

// Post-verify gates + SSE support (all optional):
//   new OneloAuthenticate($onelo, options: [
//     'acceptQueryToken'     => true,          // allow ?token= for EventSource/SSE
//     'identifyMonitor'      => false,         // opt out of auto monitor->setUser (on by default when the monitor scope is request-isolated)
//   ]);
//   OneloAuthenticate::optional($onelo)  // oneloUser = null instead of 401 (503 outage still surfaces)

// ── PSR-15 (Slim, Mezzio, Symfony) — one middleware, same $onelo client ──
//   use Onelo\Psr15\OneloAuthMiddleware;
//   $app->add(new OneloAuthMiddleware($onelo, $responseFactory));  // $responseFactory = your PSR-17 factory
//   // then in the handler: $request->getAttribute('oneloUser')

// ── Any other framework / a queue worker / no framework ──
//   The core is framework-agnostic — verify a token directly:
//   use Onelo\Auth\Verify;
//   $user = Verify::verifyToken($onelo, $token);   // → OneloUser, or a typed exception
//
// Error → status: OneloAuthInvalidToken / missing → 401 · OneloAuthForbidden /
//   failed gate → 403 · OneloAuthRateLimited (429) / OneloAuthUnavailable (5xx) → 503.
```
<!-- /onelo:snippet -->
