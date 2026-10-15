# Testing

BrewPHP projects are tested with [PHPUnit](https://phpunit.de/). The
skeleton includes it, a `phpunit.xml` and a first test in
`apps/pages/tests/`. Run the tests with:

```bash
composer test          # or: vendor/bin/phpunit
```

`phpunit.xml` finds the tests in every app's `tests/` folder and forces a
few environment variables, so tests behave like production whatever your
`.env` says:

```xml title="phpunit.xml"
<php>
    <env name="DEBUG" value="false" force="true"/>
    <env name="ALLOWED_HOSTS" value="localhost" force="true"/>
    <env name="SECRET_KEY" value="test-secret-key-do-not-use-in-production" force="true"/>
</php>
```

## Sending requests

The simplest test boots the application and hands it a request built with
`Request::create()`. No web server is involved:

```php title="apps/pages/tests/HomeTest.php"
<?php

declare(strict_types=1);

namespace Apps\Pages\Tests;

use BrewPHP\Core\Application;
use BrewPHP\Http\Request;
use PHPUnit\Framework\TestCase;

final class HomeTest extends TestCase
{
    public function testHomePageLoads(): void
    {
        $app = Application::create(dirname(__DIR__, 3));

        $response = $app->handle(Request::create('GET', '/'));

        self::assertSame(200, $response->status());
        self::assertStringContainsString('<title>Welcome to BrewPHP</title>', $response->content());
        self::assertSame('/', reverse('pages:home'));
    }
}
```

The response has `status()`, `content()`, `getHeader('Location')`,
`headers()` and `cookies()`. See
[Requests and responses](requests-responses.md).

## The test client

Pages with forms and logins need cookies kept between requests: the
session cookie, the CSRF cookie. `BrewPHP\Testing\Client` is a small
browser for tests, like Django's test client. It sends requests to the
application and keeps the cookies it gets back.

```php
use BrewPHP\Testing\Client;

$client = new Client($app);

$client->get('/accounts/login/');
$response = $client->post('/accounts/login/', [
    'username' => 'ann',
    'password' => 'correct horse battery',
    'csrfmiddlewaretoken' => $client->csrfToken(),
]);
```

| Method | Purpose |
|---|---|
| `new Client($app, $server)` | A client; `$server` holds `$_SERVER` defaults for every request. |
| `get($uri, $query, $headers)` | A `GET` request. `$query` adds query-string values. |
| `post($uri, $data, $headers, $files)` | A `POST` with form data and, optionally, `UploadedFile` objects. |
| `request($method, $uri, $parameters, $headers, $files)` | Any method. |
| `csrfToken()` | A valid CSRF token for the next form post; creates the CSRF cookie when there is none. |
| `login($user)` | Logs a `User` in without the login form, by creating their session. |
| `cookie($name)` / `setCookie($name, $value)` | Read or set a cookie of the client. |
| `clearCookies()` | Forget every cookie: logs out and drops the session. |

Headers are given by name, such as `['X-CSRFToken' => $client->csrfToken()]`
or `['Accept' => 'application/json']`. A full URL such as
`https://localhost/path/` makes an HTTPS request.

Uploads are built with `UploadedFile::fake($path, $clientName)`:

```php
use BrewPHP\Http\UploadedFile;

$client->post('/avatar/', ['csrfmiddlewaretoken' => $client->csrfToken()], files: [
    'avatar' => UploadedFile::fake(__DIR__ . '/fixtures/ann.png', 'ann.png'),
]);
```

## Testing with a database

Tests that use models or logins need tables. Give each test a fresh
in-memory SQLite database, sessions in memory, and run the migrations in
`setUp()`:

```php title="apps/pages/tests/LoginTest.php"
<?php

declare(strict_types=1);

namespace Apps\Pages\Tests;

use BrewPHP\Auth\Models\User;
use BrewPHP\Core\Application;
use BrewPHP\Core\Settings;
use BrewPHP\Db\Migrations\Migrator;
use BrewPHP\Sessions\ArraySessionStore;
use BrewPHP\Testing\Client;
use PHPUnit\Framework\TestCase;

final class LoginTest extends TestCase
{
    private Application $app;

    private Client $client;

    protected function setUp(): void
    {
        $base = dirname(__DIR__, 3);
        $settings = require $base . '/config/settings.php';
        $this->app = new Application($base, new Settings(array_replace($settings, [
            'DATABASES' => ['default' => ['ENGINE' => 'sqlite', 'NAME' => ':memory:']],
            'SESSION_ENGINE' => 'array',
            // Fast hashes: tests create many users.
            'PASSWORD_HASHER' => 'bcrypt',
            'PASSWORD_HASHER_OPTIONS' => ['cost' => 4],
        ])));
        $this->app->boot();

        $migrator = new Migrator($this->app);
        $executor = $migrator->executor();
        $executor->migrate($executor->plan());
        $migrator->runMigratedHooks();   // creates the default permissions

        ArraySessionStore::reset();
        $this->client = new Client($this->app);
    }

    protected function tearDown(): void
    {
        Application::setInstance(null);
    }

    public function testLogin(): void
    {
        User::createUser('ann', 'ann@example.com', 'correct horse battery');

        $this->client->get('/accounts/login/');
        $response = $this->client->post('/accounts/login/', [
            'username' => 'ann',
            'password' => 'correct horse battery',
            'csrfmiddlewaretoken' => $this->client->csrfToken(),
        ]);

        self::assertSame(302, $response->status());
        self::assertSame('/', $response->getHeader('Location'));
        self::assertStringContainsString('ann', $this->client->get('/')->content());
    }

    public function testFormsNeedTheCsrfToken(): void
    {
        $response = $this->client->post('/accounts/login/', ['username' => 'ann', 'password' => 'x']);

        self::assertSame(403, $response->status());
    }

    public function testPasswordChangeNeedsALogin(): void
    {
        $ann = User::createUser('ann', 'ann@example.com', 'correct horse battery');

        self::assertSame(302, $this->client->get('/accounts/password_change/')->status());

        $this->client->login($ann);
        self::assertSame(200, $this->client->get('/accounts/password_change/')->status());
    }
}
```

A few notes:

- The settings file is loaded and some values replaced before the
  application boots, so the database connection is never opened on the
  real database. `Application::create()` would also read `.env`; here the
  environment comes from `phpunit.xml`.
- `SESSION_ENGINE` `'array'` keeps sessions in memory. They go through JSON
  like the other stores, so a test catches values that cannot be stored.
  `ArraySessionStore::reset()` forgets the sessions of earlier tests.
- `runMigratedHooks()` calls every app's `migrated()` hook, as `migrate`
  does; the auth app creates the default permissions there.
- `Application::setInstance(null)` in `tearDown()` makes sure the next test
  does not see this application through helpers such as `reverse()`.

## Assertions that matter for security

Tests are a good place to pin down what must never break:

```php
// Pages that need a login redirect to it.
self::assertSame('/accounts/login/?next=%2Fdashboard%2F', $this->client->get('/dashboard/')->getHeader('Location'));

// Logout only accepts POST.
self::assertSame(405, $this->client->get('/accounts/logout/')->status());

// A user without the permission gets a 403.
$this->client->login($ann);
self::assertSame(403, $this->client->get('/posts/new/')->status());

// The session cookie is HttpOnly.
self::assertTrue($response->cookies()['brewphp_session']->httpOnly);
```
