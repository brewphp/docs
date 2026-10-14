# Views

A **view** takes a request and returns a response. Routes point to views
(see [Routing](routing.md)). BrewPHP accepts several kinds of views, so you
can start simple and grow.

## Function views

The simplest view is a function. Put it in your app's `views.php`, which
BrewPHP loads when the app boots:

```php title="apps/blog/views.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Http\Request;
use BrewPHP\Http\Response;

function post_detail(Request $request, int $id): Response
{
    return render('blog/post_detail.php', ['id' => $id]);
}
```

```php title="apps/blog/urls.php"
<?php

declare(strict_types=1);

use function Apps\Blog\post_detail;

return [
    path('posts/<int:id>/', post_detail(...), name: 'post-detail'),
];
```

`post_detail(...)` is PHP's first-class callable syntax: it refers to the
function without calling it. A fully qualified name as a string,
`'Apps\Blog\post_detail'`, works too.

For tiny views, a closure right in `urls.php` is fine:

```php
path('ping/', fn () => 'pong', name: 'ping'),
```

## Controller methods

Group related views as methods of a class and point the route to a
`[Class::class, 'method']` pair:

```php title="apps/blog/views.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Http\Request;

final class PostController
{
    public function list(Request $request): string
    {
        return 'All posts, page ' . e($request->query('page', '1'));
    }

    public function detail(Request $request, int $id): string
    {
        return "Post $id";
    }
}
```

```php title="apps/blog/urls.php"
<?php

declare(strict_types=1);

use Apps\Blog\PostController;

return [
    path('', [PostController::class, 'list'], name: 'post-list'),
    path('posts/<int:id>/', [PostController::class, 'detail'], name: 'post-detail'),
];
```

The controller is created for you by the container, so its constructor can
ask for services (see [Dependency injection](#dependency-injection)).
Static methods are called without creating an object.

## Class-based views

A class-based view extends `BrewPHP\Views\View` and has one method per HTTP
verb: `get()`, `post()`, `put()`, `patch()`, `delete()`, `head()` and
`options()`. Point the route at the class name:

```php title="apps/pages/views.php"
<?php

declare(strict_types=1);

namespace Apps\Pages;

use BrewPHP\Http\Request;
use BrewPHP\Http\Response;
use BrewPHP\Views\View;

final class ContactView extends View
{
    public string $title = 'Contact us';

    public function get(Request $request): string
    {
        return '<h1>' . e($this->title) . '</h1><form method="post"><button>Send</button></form>';
    }

    public function post(Request $request): Response
    {
        return redirect('/contact/?sent=1', status: 303);
    }
}
```

```php title="apps/pages/urls.php"
<?php

declare(strict_types=1);

use Apps\Pages\ContactView;

return [
    path('contact/', ContactView::class, name: 'contact'),
];
```

What happens for each request:

- the method named after the HTTP verb is called (`GET` calls `get()`);
- `HEAD` falls back to `get()` when there is no `head()` method;
- `OPTIONS` is answered automatically with `204 No Content` and an `Allow`
  header listing the methods the view has;
- any other verb gets `405 Method Not Allowed`, with the same `Allow` header.

Inside the handler, `$this->request` is the current request and
`$this->params` holds the URL parameters.

### asView(): configuring a view in the URLconf

`asView()` creates a configured view by setting its public properties. Each
request works on its own copy, so one request can never affect another:

```php
use Apps\Pages\ContactView;

return [
    path('contact/', ContactView::class, name: 'contact'),
    path('support/', ContactView::asView(['title' => 'Get support']), name: 'support'),
];
```

`asView()` throws if a property does not exist or is not public, or if the
view's constructor has required arguments.

### setup() and dispatch()

Override `setup()` to prepare state shared by all handlers, and `dispatch()`
to run code around every handler. Always call the parent method:

```php
<?php

declare(strict_types=1);

namespace Apps\Pages;

use BrewPHP\Http\Request;
use BrewPHP\Views\View;
use Closure;

final class TimedView extends View
{
    private float $start = 0.0;

    public function setup(Request $request, array $params): void
    {
        parent::setup($request, $params);
        $this->start = microtime(true);
    }

    public function dispatch(Request $request, Closure $call): mixed
    {
        $response = parent::dispatch($request, $call);
        error_log(sprintf('%s took %.3fs', static::class, microtime(true) - $this->start));

        return $response;
    }

    public function get(): string
    {
        return 'Timed!';
    }
}
```

!!! note "Generic views are coming"
    Django-style generic views (`TemplateView`, `ListView`, `DetailView`,
    `CreateView`, `UpdateView`, `DeleteView`) and `ApiView` are coming in a
    later release, together with the template engine and the ORM.

## How arguments are filled

Every kind of view (and every class-based handler method) gets its
arguments the same way. For each parameter, in order:

1. a parameter typed `Request` (or a subclass) receives the current request;
2. a parameter whose **name** matches a URL parameter receives its value;
3. a parameter typed with another class or interface is taken from the
   [container](#dependency-injection);
4. otherwise its default value is used, or `null` if it is nullable.

If none of these apply, the request fails with a clear error naming the
parameter.

URL parameters keep the type their converter gives (`<int:id>` gives an
`int`). Strings captured by `re_path()` are converted when the parameter is
typed `int`, `float` or `bool`.

```php
// URL pattern: 'posts/<int:id>/<slug:slug>/'
function show(Request $request, int $id, string $slug, string $format = 'html'): string
{
    return "$id $slug $format";   // $format keeps its default
}
```

## What a view can return

| Return value | Response |
|---|---|
| a `Response` (or subclass) | sent as it is |
| a `string` or `Stringable` | HTML response, status 200 |
| an `array` or `JsonSerializable` | `JsonResponse`, status 200 |
| anything else (including `null`) | error: views must return a response |

```php
path('hello/', fn () => '<h1>Hello</h1>'),                    // HTML
path('api/hello/', fn () => ['message' => 'Hello']),          // JSON
path('teapot/', fn () => response("I'm a teapot", 418)),      // custom status
```

See [Requests and responses](requests-responses.md) for every response
class and helper.

## Rendering templates

`render()` renders a template and wraps it in a response:

```php
return render('blog/post_detail.php', ['id' => $id, 'title' => 'Hello']);
```

The template is searched in the project's `templates/`, then each app's
`templates/` folder (see [Project structure](../getting-started/project-structure.md#templates)).
In v0.1 templates are plain PHP; the context values become variables. Always
escape what you print with `e()`:

```php title="apps/blog/templates/blog/post_detail.php"
<h1><?= e($title) ?></h1>
<p>Post number <?= e($id) ?></p>
```

!!! warning "Escape everything"
    Printing `<?= $title ?>` without `e()` lets a user who controls the value
    inject HTML and JavaScript (XSS). The template engine coming in v0.3
    escapes automatically.

## Errors in views

Stop with an error page by throwing an HTTP exception, or with `abort()`:

```php
use BrewPHP\Http\Http404;

function post_detail(int $id): string
{
    if ($id > 100) {
        throw new Http404('No such post.');
    }

    return "Post $id";
}

function members(): never
{
    abort(403, 'Members only.');
}
```

Any other exception becomes a `500` error: a detailed page with `DEBUG` on,
the `500.php` template in production. See
[Error pages](../getting-started/configuration.md#error-pages).

## Dependency injection

BrewPHP has a small service container. `app()` returns the application,
which is the container. Register services in an app's `ready()` method:

```php title="apps/blog/App.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Core\AppConfig;
use BrewPHP\Core\Application;

final class App extends AppConfig
{
    public function ready(Application $app): void
    {
        // A new instance every time:
        $app->bind(Clock::class, SystemClock::class);
        // One shared instance, built on first use:
        $app->singleton(Mailer::class, fn () => new Mailer('smtp.example.com'));
    }
}
```

Then ask for the service by type in a view or a controller constructor:

```php
final class PostController
{
    public function __construct(private readonly Mailer $mailer)
    {
    }

    public function share(Request $request, int $id, Clock $clock): string
    {
        // $this->mailer and $clock come from the container.
        return 'Shared';
    }
}
```

Classes that are not registered are built automatically ("autowiring"):
their constructor arguments are resolved the same way. Interfaces must be
bound to a concrete class.

| Method | Purpose |
|---|---|
| `bind($id, $concrete)` | factory or class, new instance every time |
| `singleton($id, $concrete)` | factory or class, built once |
| `instance($id, $value)` | store a ready value |
| `get($id)` / `app($id)` | resolve an entry |
| `make($class, $parameters)` | build a class, with some constructor arguments given by name |
| `call($callable, $parameters)` | call a function, resolving its arguments |
| `has($id)` | whether something is registered |
