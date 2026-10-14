# Routing

A **URLconf** is a PHP file that returns a list of routes. Each route maps
a URL pattern to a [view](views.md). The root URLconf is `config/urls.php`
(the `ROOT_URLCONF` setting):

```php title="config/urls.php"
<?php

declare(strict_types=1);

use BrewPHP\Http\Request;

return [
    path('', fn (Request $request) => 'Home', name: 'home'),
    path('about/', fn () => 'About us', name: 'about'),
    path('posts/<int:id>/', fn (int $id) => "Post $id", name: 'post-detail'),
];
```

When a request comes in, BrewPHP tries the routes **in order** and calls the
view of the first one that matches. The URL parameters (`id` above) are
passed to the view as arguments with the same name.

List every route with `php manage.php routes`:

```text
+---------+------------------+-------------+-----------------------+------------+
| Methods | Pattern          | Name        | View                  | Middleware |
+---------+------------------+-------------+-----------------------+------------+
| ANY     | /                | home        | Closure (urls.php:8)  |            |
| ANY     | /about/          | about       | Closure (urls.php:9)  |            |
| ANY     | /posts/<int:id>/ | post-detail | Closure (urls.php:10) |            |
+---------+------------------+-------------+-----------------------+------------+
3 routes
```

## path()

```php
path(string $route, mixed $view, ?string $name = null, array $methods = [], array $middleware = [])
```

| Argument | Meaning |
|---|---|
| `$route` | The pattern, without a leading slash: `'posts/<int:id>/'`. A leading `/` is allowed and ignored. |
| `$view` | What to call: a closure, a function, `[Controller::class, 'method']`, a `View` class… See [Views](views.md). |
| `$name` | A name to build the URL later with `reverse()`. |
| `$methods` | Allowed HTTP methods, such as `['GET', 'POST']`. Empty means all. |
| `$middleware` | Middleware for this route only. See [Middleware](middleware.md). |

The pattern must match the **whole** path. `posts/` matches `/posts/` but
not `/posts/archive/`.

## Path converters

A part written `<converter:name>` captures a value. The converter decides
what text matches and turns it into a PHP value:

| Converter | Matches | PHP value | Example |
|---|---|---|---|
| `str` (default) | any text without `/` | `string` | `<str:username>` or `<username>` |
| `int` | digits `0-9` | `int` | `<int:id>` |
| `slug` | letters, digits, `-` and `_` | `string` | `<slug:slug>` |
| `uuid` | a lower-case UUID | `string` | `<uuid:uuid>` |
| `path` | any text, including `/` | `string` | `<path:file>` |

```php
return [
    path('users/<username>/', fn (string $username) => "Profile of $username"),
    path('tags/<slug:tag>/', fn (string $tag) => "Tag $tag"),
    path('files/<path:file>', fn (string $file) => "File $file"),
];
```

Values are URL-decoded before matching, so `/users/ada%20lovelace/` gives
`$username = 'ada lovelace'`.

### Custom converters

Register your own converter with a regex, a function to convert the matched
text to PHP, and optionally one to convert a value back to URL text:

```php
use BrewPHP\Routing\Converter;
use BrewPHP\Routing\Router;

Router::registerConverter('year', new Converter(
    '[0-9]{4}',
    fn (string $value): int => (int) $value,
    fn (mixed $value): string => sprintf('%04d', $value),
));
```

Then use it like a built-in one: `path('archive/<year:year>/', ...)`.
Register converters before the routes are loaded: at the top of
`config/urls.php`, or in an app's `ready()` method.

## Regular expressions: re_path()

When a converter is not enough, write the pattern as a regular expression.
Named groups `(?P<name>...)` become view arguments:

```php
return [
    re_path('^archive/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', fn (int $year, string $month) => "$year-$month", name: 'archive'),
];
```

Captured values are strings. When the view declares an `int`, `float` or
`bool` parameter, the string is converted for you, so `int $year` works.

!!! note "Reversing regex routes"
    `reverse()` works for regex routes made of plain text and named groups,
    like the one above. Patterns with other regex syntax (optional parts,
    alternatives…) cannot be reversed and raise `NoReverseMatch`.

## Naming routes and reverse()

Never write URLs by hand in your code. Give the route a name and build the
URL from it:

```php
reverse('post-detail', ['id' => 5]);                    // "/posts/5/"
reverse('post-detail', ['id' => 5], ['page' => 2]);     // "/posts/5/?page=2"
```

Parameters are converted back to URL text and encoded. `reverse()` throws
`BrewPHP\Routing\NoReverseMatch` when the name does not exist (with a
"Did you mean...?" suggestion for typos), when a parameter is missing or
unknown, or when a value does not match the pattern (`['id' => 'abc']` for
an `int`).

To build an **absolute** URL, wrap the path in `url()`:

```php
url(reverse('post-detail', ['id' => 5]));   // "https://example.com/posts/5/"
```

During a request `url()` uses the request's scheme and host. Outside a
request (in a command) it uses the `APP_URL` setting.

`redirect()` also accepts a route name:

```php
return redirect('post-detail', ['id' => 5]);
```

## Including other URLconfs

Each app keeps its own routes in its `urls.php`. Mount them in the root
URLconf with `include_urls()`:

```php title="config/urls.php"
<?php

declare(strict_types=1);

return [
    path('', include_urls('apps/pages/urls.php', namespace: 'pages')),
    path('blog/', include_urls('apps/blog/urls.php', namespace: 'blog')),
];
```

```php title="apps/blog/urls.php"
<?php

declare(strict_types=1);

use BrewPHP\Http\Request;

return [
    path('', fn (Request $request) => 'All posts', name: 'index'),
    path('posts/<int:id>/', fn (int $id) => "Post $id", name: 'post-detail'),
];
```

The prefix `blog/` is added to every included route, so the second route
answers `/blog/posts/5/`.

!!! info "Why `include_urls()` and not `include()`?"
    Django calls this function `include()`, but `include` is a reserved word
    in PHP and cannot be used as a function name. Everything else works the
    same way.

`include_urls()` accepts a file path (relative to the project folder, or
absolute) or an array of routes:

```php
path('api/', include_urls([
    path('status/', fn () => ['ok' => true], name: 'status'),
], namespace: 'api')),
```

### Namespaces

The `namespace` keeps route names of different apps apart. Names inside an
included URLconf are prefixed with the namespace and a colon:

```php
reverse('blog:post-detail', ['id' => 5]);   // "/blog/posts/5/"
reverse('pages:home');                      // "/"
```

Namespaces nest: a URLconf with namespace `admin` included inside `blog`
gives names like `blog:admin:home`.

## Groups

`group()` shares a prefix, middleware and optionally a namespace between
routes in the same file:

```php
use BrewPHP\Middleware\SecurityHeadersMiddleware;

return [
    group('api/v1/', [
        path('status/', fn () => ['ok' => true], name: 'status'),
        path('posts/<int:id>/', fn (int $id) => ['id' => $id], name: 'post'),
    ], middleware: [SecurityHeadersMiddleware::class], namespace: 'api'),
];
```

```php
reverse('api:post', ['id' => 3]);   // "/api/v1/posts/3/"
```

Middleware given to `path()` together with an `include_urls()` applies to
every included route too. Outer middleware runs before inner middleware.

## Restricting HTTP methods

```php
return [
    path('contact/', fn () => 'The form', name: 'contact', methods: ['GET']),
    path('contact/', fn () => 'Thanks!', name: 'contact-send', methods: ['POST']),
];
```

A route that allows `GET` also answers `HEAD`. When the path matches but the
method does not, the response is `405 Method Not Allowed` with an `Allow`
header listing the accepted methods.

!!! tip
    For several methods on one URL, a [class-based view](views.md#class-based-views)
    is often clearer: it answers each method with its own `get()`, `post()`…
    method.

## Attribute routes

As an alternative to `urls.php`, declare routes with the `#[Route]`
attribute on controller methods. On the class, the attribute sets a prefix,
a namespace and middleware for every method:

```php title="apps/blog/PostController.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Http\Request;
use BrewPHP\Routing\Attributes\Route;

#[Route('posts/', namespace: 'posts')]
final class PostController
{
    #[Route('', name: 'list')]
    public function list(Request $request): string
    {
        return 'All posts';
    }

    #[Route('<int:id>/', name: 'detail', methods: ['GET'])]
    public function detail(Request $request, int $id): string
    {
        return "Post $id";
    }
}
```

Register them in a URLconf with `routes_from()`, spreading the result into
the list:

```php title="config/urls.php"
<?php

declare(strict_types=1);

use Apps\Blog\PostController;

return [
    ...routes_from(PostController::class),
];
```

```php
reverse('posts:detail', ['id' => 1]);   // "/posts/1/"
```

A method can carry several `#[Route]` attributes to answer several URLs.

## Trailing slashes

With `APPEND_SLASH` on (the default), a `GET` or `HEAD` request for
`/about` that matches no route is redirected (`301`) to `/about/` when that
URL exists. The query string is kept. Turn it off with
`'APPEND_SLASH' => false`.

## When nothing matches

No match gives a `404`. With `DEBUG` on, the page lists every pattern that
was tried, in order, which makes typos easy to spot. With `DEBUG` off, the
`404.php` template is shown (see [Error pages](../getting-started/configuration.md#error-pages)).

While `DEBUG` is on and the URLconf has no routes at all, `/` shows the
BrewPHP welcome page.

## The matched route

The view can find out which route matched:

```php
use BrewPHP\Http\Request;
use BrewPHP\Routing\RouteMatch;

function show(Request $request): string
{
    /** @var RouteMatch $match */
    $match = $request->attribute('route');

    return $match->name() . ' ' . json_encode($match->parameters);
}
```

## Coming from Django

| Django | BrewPHP |
|---|---|
| `urlpatterns = [...]` | `return [...];` |
| `path('posts/<int:id>/', views.detail, name='detail')` | `path('posts/<int:id>/', detail(...), name: 'detail')` |
| `re_path(r'^archive/(?P<year>[0-9]{4})/$', ...)` | `re_path('^archive/(?P<year>[0-9]{4})/$', ...)` |
| `include('blog.urls', namespace='blog')` | `include_urls('apps/blog/urls.php', namespace: 'blog')` |
| `reverse('blog:detail', kwargs={'id': 5})` | `reverse('blog:detail', ['id' => 5])` |
| `register_converter(YearConverter, 'yyyy')` | `Router::registerConverter('yyyy', new Converter(...))` |
