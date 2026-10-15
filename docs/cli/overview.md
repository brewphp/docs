# manage.php

`manage.php` is the command-line tool of every BrewPHP project, like
Django's `manage.py`. It runs the development server, checks your project,
migrates the database, lists routes and generates code.

```bash
php manage.php <command> [options] [arguments]
```

Run it without a command (or with `list`) to see every command:

```text
BrewPHP 0.6.0-dev

Usage:
  php manage.php <command> [options] [arguments]

Global options:
  -h, --help              Show help for a command
  -n, --no-interaction    Do not ask any questions
      --ansi|--no-ansi    Force (or disable) coloured output
  -v, --verbose           Show stack traces for errors
  -V, --version           Show the BrewPHP version

Available commands:
  about            Show versions, environment and installed apps
  changepassword   Change the password of a user
  check            Check the project for common problems
  clearsessions    Delete expired sessions (run it from cron)
  createsuperuser  Create a user with every permission (and access to the admin)
  makemigrations   Create migrations from changes to your models
  migrate          Apply (or unapply) migrations to the database
  routes           List every URL pattern with its name, view and middleware
  serve            Start the development server [runserver]
  showmigrations   List migrations and whether they are applied
  sqlmigrate       Print the SQL of a migration without running it
  startapp         Create a new app in apps/<label>/
 key
  key:generate     Generate a new SECRET_KEY and write it to .env
 make
  make:command     Create a new manage.php command in an app
```

`changepassword` and `createsuperuser` come from the built-in
`brewphp.auth` app, and `clearsessions` from `brewphp.sessions`; they are
listed when those apps are installed (as in the skeleton).

## How it works

`manage.php` is a short PHP script in your project:

```php title="manage.php"
#!/usr/bin/env php
<?php

declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

exit(BrewPHP\Core\Application::create(__DIR__)->runConsole($argv));
```

It boots the application exactly like a web request does (loading `.env`,
the settings and the installed apps), then runs the command. Commands come
from three places:

1. the framework's built-in commands;
2. every installed app's `Commands/` folder;
3. the `COMMANDS` setting, a list of command class names.

## Global options

Every command accepts these options:

| Option | Effect |
|---|---|
| `-h`, `--help` | Show the command's description, arguments and options. `php manage.php help <command>` does the same. |
| `-n`, `--no-interaction` | Never ask questions; use the default answers. Use it in scripts and CI. |
| `--ansi`, `--no-ansi` | Force or disable colours. Colours are on when the terminal supports them; set the `NO_COLOR` environment variable to turn them off everywhere. |
| `-v`, `--verbose` | Print the stack trace when a command fails. |
| `-V`, `--version` | Print the BrewPHP version. |

Options that take a value can be written `--format=json` or
`--format json`. A command's own short flags can be combined (`-sf` for
`-s -f`), but global options must be written on their own (`-n`, not `-fn`).
Everything after `--` is treated as an argument.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | The command failed (for example, `check` found errors) |
| `2` | Usage error: unknown command or option, missing argument |

```text
$ php manage.php rotues
✘ Command "rotues" is not defined. Did you mean: routes?
```

## Commands

### about

Show versions, the environment and the installed apps.

```bash
php manage.php about [--json]
```

| Option | Meaning |
|---|---|
| `--json` | Print the information as JSON |

```text
Environment
  BrewPHP version .................................. 0.6.0-dev
  PHP version ......................................... 8.2.34
  Operating system ..................................... Linux
  Project path ............................... /home/ada/myapp
  Debug mode .............................................. ON
  Time zone .............................................. UTC
  Language ................................................ en
  URLconf .................................... config/urls.php
  Database ............................ sqlite (storage/db.sqlite3)

Installed apps
  admin ........................................ brewphp.admin
  auth .......................................... brewphp.auth
  sessions .................................. brewphp.sessions
  pages ........................................... apps.pages

Extensions
  PDO drivers ........................... mysql, pgsql, sqlite
  mbstring ............................................... yes
  intl ................................................... yes
  openssl ................................................ yes
  sodium ................................................. yes
  OPcache ................................................ off
```

### changepassword

Change the password of a user (needs `brewphp.auth`).

```bash
php manage.php changepassword <username>
```

```text
Changing password for user "ann"
Password:
Password (again):
✔ Password changed successfully for user "ann".
```

The password is asked twice (hidden where the terminal allows it) and
checked against `AUTH_PASSWORD_VALIDATORS`; after three failed tries
nothing changes. It cannot run with `-n`. The user's other sessions are
logged out. See [Authentication](../topics/auth.md#commands).

### check

Run the system checks: settings, URLconf, folders and security.

```bash
php manage.php check [--deploy] [--fail-level=error|warning]
```

| Option | Meaning |
|---|---|
| `--deploy` | Also run the checks for production deployments |
| `--fail-level=VALUE` | Exit with status 1 at this level or worse: `error` (default) or `warning` |

```text
$ php manage.php check
✔ System check identified no issues.
```

With problems, each one has an identifier and often a hint:

```text
$ php manage.php check --deploy
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

Warnings alone exit with `0`; add `--fail-level=warning` to fail on them in
CI. The checks:

| ID | Level | Problem |
|---|---|---|
| `security.E001` | error | `SECRET_KEY` is empty |
| `core.E001` | error | the storage folder does not exist |
| `core.E002` | error | the storage folder is not writable |
| `urls.E001` | error | the URLconf cannot be loaded |
| `urls.W001` | warning | a route name is used more than once |
| `templates.W001` | warning | a folder in `TEMPLATES` `DIRS` does not exist |
| `security.W018` | warning (deploy) | `DEBUG` is on |
| `security.W009` | warning (deploy) | `SECRET_KEY` is shorter than 50 characters or looks insecure |
| `security.W020` | warning (deploy) | `ALLOWED_HOSTS` is empty |
| `security.W004` | warning (deploy) | `SECURE_HSTS_SECONDS` is not set |
| `security.W001` | warning (deploy) | `SecurityHeadersMiddleware` is not in `MIDDLEWARE` |
| `security.W003` | warning (deploy) | `CsrfViewMiddleware` is not in `MIDDLEWARE` |
| `security.W012` | warning (deploy) | `SESSION_COOKIE_SECURE` is off while `SessionMiddleware` is used |
| `security.W016` | warning (deploy) | `CSRF_COOKIE_SECURE` is off while `CsrfViewMiddleware` is used (and `CSRF_USE_SESSIONS` is off) |
| `auth.W001` | warning | `brewphp.auth` is installed but `AuthenticationMiddleware` is not in `MIDDLEWARE` |
| `auth.E001` | error | `AuthenticationMiddleware` comes before `SessionMiddleware` (or the latter is missing) |
| `admin.E001` | error | `brewphp.admin` is installed without `brewphp.auth` |
| `admin.E002` | error | `brewphp.admin` is installed but `SessionMiddleware` is not in `MIDDLEWARE` |
| `admin.E003` | error | `brewphp.admin` is installed but `AuthenticationMiddleware` is not in `MIDDLEWARE` |
| `admin.E004` | error | `brewphp.admin` is installed but `CsrfViewMiddleware` is not in `MIDDLEWARE` |
| `admin.E108` | error | a `listDisplay` or `listFilter` entry of a ModelAdmin is neither a model field nor an admin method |
| `admin.E116` | error | a `listFilter` entry of a ModelAdmin is not a model field |
| `db.E001` | error | `DATABASES` has no `default` connection |
| `db.E002` | error | a `DATABASES` entry is not an array or has an unknown `ENGINE` |
| `db.E003` | error | the PDO driver of a database engine is not installed |

Apps can register their own checks in `ready()`:

```php
use BrewPHP\Core\Application;
use BrewPHP\Core\CheckMessage;
use BrewPHP\Core\SystemChecks;

public function ready(Application $app): void
{
    $app->get(SystemChecks::class)->register(function (Application $app): array {
        if (env('MAILER_URL') === null) {
            return [CheckMessage::warning('blog.W001', 'MAILER_URL is not set.', 'Add it to .env.')];
        }

        return [];
    });
}
```

Pass `deploy: true` to `register()` for checks that only run with `--deploy`.

### clearsessions

Delete expired sessions from the store chosen by `SESSION_ENGINE` (needs
`brewphp.sessions`). Run it from cron on busy sites or when
`SESSION_GC_PROBABILITY` is `0`.

```bash
php manage.php clearsessions
```

```text
✔ Deleted 3 expired sessions.
```

See [Sessions](../topics/sessions.md#expiry-and-clearsessions).

### createsuperuser

Create a user with `is_staff` and `is_superuser` set, so with access to
the [admin](../topics/admin.md) and every permission (needs
`brewphp.auth`).

```bash
php manage.php createsuperuser [--username=VALUE] [--email=VALUE]
```

| Option | Meaning |
|---|---|
| `--username=VALUE` | The username (asked when missing) |
| `--email=VALUE` | The email address (asked when missing, may be empty) |

The password is asked twice and checked against
`AUTH_PASSWORD_VALIDATORS`; you may bypass a failed rule. It is never
accepted as an argument. With `-n`, `--username` is required and the
password is read from the `BREWPHP_SUPERUSER_PASSWORD` environment
variable; without it the account has no usable password:

```bash
BREWPHP_SUPERUSER_PASSWORD='a long admin passphrase' \
    php manage.php createsuperuser -n --username=admin --email=admin@example.com
```

```text
✔ Superuser created successfully.
```

See [Authentication](../topics/auth.md#commands).

### key:generate

Generate a new random `SECRET_KEY` (50 characters) and write it to `.env`.

```bash
php manage.php key:generate [--show] [-f|--force]
```

| Option | Meaning |
|---|---|
| `--show` | Only print a new key, do not write it |
| `-f`, `--force` | Replace an existing key without asking |

```text
$ php manage.php key:generate
✔ New SECRET_KEY written to .env.
```

The `SECRET_KEY=` line is replaced, or added when it is missing. `.env`
must exist (copy `.env.example` first). When a key already exists you are
asked before replacing it, because changing the key invalidates sessions
and signed links. With `--no-interaction` an existing key is kept and the
command exits with `1`, unless you pass `--force`.

### make:command

Create a new command class in an app's `Commands/` folder.

```bash
php manage.php make:command [options] <class>
```

| Argument / option | Meaning |
|---|---|
| `class` | Class name, e.g. `SendReport` (`Command` is appended if missing) |
| `--app=VALUE` | The app to put the command in. Optional when only one app is installed; otherwise you are asked. |
| `--command=VALUE` | The command name. Default: the class name in kebab-case without `Command`, e.g. `send-report` |
| `-f`, `--force` | Overwrite the file if it exists |

```text
$ php manage.php make:command PublishPosts --app=blog
✔ Created apps/blog/Commands/PublishPostsCommand.php
  Run it with: php manage.php publish-posts
```

### makemigrations

Compare your models with the migrations of each app and write new
migration files for the differences. See [Migrations](../topics/migrations.md).

```bash
php manage.php makemigrations [options] [<app>...]
```

| Argument / option | Meaning |
|---|---|
| `app` | Only these apps (one or more labels). Default: every installed app |
| `--name=VALUE` | Name of the migration after the number, e.g. `add_subtitle`. Lower-case letters, digits and underscores |
| `--empty` | Write an empty migration to fill in by hand. Needs an app |
| `--dry-run` | Show what would be written, write nothing |
| `--check` | Write nothing; exit with `1` when models have changes without migrations (for CI) |
| `--merge` | Write a merge migration for each app (or each given app) with conflicting migrations |

```text
$ php manage.php makemigrations
Migrations for "blog":
  apps/blog/migrations/0001_initial.php
    - Create model Author
    - Create model Post
Run "php manage.php migrate" to apply them.
```

Without changes it prints `No changes detected.` When it cannot tell a
rename from a removal, or needs a value for existing rows, it asks (see
[Renames](../topics/migrations.md#renames) and
[One-off defaults](../topics/migrations.md#one-off-defaults)). With
`--no-interaction` (and with `--check`) it asks nothing: it never assumes
a rename, and stops instead of guessing a value.

```text
$ php manage.php makemigrations blog --empty --name=fill_slugs
Migrations for "blog":
  apps/blog/migrations/0003_fill_slugs.php
Run "php manage.php migrate" to apply them.
```

When an app has more than one latest migration (two branches each added
one), `makemigrations` and `migrate` stop with a "Conflicting migrations"
error. `--merge` fixes it: for each app in conflict it writes
`NNNN_merge_<name1>_<name2>.php`, a migration with no operations that
depends on every latest migration (`merge_<date>_<time>` when the name
would be too long). See [Conflicts](../topics/migrations.md#conflicts).

```text
$ php manage.php makemigrations --merge
Migrations for "blog":
  apps/blog/migrations/0004_merge_0003_post_summary_0003_post_tags.php
    - Merge blog.0003_post_summary, blog.0003_post_tags
Run "php manage.php migrate" to apply them.
```

Without conflicts it prints `No conflicts detected to merge.`

### migrate

Apply (or unapply) migrations so the database matches the migration files.

```bash
php manage.php migrate [options] [<app>] [<migration>]
```

| Argument / option | Meaning |
|---|---|
| `app` | The app to migrate. Default: every app |
| `migration` | The migration to move to: a name or a prefix such as `0002`, or `zero` to unapply every migration of the app. Earlier migrations move forwards, later ones are unapplied |
| `--database=VALUE` | The connection from `DATABASES`. Default: `default` |
| `--fake` | Mark migrations as applied or unapplied without running them |
| `--plan` | Show the operations that would run, and stop |

```text
$ php manage.php migrate
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0001_initial... OK
  Applying blog.0002_post_slug... OK
```

```text
$ php manage.php migrate blog 0001
Operations to perform:
  Target blog.0001_initial
Running migrations:
  Unapplying blog.0002_post_slug... OK
```

```text
$ php manage.php migrate --plan
Planned operations:
blog.0001_initial
    Create model Author
    Create model Post
```

When moving backwards, `--plan` lists each migration's operations in the
order they are undone, last one first.

`migrate` refuses to run, and exits with `1`, while any app has
conflicting migrations:

```text
$ php manage.php migrate
✘ Conflicting migrations in app "blog": blog.0003_post_summary and blog.0003_post_tags. Run "php manage.php makemigrations --merge" to merge them.
```

After a successful run, `migrate` calls every app's
[`migrated()` hook](../topics/migrations.md#after-migrating), and warns
when your models have changes that no migration describes yet.

### routes

List every URL pattern with its methods, name, view and middleware.

```bash
php manage.php routes [--filter=VALUE] [--json]
```

| Option | Meaning |
|---|---|
| `--filter=VALUE` | Only show routes whose pattern or name contains this text |
| `--json` | Print the routes as JSON |

```text
$ php manage.php routes
+---------+------------------+-------------+-----------------------+------------+
| Methods | Pattern          | Name        | View                  | Middleware |
+---------+------------------+-------------+-----------------------+------------+
| ANY     | /                | home        | Closure (urls.php:8)  |            |
| ANY     | /about/          | about       | Closure (urls.php:9)  |            |
| ANY     | /posts/<int:id>/ | post-detail | Closure (urls.php:10) |            |
+---------+------------------+-------------+-----------------------+------------+
3 routes
```

### serve (runserver)

Start PHP's built-in web server for development.

```bash
php manage.php serve [--skip-checks] [<addrport>]
```

| Argument / option | Meaning |
|---|---|
| `addrport` | Address and/or port. Default `127.0.0.1:8000`. Examples: `8080`, `0.0.0.0:8000`, `localhost`, `[::1]:8000` |
| `--skip-checks` | Do not run the system checks first |

```text
$ php manage.php serve
Performing system checks...
System check identified no issues.

BrewPHP development server at http://127.0.0.1:8000/
Quit the server with CTRL-C.
```

Files that exist in `public/` are served directly (except hidden files such
as `.htaccess`); every other request goes to `public/index.php`. The command
stops with an error if the port is already in use. `runserver` is an alias,
for Django habits.

!!! danger "Development only"
    PHP's built-in server is single-threaded and not hardened. Never use it
    in production.

### showmigrations

List the migrations of each app and whether they are applied.

```bash
php manage.php showmigrations [options] [<app>...]
```

| Argument / option | Meaning |
|---|---|
| `app` | Only these apps (one or more labels). Default: every installed app |
| `--database=VALUE` | The connection from `DATABASES`. Default: `default` |
| `--plan` | List every migration in the order `migrate` applies them |

```text
$ php manage.php showmigrations
blog
 [X] 0001_initial
 [ ] 0002_post_slug
```

```text
$ php manage.php showmigrations --plan
[X] blog.0001_initial
[ ] blog.0002_post_slug
```

An installed app without migrations shows `(no migrations)`.

### sqlmigrate

Print the SQL of a migration without running it. The SQL is generated for
the engine of the connection.

```bash
php manage.php sqlmigrate [options] <app> <migration>
```

| Argument / option | Meaning |
|---|---|
| `app` | The app label |
| `migration` | The migration name or a prefix, e.g. `0001` |
| `--database=VALUE` | The connection whose SQL dialect to use. Default: `default` |
| `--backwards` | Print the SQL that unapplies the migration |

```text
$ php manage.php sqlmigrate blog 0001
-- Create model Author
CREATE TABLE "blog_author" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "name" varchar(100) NOT NULL);
-- Create model Post
CREATE TABLE "blog_post" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "title" varchar(200) NOT NULL, "body" text NOT NULL, "author_id" bigint NOT NULL, "published_at" datetime NULL, "created_at" datetime NOT NULL, CONSTRAINT "blog_post_author_id_96f44456_fk" FOREIGN KEY ("author_id") REFERENCES "blog_author" ("id") DEFERRABLE INITIALLY DEFERRED);
CREATE INDEX "blog_post_author_id_b94eb986_idx" ON "blog_post" ("author_id");
```

Each operation starts with a `--` comment. A `RunPHP` operation shows only
its comment, because its SQL is only known when the code runs.

### startapp

Create a new app in `apps/<label>/`.

```bash
php manage.php startapp <label>
```

| Argument | Meaning |
|---|---|
| `label` | The app label: starts with a lower-case letter, then lower-case letters, digits and underscores (at most 50 characters) |

```text
$ php manage.php startapp blog
✔ Created the app "blog" in apps/blog/

Next steps:
  1. Add 'blog' to INSTALLED_APPS in config/settings.php
  2. Include its URLs in config/urls.php: path('blog/', include_urls('apps/blog/urls.php', namespace: 'blog')),
  3. Write models in apps/blog/models.php, then run makemigrations and migrate
```

It creates `App.php`, `models.php`, `views.php`, `urls.php`, `admin.php`
(for [admin](../topics/admin.md) registrations), the template
`templates/blog/index.html` and an empty `migrations/` folder (see
[Creating an app](../getting-started/project-structure.md#creating-an-app)).
It refuses labels that are already installed or used by BrewPHP's own apps
(`admin`, `auth`, `brewphp`, `sessions`, `messages`, `static`, `media`),
and never overwrites an existing folder.

The files come from stubs in the framework's `resources/stubs/app/`. To
change them, put your own in `stubs/app/` in your project, for example
`stubs/app/views.php.stub`. The placeholders are `{{ label }}`,
`{{ namespace }}` (e.g. `Apps\Blog`) and `{{ verbose_name }}` (e.g.
`Blog`).

## Writing your own commands

Generate a command with `make:command`, or create a class in an app's
`Commands/` folder. Any non-abstract subclass of `BrewPHP\Console\Command`
there is discovered automatically:

```php title="apps/blog/Commands/GreetCommand.php"
<?php

declare(strict_types=1);

namespace Apps\Blog\Commands;

use BrewPHP\Console\Command;

final class GreetCommand extends Command
{
    public string $name = 'blog:greet';

    public string $description = 'Greet someone';

    public string $help = 'Prints a greeting. Use --shout to print it in capitals.';

    /** @var list<string> */
    public array $aliases = ['greet'];

    protected function configure(): void
    {
        $this->addArgument('who', 'Who to greet', default: 'world');
        $this->addOption('shout', 'Print in capitals', shortcut: 's');
        $this->addOption('times', 'How many times', value: true, default: '1');
    }

    public function handle(): int
    {
        $text = 'Hello, ' . $this->argument('who') . '!';
        for ($i = 0; $i < (int) $this->option('times'); $i++) {
            $this->line($this->option('shout') ? strtoupper($text) : $text);
        }

        return self::SUCCESS;
    }
}
```

```text
$ php manage.php blog:greet Ada --times=2 -s
HELLO, ADA!
HELLO, ADA!
```

Commands whose names contain a colon are grouped by their prefix in the
command list. Commands outside apps can be registered in the `COMMANDS`
setting:

```php title="config/settings.php"
'COMMANDS' => [
    App\Console\BackupCommand::class,
],
```

Commands are built by the container, so the constructor can ask for
services. Inside `handle()`, `$this->app` is the application.

### Arguments and options

| Method | Purpose |
|---|---|
| `addArgument($name, $description, required: false, default: null, array: false)` | A positional argument. With `array: true` it collects every remaining word as a list |
| `addOption($name, $description, shortcut: null, value: false, default: null)` | An option. Without `value: true` it is a flag: `true` when given, `false` otherwise |
| `argument($name)` | Read an argument |
| `option($name)` | Read an option |
| `relative($path)` | A path relative to the project, with forward slashes, for messages |

The shortcuts `h`, `n`, `v` and `V` are taken by the global options.

An array argument must be the last one. It is shown as `[<app>...]` in the
help, and `argument()` returns a list (empty when no word is given):

```php
protected function configure(): void
{
    $this->addArgument('app', 'Only these apps', array: true);
}

public function handle(): int
{
    foreach ($this->argument('app') as $label) {
        $this->line($label);
    }

    return self::SUCCESS;
}
```

### Output

| Method | Prints |
|---|---|
| `line($text)` | plain text |
| `info($text)` | cyan text |
| `success($text)` | `✔` and the text |
| `comment($text)` | grey text |
| `warn($text)` | yellow `! text` |
| `error($text)` | red `✘ text`, on standard error |
| `title($text)` | a bold section title |
| `table($headers, $rows)` | an aligned table |

```php
$this->title('Posts');
$this->table(['ID', 'Title'], [['1', 'Hello'], ['2', 'World']]);
$this->success('Done.');
```

### Asking questions

| Method | Returns |
|---|---|
| `ask($question, $default)` | the answer, or `$default` when empty |
| `secret($question)` | the answer, typed without being shown (on Unix terminals) |
| `confirm($question, $default = false)` | `true` for `y` or `yes` |
| `choice($question, $choices, $default)` | one of `$choices`, picked by number or by value |

```php
$name = $this->ask('Post title', 'Untitled');
if (!$this->confirm("Publish \"$name\" now?", true)) {
    $this->warn('Cancelled.');

    return self::FAILURE;
}
$category = $this->choice('Category', ['news', 'tutorials', 'releases'], 'news');
```

With `--no-interaction`, questions are not asked: `ask()`, `confirm()` and
`choice()` return their default, and `secret()` returns `null`.

### Return values

Return `self::SUCCESS` (0), `self::FAILURE` (1) or `self::INVALID` (2) from
`handle()`. Uncaught exceptions print the error and exit with `1`; add `-v`
to see the stack trace.

### Customising generated code

`make:command` uses the template `resources/stubs/command.stub` of the
framework. To change what it generates, copy it to `stubs/command.stub` in
your project and edit it. The placeholders are `{{ namespace }}`,
`{{ class }}` and `{{ command }}`.

`startapp` works the same way with the stubs in `resources/stubs/app/`
(`App.php.stub`, `models.php.stub`, `views.php.stub`, `urls.php.stub`,
`admin.php.stub` and `index.html.stub`); override them in `stubs/app/`.
