# Sessions

A session keeps data about one visitor between requests: the contents of a
cart, the logged-in user, a message to show on the next page. BrewPHP
sessions work like Django's: the browser only holds a random session id in
a cookie, and the data stays on the server.

```php
use BrewPHP\Http\Request;

function add_to_cart(Request $request, int $id): \BrewPHP\Http\Response
{
    $cart = $request->session()->get('cart', []);
    $cart[] = $id;
    $request->session()->set('cart', $cart);

    return redirect('shop:cart');
}
```

## Enabling sessions

Add `SessionMiddleware` to `MIDDLEWARE`. The skeleton already does:

```php title="config/settings.php"
use BrewPHP\Auth\AuthenticationMiddleware;
use BrewPHP\Middleware\SecurityHeadersMiddleware;
use BrewPHP\Security\CsrfViewMiddleware;
use BrewPHP\Sessions\SessionMiddleware;

'MIDDLEWARE' => [
    SecurityHeadersMiddleware::class,
    SessionMiddleware::class,
    CsrfViewMiddleware::class,
    AuthenticationMiddleware::class,
],
```

Put it before every middleware that uses the session:
`AuthenticationMiddleware` needs it, and so does `CsrfViewMiddleware` when
`CSRF_USE_SESSIONS` is on. See [Middleware](middleware.md#recommended-order).

Sessions are stored in files by default, which needs no setup. To keep them
in the database instead, see [Stores](#stores).

## Using the session

`$request->session()` returns the visitor's `BrewPHP\Sessions\Session`.
Without `SessionMiddleware` it throws `ImproperlyConfigured`;
`$request->hasSession()` tells you whether there is one.

```php
$session = $request->session();

$session->set('theme', 'dark');
$session->get('theme');             // 'dark'
$session->get('missing', 'light');  // 'light' (the default)
$session->has('theme');             // true
$session->remove('theme');
$session->pull('coupon');           // read the value and remove it
$session->all();                    // every value, as an array
count($session);                    // how many values
```

| Method | Purpose |
|---|---|
| `get($key, $default)` | A value, or `$default` when the key is missing. |
| `set($key, $value)` | Store a value. It must be [JSON-encodable](#values-are-json). |
| `has($key)` | Whether the key exists (even with a `null` value). |
| `remove($key)` | Delete one value. |
| `pull($key, $default)` | Read a value and remove it. |
| `all()` / `isEmpty()` / `count()` | Every value, whether there are none, how many. |
| `clear()` | Delete every value, keep the session id. |
| `flush()` | Delete every value **and** the stored session; the next save gets a new id. Use it on logout. |
| `cycleKey()` | Keep the values under a new id and delete the old one. Use it when privileges change, such as on login. |
| `id()` | The session id, or `null` for a session that was never saved. |

!!! note "Coming from Django"
    `request.session['cart']` becomes `$request->session()->get('cart')`,
    `del request.session['cart']` becomes `remove('cart')`, and
    `cycle_key()` / `flush()` keep their meaning.

### Values are JSON

Session data is stored as JSON, never with PHP's `serialize()`. Store
strings, numbers, booleans, `null` and arrays of those:

```php
$session->set('cart', [12, 15]);                    // fine
$session->set('filters', ['tag' => 'php', 'page' => 2]);  // fine
$session->set('user_id', $user->pk);                // store ids...
$session->set('user', $user);                       // ...not model objects
```

A value that cannot be encoded makes the save fail with a `JsonException`.
Because data is never unserialized, a session can never create PHP objects,
which closes a whole class of attacks.

### Loaded lazily, saved when changed

The session is read from the store the first time a view (or middleware, or
template) uses it, and written back only when it changed. A page that never
touches the session costs nothing and sets no cookie.

When the session was used, the response gets a `Vary: Cookie` header, so
caches and proxies never give one visitor's page to another. Nothing is
saved when the view fails with a 5xx error.

When the session ends up empty (for example after `clear()` or `flush()`),
the stored session is deleted and the browser's cookie is removed.

## Stores

`SESSION_ENGINE` picks where session data lives:

| Value | Store | Notes |
|---|---|---|
| `'file'` (default) | `FileSessionStore` | One JSON file per session in `storage/sessions/`, readable only by its owner. Needs no setup. |
| `'database'` | `DatabaseSessionStore` | The `sessions_session` table. Install `brewphp.sessions` and run `migrate`. |
| `'array'` | `ArraySessionStore` | In memory, gone when the PHP process ends. For [tests](testing.md). |
| a class name | your store | A class implementing `BrewPHP\Sessions\SessionStore`, built by the container. |

### Database sessions

Use the database when several web servers share the sessions, or when
you prefer not to keep files on disk. Add the built-in `brewphp.sessions`
app and create its table:

```php title="config/settings.php"
'INSTALLED_APPS' => [
    'brewphp.auth',
    'brewphp.sessions',
    'pages',
],
'SESSION_ENGINE' => 'database',
```

```bash
php manage.php migrate
```

```text
Operations to perform:
  Apply all migrations: auth, sessions
Running migrations:
  Applying auth.0001_initial... OK
  Applying sessions.0001_initial... OK
```

The table has three columns: `session_key` (the id, primary key),
`session_data` (JSON) and `expire_date` (indexed). The app also provides
the [`clearsessions`](#expiry-and-clearsessions) command; the file store
works without the app.

### A custom store

Implement `BrewPHP\Sessions\SessionStore` and give its class name to
`SESSION_ENGINE`:

```php title="apps/core/RedisSessionStore.php"
<?php

declare(strict_types=1);

namespace Apps\Core;

use BrewPHP\Sessions\SessionStore;

final class RedisSessionStore implements SessionStore
{
    public function __construct(private readonly \Redis $redis)
    {
    }

    public function read(string $id): ?array
    {
        $json = $this->redis->get('session:' . $id);
        $data = is_string($json) ? json_decode($json, true) : null;

        return is_array($data) ? $data : null;
    }

    public function write(string $id, array $data, int $lifetime): void
    {
        $this->redis->setex('session:' . $id, $lifetime, json_encode($data, JSON_THROW_ON_ERROR));
    }

    public function destroy(string $id): void
    {
        $this->redis->del('session:' . $id);
    }

    public function exists(string $id): bool
    {
        return $this->redis->exists('session:' . $id) > 0;
    }

    public function gc(): int
    {
        return 0; // Redis expires keys by itself.
    }
}
```

```php title="config/settings.php"
'SESSION_ENGINE' => Apps\Core\RedisSessionStore::class,
```

| Method | Must |
|---|---|
| `read($id)` | Return the data, or `null` when the session does not exist or expired. |
| `write($id, $data, $lifetime)` | Save the data for `$lifetime` seconds. |
| `destroy($id)` | Delete the session. |
| `exists($id)` | Tell whether an id is taken (used when a new id is generated). |
| `gc()` | Delete expired sessions and return how many. |

The store is created by the container, so its constructor can ask for
services (bind `\Redis` in the container for the example above). Keep the
data as JSON: never `unserialize()` data you read back.

## Settings

| Setting | Default | Description |
|---|---|---|
| `SESSION_ENGINE` | `'file'` | `'file'`, `'database'`, `'array'` or a `SessionStore` class name. |
| `SESSION_COOKIE_NAME` | `'brewphp_session'` | Name of the cookie holding the session id. |
| `SESSION_COOKIE_AGE` | `1209600` (2 weeks) | Lifetime of the session and its cookie, in seconds. |
| `SESSION_COOKIE_SECURE` | `false` | Only send the cookie over HTTPS. Set it to `true` in production. |
| `SESSION_COOKIE_HTTPONLY` | `true` | Hide the cookie from JavaScript. |
| `SESSION_COOKIE_SAMESITE` | `'Lax'` | `'Lax'`, `'Strict'` or `'None'`; any other value is treated as `'Lax'`. |
| `SESSION_COOKIE_DOMAIN` | `null` | Domain of the cookie, such as `'.example.com'` to share it with subdomains. `null` means the current host only. |
| `SESSION_COOKIE_PATH` | `'/'` | Path of the cookie. |
| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | `false` | Send a browser-session cookie (no expiry date). The stored session still expires after `SESSION_COOKIE_AGE`. |
| `SESSION_SAVE_EVERY_REQUEST` | `false` | Save the session (and so extend its lifetime) on every request that uses it, not only when it changed. |
| `SESSION_GC_PROBABILITY` | `1` | Percentage of requests that delete expired sessions from the store. `0` turns it off (use [`clearsessions`](#expiry-and-clearsessions) instead). |

## Security

The session system is built to resist the usual attacks:

- **Random ids.** Ids are 40 characters from `[a-z0-9]`, made with
  `random_int()`. A cookie with any other shape is ignored, so a crafted
  cookie can never point outside the session folder.
- **No session fixation.** An id sent by the browser is only used when the
  store knows it. An attacker who plants an id of their choosing in a
  victim's browser gets nothing: the victim receives a fresh id.
- **New id at login.** `Auth::login()` calls `cycleKey()`, so an id seen
  before login is useless afterwards. The old session is deleted at once.
- **Clean logout.** `Auth::logout()` calls `flush()`: the data and the
  stored session are deleted.
- **Cookie flags.** The cookie is `HttpOnly` (JavaScript cannot read it)
  and `SameSite=Lax` (it is not sent with cross-site POST requests).
- **No objects.** Data is JSON, never unserialized.

!!! warning "Use HTTPS cookies in production"
    Set `SESSION_COOKIE_SECURE` to `true` on an HTTPS site, so the session
    id never travels over plain HTTP. The skeleton reads it from
    `SECURE_COOKIES` in `.env`. `php manage.php check --deploy` warns
    (`security.W012`) when it is off.

## Expiry and clearsessions

A session expires `SESSION_COOKIE_AGE` seconds after it was last saved.
Expired sessions are no longer read, but they stay in the store until they
are deleted. On about `SESSION_GC_PROBABILITY` percent of requests,
`SessionMiddleware` deletes them.

On a busy site, or with `SESSION_GC_PROBABILITY` set to `0`, delete them
from cron with `clearsessions` (available when `brewphp.sessions` is
installed):

```bash
php manage.php clearsessions
```

```text
✔ Deleted 3 expired sessions.
```

```text title="crontab"
# Every night at 03:00
0 3 * * * cd /var/www/myapp && php manage.php clearsessions --no-ansi
```

It cleans the store chosen by `SESSION_ENGINE`.

## Flash messages

Messages are one-time notifications shown on the next page, like Django's
messages framework: "The post was saved.", "Your password was changed."
They are kept in the session until a template displays them.

```php
use BrewPHP\Http\Request;
use BrewPHP\Sessions\Messages;

function save_post(Request $request): \BrewPHP\Http\Response
{
    // ... save the post ...
    Messages::success($request, 'The post was saved.');

    return redirect('blog:index');
}
```

| Method | Level |
|---|---|
| `Messages::debug($request, $text)` | `debug` |
| `Messages::info($request, $text)` | `info` |
| `Messages::success($request, $text)` | `success` |
| `Messages::warning($request, $text)` | `warning` |
| `Messages::error($request, $text)` | `error` |
| `Messages::add($request, $level, $text)` | any level you choose |
| `Messages::pull($request)` | takes the pending messages out of the session, as a list of `Message` objects |

### Showing messages

Add the `messages` [context processor](templates.md#context-processors)
(the skeleton does):

```php title="config/settings.php"
use BrewPHP\Templates\ContextProcessors;

'TEMPLATES' => [
    'DIRS' => ['templates'],
    'CONTEXT_PROCESSORS' => [
        [ContextProcessors::class, 'request'],
        [ContextProcessors::class, 'auth'],
        [ContextProcessors::class, 'messages'],
    ],
],
```

Then loop over `messages` in your layout:

```html+django title="templates/base.html"
{% if messages %}
    <ul class="messages">
        {% for message in messages %}
            <li class="{{ message.tags }}">{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

Each message prints its text, escaped like any other variable;
`message.tags` and `message.level` give the level (`success`, ...), handy
as a CSS class. Reading the messages removes them from the session, so they
are shown once. If no template reads them, they wait for the next page.
