# Changelog

All notable changes to BrewPHP are listed here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
uses [Semantic Versioning](https://semver.org/). The canonical file is
[CHANGELOG.md](https://github.com/brewphp/brewphp/blob/main/CHANGELOG.md)
in the framework repository.

## [Unreleased]

The v0.1 foundation, the v0.2 ORM, the v0.3 template language and the v0.4
migrations and `startapp`.

### Added

- **Migrations (v0.4):** migration files in each app's `migrations/`
  folder (`0001_initial.php`...), each returning an anonymous `Migration`
  class with `$dependencies`, `$initial`, `$atomic` and `operations()`;
  the operations `CreateModel`, `DeleteModel`, `RenameModel`,
  `AlterModelTable`, `AddField`, `RemoveField`, `AlterField`,
  `RenameField`, `AddIndex`, `RemoveIndex`, `RunSQL` and `RunPHP`, with
  one-off defaults (`preserveDefault: false`); historical model states;
  dependency order, conflict and cycle detection; the `brewphp_migrations`
  table; forwards, backwards and `zero`, fake runs, one transaction per
  migration on SQLite and PostgreSQL (with foreign key checks on SQLite)
  and a consistent-history check. Only what changes the database is
  recorded: options such as `helpText`, `choices` or `onDelete` need no
  migration. See [Migrations](topics/migrations.md).
- **Automatic migrations (v0.4):** `makemigrations` detects new, removed
  and renamed models and fields, changed fields, tables and indexes, asks
  about renames and values for new non-null fields, and writes files that
  contain only plain values (through `var_export()`).
- **Migration commands (v0.4):** `makemigrations` (apps, `--name`,
  `--empty`, `--dry-run`, `--check` for CI, `--merge` for conflicting
  migrations), `migrate` (app, target migration or `zero`, `--fake`,
  `--plan`, `--database`, a refusal to run while migrations conflict, and a
  warning for model changes without migrations), `showmigrations` (`--plan`) and
  `sqlmigrate` (`--backwards`). Migrations are never written into the
  framework or a Composer package.
- **`startapp` (v0.4):** creates `apps/<label>/` with `App.php`,
  `models.php`, `views.php`, `urls.php`, a template and `migrations/`, from
  stubs a project can override in `stubs/app/`. Labels are validated (no
  path tricks, no reserved or installed labels).
- **Apps (v0.4):** `AppConfig::migrationsPath()` and the
  `AppConfig::migrated()` hook, called after every successful `migrate`.
- **CLI (v0.4):** array arguments (`addArgument(array: true)`, shown as
  `[<app>...]`) and `Command::relative()`.
- **Database (v0.4):** `Connection::hasTable()`; `SchemaEditor::execute()`,
  `connection()` and `isCollectOnly()` are public;
  `Field::deconstructForMigration()`.
- **Templates (v0.3):** a Django-like template language compiled to PHP and
  cached in `storage/templates/`: variables with dotted lookups into
  arrays, objects, models, QuerySets and pages; filters; template
  inheritance with `{% extends %}`, `{% block %}` and `{{ block.super }}`;
  `{% include %}`; automatic HTML escaping with `SafeString`, `|safe` and
  `{% autoescape %}`. `.php` templates keep working: the extension picks
  the engine.
- **Template tags and filters (v0.3):** `if`, `for`, `with`, `url`,
  `static`, `now`, `cycle`, `firstof`, `spaceless`, `verbatim`, `filter`,
  `comment`, `csrf_token` and more; 33 filters from `add` to `yesno`,
  including `date`, `floatformat`, `linebreaks` and `json_script`.
- **Custom tags and filters (v0.3):** `Library` with `filter()`,
  `simpleTag()` and compiled `tag()`, registered from an app's `ready()`;
  context processors (`TEMPLATES.CONTEXT_PROCESSORS`) with the built-in
  `request` processor; `render_to_string()` and `mark_safe()`.
- **Template errors (v0.3):** syntax and rendering errors name the template
  and line, and the debug page shows the template source; `TEMPLATES.STRICT`
  reports missing variables.
- **Safety (v0.3):** templates never run PHP from their source, never call
  methods marked `#[AltersData]` (such as `save()` and `delete()`), and
  `json_script` encodes HTML characters.
- **Error pages and routing (v0.3):** `404.html`-style error templates;
  `reverse()` accepts positional parameters.
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
  every engine (SQLite through table rebuilds); the lower-level tool that
  migrations use.
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

### Changed

- **SQLite schema editor (v0.4):** adding a non-null column without a
  default to an empty table, or making a column non-null without a default,
  now works as on MySQL and PostgreSQL; it only fails when a row needs a
  value.
