# Changelog

All notable changes to BrewPHP are listed here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
uses [Semantic Versioning](https://semver.org/). The canonical file is
[CHANGELOG.md](https://github.com/brewphp/brewphp/blob/main/CHANGELOG.md)
in the framework repository.

## [1.0.0] - 2026-10-16

The first stable release. It brings together the v0.1 foundation, the v0.2
ORM, the v0.3 template language, the v0.4 migrations and `startapp`, the
v0.5 sessions, CSRF protection, authentication and forms, and the v0.6
automatic admin. From now on the public API follows semantic versioning:
no breaking change before 2.0.

### Added

- **CI (1.0):** GitHub Actions run code style, PHPStan and the tests on PHP
  8.2, 8.3 and 8.4 with SQLite, MySQL and PostgreSQL, and the live tests on
  MariaDB.

- **Admin (v0.6):** the `brewphp.admin` app, an automatic admin like
  Django's: `AdminSite` (mounted with `AdminSite::default()->urls()` under
  the `admin` namespace) with a dashboard, app pages, change lists,
  add/change/delete pages with a confirmation that lists cascaded objects,
  object history, and staff login, logout (POST only) and password change;
  `ModelAdmin` with `listDisplay` (fields, `__str__` or admin methods),
  `listDisplayLinks`, `listFilter` (booleans, choices, foreign keys, date
  ranges, values), `searchFields` (`^`, `=` and related lookups),
  `ordering`, `listPerPage`, `fields`, `exclude`, `readonlyFields`,
  `labels`, `formClass`, custom `actions` and "Delete selected", and the
  hooks `getQueryset()`, `getFields()`, `getReadonlyFields()`, `getForm()`,
  `saveModel()`, `saveRelated()`, `deleteModel()` and the
  `has*Permission()` methods; the `#[Register]` attribute and
  auto-discovery of each app's `admin.php`; `UserAdmin` and `GroupAdmin`;
  the `LogEntry` model (`admin_log_entry`) for recent actions and history;
  overridable templates in `admin/`, and a stylesheet and script served
  from a fixed list (no inline code, so a strict CSP works); the
  `admin.E001` to `admin.E004`, `admin.E108` and `admin.E116` checks. See
  [Admin](topics/admin.md).
- **Admin security (v0.6):** only active staff get in; every page checks
  the model's view/add/change/delete permission; `getQueryset()` limits
  every page and action (rows outside it are 404s); filters, sorting and
  actions only accept what the ModelAdmin declares; deleting needs the
  delete permission of every cascaded model; staff who are not superusers
  cannot see superusers or change superuser status, groups or permissions;
  admin pages are never cached. See [Admin](topics/admin.md#security).
- **`startapp` (v0.6):** also writes an `admin.php`.
- **Forms (v0.6):** `AuthenticationForm::getUserUnchecked()` for subclasses
  that add login rules.
- **Database (v0.6):** `Collector::objects()` lists what a deletion would
  remove.

- **Sessions (v0.5):** `SessionMiddleware` and `$request->session()`
  (`get`, `set`, `has`, `remove`, `pull`, `clear`, `flush`, `cycleKey`),
  loaded lazily and saved only when changed; file (default), database
  (`brewphp.sessions` app) and array stores, or your own `SessionStore`;
  data stored as JSON, never unserialized; ids the store does not know are
  never adopted (no session fixation); HttpOnly, SameSite=Lax cookies and
  the `SESSION_*` settings; the `clearsessions` command. See
  [Sessions](topics/sessions.md).
- **Messages (v0.5):** one-time flash messages (`Messages::success()`,
  `info()`, `warning()`, `error()`, `debug()`) and the `messages` context
  processor.
- **CSRF protection (v0.5):** `CsrfViewMiddleware` with masked tokens (a
  different token on every page, against BREACH), the
  `csrfmiddlewaretoken` field or `X-CSRFToken` header, Origin checks,
  Referer checks over HTTPS, `CSRF_TRUSTED_ORIGINS` (with
  `https://*.example.com`), `CSRF_USE_SESSIONS`, the `#[CsrfExempt]`
  attribute, a new token at login, and `csrf_token` in every template
  rendered with a request. See [Security](topics/security.md#csrf-protection).
- **Authentication (v0.5):** the `brewphp.auth` app with `User`, `Group`
  and `Permission` models, and default add/change/delete/view permissions
  created after `migrate`; `Auth::authenticate()`, `login()`, `logout()`
  and `updateSessionAuthHash()`; `AuthenticationMiddleware` and
  `$request->user()`; `ModelBackend` and custom `AUTHENTICATION_BACKENDS`;
  Argon2id or bcrypt-SHA256 hashes upgraded at login; password validators;
  other sessions logged out when the password changes; login throttling;
  `LoginRequired` and `PermissionRequired` route middleware; login, logout
  (POST only) and password change pages with overridable templates; the
  `auth` context processor (`user`, `perms`); the `createsuperuser` and
  `changepassword` commands. See [Authentication](topics/auth.md).
- **Forms (v0.5):** `Form` with fields, `clean<Name>()` hooks, `clean()`,
  errors, prefixes, initial values and escaped, accessible rendering;
  18 field types, including `FileField`, `ModelChoiceField` and
  `ModelMultipleChoiceField` (limited to their QuerySet); widgets for every
  input type, selects, radios and checkboxes; `ModelForm`, which requires
  an explicit field list and runs the model's validation. See
  [Forms](topics/forms.md).
- **Security helpers (v0.5):** `Signer` (HMAC-SHA256 with salts,
  timestamps and signed JSON), `Redirects::isSafe()` against open
  redirects, and `RateLimiter`.
- **Testing (v0.5):** `BrewPHP\Testing\Client`, a test browser that keeps
  cookies, logs users in and provides CSRF tokens. See
  [Testing](topics/testing.md).
- **Templates and HTTP (v0.5):** the `HtmlSafe` interface;
  `Response::vary()`; `Request::session()`, `hasSession()`, `user()` and
  `setUser()`.
- **Checks (v0.5):** `auth.W001` and `auth.E001` for the middleware
  order, and `check --deploy` warnings `security.W003`, `security.W012`
  and `security.W016` for a missing `CsrfViewMiddleware` and insecure
  session or CSRF cookies.

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

### Security

- **Login throttling (v0.6):** an attempt is counted before the password is
  checked, in one locked step, so parallel requests cannot exceed
  `AUTH_LOGIN_ATTEMPTS`; spelling variants of a username that the database
  treats as equal share one counter. See
  [Login throttling](topics/auth.md#login-throttling).
- **Admin (v0.6):** the "all values" filter only lists values of rows from
  `getQueryset()`; the change list needs view (or change) permission; the
  admin login gives the same error for a wrong password and a non-staff
  account; bulk actions check the per-object permission of every selected
  object; admin pages send `X-Frame-Options: DENY` even without
  `SecurityHeadersMiddleware`. See [Admin](topics/admin.md#security).
- **Redirects (v0.6):** `Redirects::isSafe()` refuses URLs with surrounding
  spaces or line breaks (a crafted `next` could make login fail).
- **Uploads (v0.6):** `UploadedFile::hashName()` drops extensions a web
  server could execute (php, phtml, phar, cgi...).
- **Signing (v0.6):** timestamped signatures from the future are refused.

### Fixed

- **Database (1.0):** model metadata is rebuilt when another Application
  becomes the current one, so a model first seen without its app no longer
  keeps the wrong app label.
- **CLI (v0.6):** password prompts work with piped input.
- **Forms (v0.6):** a field without a limit no longer removes `maxlength`,
  `min` or `max` set on its widget.
- **Admin (v0.6):** `AdminSite::default()` (and `AdminSite::of()`) return
  the same site whatever the order of `INSTALLED_APPS`; view-only pages also
  show `readonlyFields`; recent actions use the model's verbose name.

### Changed

- **Migrations (v0.5):** generated files start with
  `return new class () extends Migration {` (PSR-12).
- **Debug page (v0.5):** values whose names contain `SESSION` or `CSRF`
  (such as the session cookie) are masked too.
- **SQLite schema editor (v0.4):** adding a non-null column without a
  default to an empty table, or making a column non-null without a default,
  now works as on MySQL and PostgreSQL; it only fails when a row needs a
  value.
