# Middleware

Middleware is code that runs around the handling of a request. Each
middleware receives the request and a `$next` function; it can act before
calling `$next($request)`, after it (on the response), or answer by itself
without calling it at all.

```text
request ─► SecurityHeaders ─► Timing ─► route middleware ─► view
response ◄─────────────────────────────────────────────────┘
```

## Writing middleware

A middleware class implements `BrewPHP\Middleware\Middleware`:

```php title="apps/pages/Middleware/TimingMiddleware.php"
<?php

declare(strict_types=1);

namespace Apps\Pages\Middleware;

use BrewPHP\Http\Request;
use BrewPHP\Http\Response;
use BrewPHP\Middleware\Middleware;
use Closure;

final class TimingMiddleware implements Middleware
{
    public function process(Request $request, Closure $next): Response
    {
        $start = microtime(true);

        $response = $next($request);

        return $response->header('X-Response-Time', sprintf('%.1fms', (microtime(true) - $start) * 1000));
    }
}
```

Middleware classes are created by the container, so their constructor can
ask for services, such as the settings:

```php
use BrewPHP\Core\Settings;

public function __construct(private readonly Settings $settings)
{
}
```

### Stopping early

Return a response without calling `$next` to stop the request:

```php
<?php

declare(strict_types=1);

namespace Apps\Pages\Middleware;

use BrewPHP\Http\Request;
use BrewPHP\Http\Response;
use BrewPHP\Middleware\Middleware;
use Closure;

final class ApiKeyMiddleware implements Middleware
{
    public function process(Request $request, Closure $next): Response
    {
        if ($request->header('X-Api-Key') !== env('API_KEY')) {
            return json(['error' => 'Invalid API key'], 401);
        }

        return $next($request);
    }
}
```

### Closures

For something small, a closure with the same signature works anywhere a
middleware class does:

```php title="config/urls.php"
<?php

declare(strict_types=1);

use BrewPHP\Http\Request;
use BrewPHP\Http\Response;

$noCache = fn (Request $request, Closure $next): Response => $next($request)->header('Cache-Control', 'no-store');

return [
    path('account/', fn () => 'Private page', name: 'account', middleware: [$noCache]),
];
```

## Where middleware runs

### On every request: the MIDDLEWARE setting

```php title="config/settings.php"
use Apps\Pages\Middleware\TimingMiddleware;
use BrewPHP\Middleware\SecurityHeadersMiddleware;

'MIDDLEWARE' => [
    SecurityHeadersMiddleware::class,
    TimingMiddleware::class,
],
```

The first middleware in the list is the **outermost**: it runs first on the
way in and last on the way out.

### On some routes

Pass `middleware:` to `path()`, `group()`, or a `path()` that mounts an
`include_urls()`:

```php title="config/urls.php"
<?php

declare(strict_types=1);

use Apps\Pages\Middleware\ApiKeyMiddleware;
use Apps\Pages\Middleware\TimingMiddleware;

return [
    path('report/', fn () => 'Report', name: 'report', middleware: [TimingMiddleware::class]),

    group('api/', [
        path('status/', fn () => ['ok' => true], name: 'status'),
    ], middleware: [ApiKeyMiddleware::class], namespace: 'api'),
];
```

Route middleware runs **inside** the global middleware, after the route has
been matched. Group (outer) middleware runs before the route's own
middleware. `php manage.php routes` shows the middleware of each route.

## Errors and middleware

When a view throws an exception, it is turned into an error response
*inside* the global middleware stack. Global middleware therefore still sees
404 and 500 responses; for example, security headers are added to error
pages too. Exceptions thrown by middleware itself also become error pages.

## Security headers

`BrewPHP\Middleware\SecurityHeadersMiddleware` adds security-related headers
to every response. The skeleton enables it in `MIDDLEWARE`. Each header is
controlled by a setting, and a header the view already set is left alone:

| Setting | Header | Default |
|---|---|---|
| `SECURE_CONTENT_TYPE_NOSNIFF` | `X-Content-Type-Options: nosniff` | `true` |
| `SECURE_FRAME_OPTIONS` | `X-Frame-Options` | `'DENY'` |
| `SECURE_REFERRER_POLICY` | `Referrer-Policy` | `'same-origin'` |
| `SECURE_CROSS_ORIGIN_OPENER_POLICY` | `Cross-Origin-Opener-Policy` | `'same-origin'` |
| `SECURE_HSTS_SECONDS` | `Strict-Transport-Security: max-age=N` | `0` (off) |
| `SECURE_HSTS_INCLUDE_SUBDOMAINS` | adds `; includeSubDomains` to HSTS | `false` |
| `SECURE_HSTS_PRELOAD` | adds `; preload` to HSTS | `false` |
| `SECURE_CSP` | `Content-Security-Policy` | `null` (off) |

Set a string setting to `''` to stop sending that header.

```php title="config/settings.php"
'SECURE_HSTS_SECONDS' => 31536000,
'SECURE_CSP' => "default-src 'self'; img-src 'self' data:",
```

!!! warning "HSTS"
    The `Strict-Transport-Security` header is only sent on HTTPS requests.
    Once a browser has seen it, it refuses plain HTTP for your domain for
    `SECURE_HSTS_SECONDS` seconds. Enable it only when the whole site works
    over HTTPS, and start with a small value.

To allow a page to be shown in a frame, set the header in the view; the
middleware will not replace it:

```php
return response('Embeddable widget')->header('X-Frame-Options', 'SAMEORIGIN');
```

## Coming soon

Session, CSRF, authentication, CORS and maintenance-mode middleware arrive
with their features in later releases.
