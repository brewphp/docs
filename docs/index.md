---
title: BrewPHP
hide:
  - navigation
---

<div class="brew-hero" markdown>

# BrewPHP

<p class="brew-lead">A small, Django-inspired PHP framework. Easy to learn, a core you can read in an afternoon, and zero runtime dependencies.</p>

[Get started](getting-started/installation.md){ .md-button .md-button--primary }
[View on GitHub](https://github.com/brewphp/brewphp){ .md-button }

</div>

## What is BrewPHP?

BrewPHP brings the ideas that make Django pleasant (apps, URLconfs, views,
`manage.php`, an automatic admin) to modern PHP 8.2+. It is written in plain,
strictly typed PHP with no runtime dependencies, so you can open any file of
the framework and understand what it does.

- **Small and readable.** No magic that can't be explained in one paragraph.
- **Django feel, PHP idioms.** Familiar concepts, idiomatic modern PHP.
- **Secure by default.** Escaped output, host validation, security headers,
  no stack traces in production.
- **Everything from the command line.** `php manage.php` runs the server,
  checks your project and generates code.

## 30-second example

Create a project and start the development server:

```bash
composer create-project brewphp/skeleton myapp
cd myapp
php manage.php serve
```

Now add a page. Views are plain PHP functions, URLs are mapped in
`urls.php`. Replace `apps/pages/urls.php` with:

```php title="apps/pages/urls.php"
<?php

declare(strict_types=1);

use BrewPHP\Http\Request;

use function Apps\Pages\home;

return [
    path('', home(...), name: 'home'),
    path('hello/<str:name>/', fn (Request $request, string $name) => 'Hello, ' . e($name) . '!', name: 'hello'),
    path('api/status/', fn () => ['status' => 'brewing'], name: 'status'),
];
```

Open <http://127.0.0.1:8000/hello/Ada/> to see `Hello, Ada!`, and
<http://127.0.0.1:8000/api/status/> for a JSON response. Strings become HTML
responses, arrays become JSON, and `reverse('pages:hello', ['name' => 'Ada'])`
builds the URL `/hello/Ada/` from the route name.

## Features

<div class="grid cards" markdown>

-   **Routing**

    ---

    Django-style `path()` patterns with converters, named routes, includes,
    namespaces, groups and `#[Route]` attributes.

    [:octicons-arrow-right-24: Routing](topics/routing.md)

-   **Views**

    ---

    Function views, controller methods and class-based views, with arguments
    filled from the URL and the container.

    [:octicons-arrow-right-24: Views](topics/views.md)

-   **Requests & responses**

    ---

    Simple Request and Response objects, JSON, redirects, streaming, cookies
    and uploads.

    [:octicons-arrow-right-24: Requests and responses](topics/requests-responses.md)

-   **Models and queries**

    ---

    A Django-style ORM for SQLite, MySQL and PostgreSQL: fields, relations,
    lookups, Q and F expressions, aggregation and pagination.

    [:octicons-arrow-right-24: Models](topics/models.md)

-   **Middleware**

    ---

    Wrap every request, a group of routes or a single route. Security headers
    included.

    [:octicons-arrow-right-24: Middleware](topics/middleware.md)

-   **manage.php**

    ---

    Development server, system checks, route listing, key generation and code
    generators.

    [:octicons-arrow-right-24: CLI](cli/overview.md)

-   **Helpful errors**

    ---

    A detailed debug page while developing, clean overridable error pages in
    production.

    [:octicons-arrow-right-24: Configuration](getting-started/configuration.md)

</div>

## Roadmap

BrewPHP is built in small, complete steps. Every feature ships with code,
tests and documentation.

| Version | Scope | Status |
|---|---|---|
| v0.1 | Router, views, request/response, settings, CLI | **Available now** |
| v0.2 | ORM (SQLite, MySQL, PostgreSQL) | **Available now** |
| v0.3 | Templates (Django-like syntax) | **Available now** |
| v0.4 | Migrations and `startapp` | Coming |
| v0.5 | Auth, sessions, CSRF, forms | Coming |
| v0.6 | Automatic admin | Coming |

