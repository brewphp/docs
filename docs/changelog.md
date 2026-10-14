# Changelog

All notable changes to BrewPHP are listed here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
uses [Semantic Versioning](https://semver.org/). The canonical file is
[CHANGELOG.md](https://github.com/brewphp/brewphp/blob/main/CHANGELOG.md)
in the framework repository.

## [Unreleased]

The v0.1 foundation and the v0.2 ORM.

### Added

- **Database (v0.2):** the `DATABASES` setting for SQLite, MySQL/MariaDB and
  PostgreSQL; lazy connections with `db()`, raw SQL helpers, nested
  transactions with savepoints and a query log while `DEBUG` is on.
- **Models (v0.2):** `Model` with fields, an implicit `id`, table names
  from the app label, ordering, indexes, hooks (`beforeSave()`,
  `afterSave()`, `beforeDelete()`, `afterDelete()`) and validation with
  `fullClean()`, `clean()` and `ValidationError`; 20 field types including
  `ForeignKey` and `OneToOneField`, choices from backed enums, and
  `OnDelete` rules applied in PHP.
- **Queries (v0.2):** lazy, immutable QuerySets with Django lookups,
  relations followed with double underscores in both directions, `Q` and
  `F` expressions, ordering, slicing, `values()`, `valuesList()`,
  aggregation (`Count`, `Sum`, `Avg`, `Min`, `Max`), `selectRelated()`,
  bulk `update()` and `delete()`, `getOrCreate()`, `updateOrCreate()`,
  `paginate()`, `raw()` and `get_object_or_404()`.
- **Schema editor (v0.2):** create, change and drop tables from models on
  every engine (SQLite through table rebuilds), until migrations arrive.
- **Pagination (v0.2):** `Paginator` and `Page`.
- **CLI (v0.2):** `about` shows the database; `check` validates
  `DATABASES`.
- **Application and configuration:** `Application` with project paths and
  installed apps, `.env` loading with `env()`, `config/settings.php` with
  framework defaults, and a small dependency-injection container with
  autowiring.
- **Apps:** `INSTALLED_APPS` with labels (`'blog'` → `apps/blog/`,
  namespace `Apps\Blog`) or `AppConfig` class names; app autoloading without
  Composer configuration; single-file modules (`models.php`, `views.php`,
  `forms.php`) loaded at boot; `ready()` hooks.
- **HTTP:** `Request`, `Response`, `JsonResponse`, `RedirectResponse`,
  `StreamedResponse`, `Cookie`, `UploadedFile`, and the HTTP exceptions
  `Http404`, `PermissionDenied`, `BadRequest`, `MethodNotAllowed` and
  `HttpException`.
- **Routing:** `path()` with the `int`, `str`, `slug`, `uuid` and `path`
  converters plus custom converters, `re_path()`, `include_urls()` with
  namespaces, `group()`, HTTP method restrictions, `#[Route]` attributes with
  `routes_from()`, `reverse()` and `url()`, and `APPEND_SLASH` redirects.
- **Views:** function views, controller methods, class-based `View` with
  `asView()`; arguments filled from the request, URL parameters and the
  container; strings and arrays converted to HTML and JSON responses.
- **Middleware:** pipeline with global (`MIDDLEWARE`), group and route
  middleware, closures, and `SecurityHeadersMiddleware`.
- **Templates:** plain PHP templates with `render()` and `e()`, looked up in
  the project, app and framework template folders.
- **Error handling:** debug page with stack trace, request and masked
  settings; debug 404 page listing the tried patterns; overridable production
  pages `404.php`, `403.php`, `400.php` and `500.php`; JSON errors; welcome
  page for projects without routes; `ALLOWED_HOSTS` validation.
- **CLI:** `manage.php` runner with global options, command discovery from
  apps and the `COMMANDS` setting, and the commands `about`, `check`
  (system checks, `--deploy`), `key:generate`, `make:command`, `routes` and
  `serve` / `runserver`.
