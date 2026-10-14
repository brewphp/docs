# Configuration

Settings live in `config/settings.php`, a PHP file that returns an array.
Names are UPPER_CASE, like in Django. Anything you leave out keeps its
default value.

```php title="config/settings.php"
<?php

declare(strict_types=1);

use BrewPHP\Middleware\SecurityHeadersMiddleware;

return [
    'DEBUG' => env('DEBUG', false),
    'SECRET_KEY' => env('SECRET_KEY', ''),
    'ALLOWED_HOSTS' => array_values(array_filter(array_map('trim', explode(',', (string) env('ALLOWED_HOSTS', ''))))),
    'INSTALLED_APPS' => [
        'pages',
    ],
    'MIDDLEWARE' => [
        SecurityHeadersMiddleware::class,
    ],
    'ROOT_URLCONF' => 'config/urls.php',
    'TEMPLATES' => [
        'DIRS' => ['templates'],
    ],
    'APP_URL' => env('APP_URL', 'http://127.0.0.1:8000'),
    'TIME_ZONE' => env('TIME_ZONE', 'UTC'),
    'LANGUAGE' => 'en',
    'STATIC_URL' => '/static/',
    'MEDIA_URL' => '/media/',
    'DATABASES' => [
        'default' => [
            'ENGINE' => env('DB_ENGINE', 'sqlite'),
            'NAME' => env('DB_NAME', 'storage/db.sqlite3'),
            'HOST' => env('DB_HOST', '127.0.0.1'),
            'PORT' => env('DB_PORT'),
            'USER' => env('DB_USER', ''),
            'PASSWORD' => env('DB_PASSWORD', ''),
        ],
    ],
];
```

## Reading settings

Use the `settings()` helper anywhere in your code. Nested values are read
with dots:

```php
$debug = settings('DEBUG');
$dirs = settings('TEMPLATES.DIRS');
$missing = settings('SOMETHING_ELSE', 'default value');
```

You can also ask the container for the `BrewPHP\Core\Settings` object, which
has typed accessors:

```php
use BrewPHP\Core\Settings;

$settings = app(Settings::class);
$settings->bool('DEBUG');              // bool
$settings->string('TIME_ZONE');        // string
$settings->array('ALLOWED_HOSTS');     // array
$settings->has('TEMPLATES.DIRS');      // bool
```

Your own settings work the same way: add `'BLOG_POSTS_PER_PAGE' => 10` to
the array and read it with `settings('BLOG_POSTS_PER_PAGE')`.

## All settings

### Core

| Setting | Type | Default | Description |
|---|---|---|---|
| `DEBUG` | bool | `false` | Show detailed error pages with stack traces. **Never** `true` in production. |
| `SECRET_KEY` | string | `''` | Secret used to sign values (sessions, password-reset links). Generate it with `php manage.php key:generate`. |
| `ALLOWED_HOSTS` | list of strings | `[]` | Host names this site serves. See [Allowed hosts](#allowed-hosts). |
| `INSTALLED_APPS` | list of strings | `[]` | Apps of the project: labels such as `'blog'` (folder `apps/blog/`) or `AppConfig` class names. |
| `MIDDLEWARE` | list | `[]` | Middleware run around every request, outermost first. See [Middleware](../topics/middleware.md). |
| `ROOT_URLCONF` | string | `'config/urls.php'` | File that returns the root list of routes. Relative paths start at the project folder. |
| `APPEND_SLASH` | bool | `true` | Redirect `/about` to `/about/` when only the latter exists. See [Routing](../topics/routing.md#trailing-slashes). |
| `TEMPLATES` | array | `['DIRS' => ['templates']]` | `DIRS` lists project template folders, searched before app templates. |
| `APP_URL` | string | `'http://localhost'` | Base URL used by `url()` outside a request (commands, emails). |
| `TIME_ZONE` | string | `'UTC'` | Default time zone, applied with `date_default_timezone_set()`. |
| `LANGUAGE` | string | `'en'` | Default language of the site. |
| `STATIC_URL` | string | `'/static/'` | URL prefix of static files. |
| `MEDIA_URL` | string | `'/media/'` | URL prefix of uploaded files. |
| `STORAGE_PATH` | string | `'storage'` | Folder for cache, logs, sessions and compiled templates. |
| `COMMANDS` | list of class names | `[]` | Extra `manage.php` commands outside app `Commands/` folders. |

### Databases

| Setting | Type | Default | Description |
|---|---|---|---|
| `DATABASES` | array | `['default' => ['ENGINE' => 'sqlite', 'NAME' => 'storage/db.sqlite3']]` | Database connections by name; models use `default`. See [Databases](../topics/databases.md). |

Each connection accepts these keys:

| Key | Default | Description |
|---|---|---|
| `ENGINE` | `'sqlite'` | `'sqlite'`, `'mysql'` or `'pgsql'` |
| `NAME` | `'storage/db.sqlite3'` (SQLite) | SQLite file (relative to the project, or `':memory:'`), or the database name |
| `HOST` | `'127.0.0.1'` | Server host |
| `PORT` | `3306` / `5432` | Server port |
| `USER` | `''` | User name |
| `PASSWORD` | `''` | Password |
| `OPTIONS` | `[]` | String keys are added to the PDO DSN, integer keys are PDO attributes |

### Security headers

Used by `SecurityHeadersMiddleware`. See [Middleware](../topics/middleware.md#security-headers).

| Setting | Type | Default | Header |
|---|---|---|---|
| `SECURE_CONTENT_TYPE_NOSNIFF` | bool | `true` | `X-Content-Type-Options: nosniff` |
| `SECURE_FRAME_OPTIONS` | string | `'DENY'` | `X-Frame-Options` |
| `SECURE_REFERRER_POLICY` | string | `'same-origin'` | `Referrer-Policy` |
| `SECURE_CROSS_ORIGIN_OPENER_POLICY` | string | `'same-origin'` | `Cross-Origin-Opener-Policy` |
| `SECURE_HSTS_SECONDS` | int | `0` (off) | `Strict-Transport-Security: max-age=...` (HTTPS requests only) |
| `SECURE_HSTS_INCLUDE_SUBDOMAINS` | bool | `false` | Adds `; includeSubDomains` |
| `SECURE_HSTS_PRELOAD` | bool | `false` | Adds `; preload` |
| `SECURE_CSP` | string or null | `null` (off) | `Content-Security-Policy` |

!!! note "More settings are coming"
    `CACHE`, `MAIL` and the session and CSRF settings arrive with the
    features that use them.

### Allowed hosts

Every request's `Host` header is checked against `ALLOWED_HOSTS`, which
protects against Host-header attacks. A request for any other host gets a
`400 Bad Request`.

| Entry | Matches |
|---|---|
| `'example.com'` | exactly `example.com` |
| `'.example.com'` | `example.com` and every subdomain, such as `www.example.com` |
| `'*'` | any host (avoid in production) |

The port is ignored and the comparison is case-insensitive. While `DEBUG` is
`true` and `ALLOWED_HOSTS` is empty, `localhost`, `*.localhost`, `127.0.0.1`
and `[::1]` are allowed so development just works.

!!! warning
    With `DEBUG` off and `ALLOWED_HOSTS` empty, **every** request is rejected.
    `php manage.php check --deploy` warns you about it.

## The .env file

Values that change between machines (debug mode, secrets, host names) should
not be written in `settings.php`, which is in git. Put them in `.env` at the
root of the project and read them with `env()`:

```ini title=".env"
# Show detailed error pages. Always false in production.
DEBUG=true

SECRET_KEY=6mJ9wS0qVd2-Xc_yLtHn8ZbR4uPeKf1aG7oQiWv3TsYxDlNjCh
ALLOWED_HOSTS=example.com,.example.org
APP_URL=https://example.com
TIME_ZONE=Europe/Paris

# Database (see Databases)
DB_ENGINE=pgsql
DB_NAME=blog
DB_USER=blog
DB_PASSWORD=secret
```

!!! danger "Keep .env out of git"
    The skeleton's `.gitignore` already ignores `.env`. Commit `.env.example`
    instead, with every variable and no secret values.

### Syntax

```ini
# Comments start with #.
NAME=value
export ALSO_OK=value          # an "export " prefix is allowed
QUOTED="hello world"          # double quotes: \n and other escapes work
LITERAL='no $escapes \n here' # single quotes: taken literally
INLINE=value # a comment after " #" is removed from unquoted values
EMPTY=
```

### env()

`env('NAME', $default)` returns the variable, or `$default` when it is not
set. A few special values are converted:

| In `.env` | `env()` returns |
|---|---|
| `true`, `(true)` | `true` |
| `false`, `(false)` | `false` |
| `null`, `(null)` | `null` |
| `empty`, `(empty)` | `''` |
| anything else | the string |

The conversion ignores case, so `TRUE` and `False` work too. Numbers stay
strings; cast them where you use them: `(int) env('PER_PAGE', 10)`.

### Precedence

Real environment variables always win over `.env`. If the server (or
Docker, or your CI) sets `DEBUG=false`, a `DEBUG=true` line in `.env` is
ignored. This lets each environment override the file without editing it.

The order in which a project starts is:

1. `.env` is loaded (if it exists);
2. `config/settings.php` is read, so it can call `env()`;
3. the installed apps are registered.

## Checking your configuration

`php manage.php check` looks for common mistakes, and `--deploy` adds the
production checks:

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

System check identified 3 issues.
```

See [`check`](../cli/overview.md#check) for every check.

## Error pages

With `DEBUG = true`, errors show a detailed page: the exception, the stack
trace with source code, the request and the settings (values whose names
contain `SECRET`, `PASSWORD`, `KEY`, `TOKEN` and similar words are masked).
An unknown URL lists the URL patterns that were tried.

With `DEBUG = false`, a plain page is shown and server errors are written to
PHP's error log. Override a page by creating a template with the status code
as its name in your project's `templates/` folder:

| Template | Shown for |
|---|---|
| `404.php` | Page not found |
| `403.php` | Permission denied |
| `400.php` | Bad request (for example a host not in `ALLOWED_HOSTS`) |
| `500.php` | Server error |

Each template receives `$status`, `$title` and `$message` (the message of
the exception for 4xx errors, empty for server errors):

```php title="templates/404.php"
<!doctype html>
<meta charset="utf-8">
<title>Not found</title>
<h1>We could not find that page.</h1>
<p><?= e($message) ?></p>
<p><a href="/">Go home</a></p>
```

Requests that ask for JSON (an `Accept: application/json` header or a JSON
body) get a JSON error instead. For example `abort(404)` gives
`{"error":"Page not found","status":404}`, and a server error gives
`{"error":"Server error","status":500}`. With `DEBUG` on, the JSON also
contains an `exception` object with the class, message, file, line and trace.
