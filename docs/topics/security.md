# Security

BrewPHP tries to make the safe way the easy way. This page gives an
overview of the protections it includes, how to use them, and what is
left to you.

## CSRF protection

Cross-site request forgery (CSRF) tricks a logged-in visitor's browser into
sending a request to your site from another site: a hidden form on
`evil.example` that posts to `/account/delete/`. The browser adds your
cookies, so without protection the request would look genuine.

`BrewPHP\Security\CsrfViewMiddleware` works like Django's. Add it to
`MIDDLEWARE` after `SessionMiddleware` (the skeleton does):

```php title="config/settings.php"
'MIDDLEWARE' => [
    SecurityHeadersMiddleware::class,
    SessionMiddleware::class,
    CsrfViewMiddleware::class,
    AuthenticationMiddleware::class,
],
```

Then put `{% csrf_token %}` in every form that posts to your site:

```html+django
<form method="post" action="{% url 'blog:new' %}">
    {% csrf_token %}
    {{ form }}
    <button type="submit">Publish</button>
</form>
```

It prints a hidden field:

```html
<input type="hidden" name="csrfmiddlewaretoken" value="Cf91sz7flU4aoOqEhDIMwi7ThNCBOyDt86XYLVWqOdD7x2Qm0UMMVxRKB8O4v55A">
```

The `csrf_token` variable is available in every template rendered with
`render()` during a request, whatever `CONTEXT_PROCESSORS` lists. In a PHP
template, write the field yourself:

```php
<input type="hidden" name="csrfmiddlewaretoken" value="<?= e($csrf_token) ?>">
```

Only forms that post to **your own** site get the token: never put it in a
form that posts to another site, where it would leak.

### How it works

- A random 32-character **secret** is kept in the `csrftoken` cookie (or in
  the session with `CSRF_USE_SESSIONS`). It is created the first time a
  page uses a token, so pages without forms set no cookie.
- Each page gets a **masked token**: the secret combined with a fresh random
  mask (64 characters). The token in the HTML changes on every response,
  which defeats the BREACH compression attack, while still proving
  knowledge of the secret.
- Every `POST`, `PUT`, `PATCH` and `DELETE` request is checked. `GET`,
  `HEAD`, `OPTIONS` and `TRACE` are not, so never change data in a `GET`
  view.
- The **Origin** header, when the browser sends one, must be this site or
  a trusted origin. Over HTTPS without an Origin header, the **Referer**
  must be an HTTPS page of this site or a trusted origin.
- The request must then send a token, in the `csrfmiddlewaretoken` field
  or the `X-CSRFToken` header, that matches the secret. Masked
  (64-character) and plain (32-character) tokens are accepted.
- At login the secret is replaced, so a token planted before login is
  useless afterwards.

A failed check is a `403`. With `DEBUG` on, the page says why:

```text
CSRF verification failed. CSRF token missing.
CSRF verification failed. CSRF token incorrect.
CSRF verification failed. CSRF cookie not set.
CSRF verification failed. Origin checking failed - https://evil.example does not match any trusted origins.
CSRF verification failed. Referer checking failed - no Referer.
```

With `DEBUG` off, it only says `CSRF verification failed. Request aborted.`
Requests for URLs that match no route get their usual `404` or `405`.

### Trusted origins

A form on another of your sites (say `https://shop.example.com` posting to
`https://example.com`) is refused unless its origin is listed:

```php title="config/settings.php"
'CSRF_TRUSTED_ORIGINS' => [
    'https://shop.example.com',
    'https://*.example.com',   // every subdomain, over HTTPS only
],
```

Entries include the scheme. `https://*.example.com` matches
`https://shop.example.com` but neither `http://shop.example.com` nor
`https://example.com.evil.net`.

### AJAX

JavaScript sends the token in the `X-CSRFToken` header. The `csrftoken`
cookie is readable by JavaScript (`CSRF_COOKIE_HTTPONLY` is `false`, as in
Django), and the plain secret in it is accepted as a token:

```javascript
function getCookie(name) {
    const match = document.cookie.match('(?:^|; )' + name + '=([^;]*)');
    return match ? decodeURIComponent(match[1]) : null;
}

fetch('/api/posts/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken'),
    },
    body: JSON.stringify({title: 'Hello'}),
});
```

The cookie only exists once a page has used a token. On a page without a
form, print `{{ csrf_token }}` somewhere JavaScript can read it (this also
works with `CSRF_USE_SESSIONS` and `CSRF_COOKIE_HTTPONLY`, when the cookie
is not readable):

```html+django
<meta name="csrf-token" content="{{ csrf_token }}">
```

```javascript
const token = document.querySelector('meta[name="csrf-token"]').content;
```

### Exempting a view

A webhook called by another server cannot send a token. Mark the view with
`#[CsrfExempt]`, and check the caller another way (most services sign
their requests):

```php
use BrewPHP\Http\Request;
use BrewPHP\Http\Response;
use BrewPHP\Security\CsrfExempt;

#[CsrfExempt]
function payment_webhook(Request $request): Response
{
    // Verify the provider's signature header first!
    return response('ok');
}
```

The attribute works on functions, closures, controller methods (or their
class) and `View` classes.

### Settings

| Setting | Default | Description |
|---|---|---|
| `CSRF_COOKIE_NAME` | `'csrftoken'` | Name of the cookie holding the secret. |
| `CSRF_COOKIE_AGE` | `31449600` (about a year) | Lifetime of the cookie, in seconds. |
| `CSRF_COOKIE_SECURE` | `false` | Only send the cookie over HTTPS. Set it to `true` in production. |
| `CSRF_COOKIE_HTTPONLY` | `false` | Hide the cookie from JavaScript (then read the token from the page for AJAX). |
| `CSRF_COOKIE_SAMESITE` | `'Lax'` | `'Lax'`, `'Strict'` or `'None'`. |
| `CSRF_COOKIE_DOMAIN` | `null` | Domain of the cookie, such as `'.example.com'` for forms on subdomains. |
| `CSRF_COOKIE_PATH` | `'/'` | Path of the cookie. |
| `CSRF_HEADER_NAME` | `'X-CSRFToken'` | Header that AJAX requests use for the token. |
| `CSRF_TRUSTED_ORIGINS` | `[]` | Other origins allowed to post, such as `'https://*.example.com'`. |
| `CSRF_USE_SESSIONS` | `false` | Keep the secret in the session instead of a cookie. Needs `SessionMiddleware` before `CsrfViewMiddleware`. |

## Cross-site scripting (XSS)

Cross-site scripting puts a visitor's input into a page as HTML, so a
comment containing `<script>` runs in other visitors' browsers.

- The template language **escapes every variable**: `{{ comment.text }}`
  prints `&lt;script&gt;`. See
  [Automatic HTML escaping](templates.md#automatic-html-escaping).
- In PHP templates and views, escape with `e()`.
- Forms, fields and error lists escape everything they print.
- `json_script` encodes `<`, `>` and `&` for data passed to JavaScript.
- `URLField` refuses `javascript:` links.

Only switch escaping off for HTML you produced yourself: `|safe`,
`{% autoescape off %}`, `mark_safe()` and `new SafeString($html)` print
their value as is. An object that builds its own escaped HTML can implement
the `BrewPHP\Templates\HtmlSafe` interface (as forms do), so templates and
`e()` print it without escaping it again.

## Clickjacking and security headers

`SecurityHeadersMiddleware` sends `X-Frame-Options: DENY` (your pages
cannot be shown in a frame on another site), `X-Content-Type-Options:
nosniff`, a `Referrer-Policy`, a `Cross-Origin-Opener-Policy`, and on
request HSTS and a Content Security Policy. See
[Security headers](middleware.md#security-headers).

## Sessions

Session ids are random, a session id chosen by the browser is never adopted
(no session fixation), the id changes at login, logout deletes the session,
and the cookie is `HttpOnly` and `SameSite=Lax`. Session data is JSON,
never unserialized. Set `SESSION_COOKIE_SECURE` on HTTPS sites. See
[Sessions](sessions.md#security).

## Passwords

Passwords are hashed with Argon2id or bcrypt-SHA256, compared in constant
time, and upgraded when the settings change. Logins are throttled, a
wrong username takes as long as a wrong password, and failed logins never
say whether the username exists. A password change logs out the user's
other sessions. Turn on the password validators. See
[Authentication](auth.md#password-hashing).

## The admin

The [admin](admin.md) only lets active staff in, checks the model
permissions on every page, limits every page and action to the
ModelAdmin's `getQueryset()`, ignores filters, sort columns and actions it
does not declare, posts every form (logout included) with a CSRF token,
and is never cached. Its stylesheet and script are files, not inline code,
so it works under a strict `SECURE_CSP`. See
[Admin security](admin.md#security).

## Open redirects

Redirecting to a URL taken from the request (`?next=/dashboard/`) is an
**open redirect** when any URL is accepted: a phishing link can send people
to your login page and then on to a look-alike site. Check such URLs with
`Redirects::isSafe()`:

```php
use BrewPHP\Security\Redirects;

$next = $request->query('next');

return redirect(Redirects::isSafe($next, $request) ? $next : '/');
```

A URL is safe when it is a path on this site (`/dashboard/`) or an `http`
or `https` URL whose host (with port) is the request's, or one of the hosts
in the third argument: `Redirects::isSafe($next, $request, ['shop.example.com'])`.
URLs such as `//evil.example`, `/\evil.example`, `javascript:...`, URLs
with control characters (such as CR or LF), surrounding spaces or user
names, and (on an HTTPS page) plain `http`
URLs are refused. The login and logout views already do this for `next`.

## Signing

`BrewPHP\Security\Signer` signs values with an HMAC-SHA256 of the
`SECRET_KEY`, like Django's `django.core.signing`. A signed value can be
read by anyone but not changed: use it for tokens in URLs or cookies
("unsubscribe", "confirm my email"), never to hide secrets.

```php
use BrewPHP\Security\BadSignature;
use BrewPHP\Security\SignatureExpired;
use BrewPHP\Security\Signer;

$signer = Signer::make('newsletter.unsubscribe');

$token = $signer->sign('42');                 // "42:" followed by the signature
$signer->unsign($token);                      // "42"

$token = $signer->signWithTime('42');         // with the current time
$signer->unsignWithTime($token, 86400);       // "42" if at most a day old

$token = $signer->dumps(['user' => 42, 'list' => 'news']);   // compact, URL-safe
$signer->loads($token);                       // ['user' => 42, 'list' => 'news']

try {
    $id = $signer->unsign((string) $request->query('token'));
} catch (SignatureExpired) {
    abort(400, 'This link has expired.');
} catch (BadSignature) {
    abort(400, 'This link is not valid.');
}
```

| Method | Purpose |
|---|---|
| `Signer::make($salt)` | A signer using `SECRET_KEY`. Without a key it throws `ImproperlyConfigured`. |
| `sign($value)` / `unsign($signed)` | Add a signature / check it and return the value. |
| `signWithTime($value)` / `unsignWithTime($signed, $maxAge)` | The same, with a timestamp; `SignatureExpired` when older than `$maxAge` seconds; a timestamp more than 60 seconds in the future is refused with `BadSignature`. |
| `dumps($data)` / `loads($signed)` | Sign JSON-encodable data as a URL-safe string, and read it back. |

The **salt** separates uses: a token signed with the salt
`'newsletter.unsubscribe'` is refused by a signer with another salt, so a
signature made for one feature cannot be replayed in another. Changing
`SECRET_KEY` makes every signature invalid. `SignatureExpired` extends
`BadSignature`, so catch it first.

## Rate limiting

`BrewPHP\Security\RateLimiter` counts attempts per key in a time window. The
login form uses it; use it to slow down anything that can be guessed or
abused:

```php
use BrewPHP\Security\RateLimiter;

$limiter = RateLimiter::make();
$key = 'coupon|' . $request->ip();

if ($limiter->tooManyAttempts($key, 10)) {
    abort(429, sprintf('Too many attempts. Try again in %d seconds.', $limiter->availableIn($key)));
}
if (!coupon_is_valid($code)) {
    $limiter->hit($key, 3600);   // count a failure; the window lasts an hour
}
```

| Method | Purpose |
|---|---|
| `RateLimiter::make()` | A limiter keeping its files in `storage/cache/ratelimit/`. |
| `hit($key, $decaySeconds)` | Count one attempt; returns the attempts in the window. The window starts with the first attempt. |
| `attempts($key)` | Attempts in the current window. |
| `tooManyAttempts($key, $max)` | Whether `attempts()` reached `$max`. |
| `availableIn($key)` | Seconds until the window ends. |
| `clear($key)` | Forget the key (after a success). |

Keys are hashed, so they can contain user input. The counters are local
files: on several web servers each one counts on its own.

## SQL injection

The ORM always sends values as query parameters, never inside the SQL
text: `Post::objects()->filter(title: $title)` is safe whatever `$title`
contains. When you write SQL yourself, use placeholders too:

```php
db()->select('SELECT * FROM blog_post WHERE views > ?', [$minViews]);
Post::objects()->raw('SELECT * FROM blog_post WHERE title = ?', [$title]);
```

Never build SQL by joining strings with user input. The `RunSQL` migration
operation runs its SQL as written; see
[Migrations](migrations.md#security).

## Host header validation

Every request's `Host` header is checked against `ALLOWED_HOSTS`, so a
forged host cannot end up in absolute URLs (password links, redirects).
Other hosts get a `400`. See
[Allowed hosts](../getting-started/configuration.md#allowed-hosts).

## Debug page

With `DEBUG` on, errors show the settings and the request. Values whose
names contain `SECRET`, `PASSWORD`, `PASS`, `KEY`, `TOKEN`, `SIGNATURE`,
`AUTH`, `COOKIE`, `CREDENTIAL`, `PRIVATE`, `SESSION` or `CSRF` are masked,
such as the `brewphp_session` and `csrftoken` cookies. Even so, **never turn
`DEBUG` on in production**.

## Deployment checklist

Before going live, run the deployment checks:

```bash
php manage.php check --deploy
```

```text
! [security.W018] DEBUG is on. Never run a production site with DEBUG = true.
    HINT: Set DEBUG=false in the production .env.
! [security.W020] ALLOWED_HOSTS is empty, so every request will be rejected.
    HINT: List the domain names this site serves.
! [security.W004] SECURE_HSTS_SECONDS is not set, so browsers are not told to always use HTTPS.
    HINT: Set it (e.g. 31536000) once the whole site works over HTTPS.
! [security.W012] SESSION_COOKIE_SECURE is off, so the session cookie can travel over plain HTTP.
    HINT: Set SESSION_COOKIE_SECURE to true on an HTTPS site.
! [security.W016] CSRF_COOKIE_SECURE is off, so the CSRF cookie can travel over plain HTTP.
    HINT: Set CSRF_COOKIE_SECURE to true on an HTTPS site.

System check identified 5 issues.
```

| Check | Problem | Fix |
|---|---|---|
| `security.W001` | `SecurityHeadersMiddleware` is not in `MIDDLEWARE`. | Add it. |
| `security.W003` | `CsrfViewMiddleware` is not in `MIDDLEWARE`. | Add it. |
| `security.W004` | `SECURE_HSTS_SECONDS` is not set. | Set it (e.g. `31536000`) once the whole site works over HTTPS. |
| `security.W009` | `SECRET_KEY` is short or looks insecure. | `php manage.php key:generate` |
| `security.W012` | `SESSION_COOKIE_SECURE` is off (with `SessionMiddleware`). | Set it to `true`; in the skeleton, `SECURE_COOKIES=true` in `.env`. |
| `security.W016` | `CSRF_COOKIE_SECURE` is off (with `CsrfViewMiddleware`, unless `CSRF_USE_SESSIONS`). | Set it to `true`; in the skeleton, `SECURE_COOKIES=true` in `.env`. |
| `security.W018` | `DEBUG` is on. | `DEBUG=false` in the production `.env`. |
| `security.W020` | `ALLOWED_HOSTS` is empty. | List the domain names of the site. |

Also, outside of what the checks can see:

- serve the site over HTTPS only;
- keep `.env` out of git, and give the production site its own
  `SECRET_KEY`;
- turn on `AUTH_PASSWORD_VALIDATORS`;
- give staff users only the [admin permissions](admin.md#permissions)
  they need, and keep superusers few;
- run `clearsessions` from cron if you set `SESSION_GC_PROBABILITY` to `0`;
- never serve uploaded files from a folder where PHP runs.
