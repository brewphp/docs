# manage.php

`manage.php` is the command-line tool of every BrewPHP project, like
Django's `manage.py`. It runs the development server, checks your project,
lists routes and generates code.

```bash
php manage.php <command> [options] [arguments]
```

Run it without a command (or with `list`) to see every command:

```text
BrewPHP 0.1.0-dev

Usage:
  php manage.php <command> [options] [arguments]

Global options:
  -h, --help              Show help for a command
  -n, --no-interaction    Do not ask any questions
      --ansi|--no-ansi    Force (or disable) coloured output
  -v, --verbose           Show stack traces for errors
  -V, --version           Show the BrewPHP version

Available commands:
  about         Show versions, environment and installed apps
  check         Check the project for common problems
  routes        List every URL pattern with its name, view and middleware
  serve         Start the development server [runserver]
 key
  key:generate  Generate a new SECRET_KEY and write it to .env
 make
  make:command  Create a new manage.php command in an app
```

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
  BrewPHP version .................................. 0.1.0-dev
  PHP version ......................................... 8.2.34
  Operating system ..................................... Linux
  Project path ............................... /home/ada/myapp
  Debug mode .............................................. ON
  Time zone .............................................. UTC
  Language ................................................ en
  URLconf .................................... config/urls.php
  Database ............................ sqlite (storage/db.sqlite3)

Installed apps
  pages ........................................... apps.pages

Extensions
  PDO drivers ........................... mysql, pgsql, sqlite
  mbstring ............................................... yes
  intl ................................................... yes
  openssl ................................................ yes
  sodium ................................................. yes
  OPcache ................................................ off
```

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

System check identified 3 issues.
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
| `addArgument($name, $description, required: false, default: null)` | A positional argument |
| `addOption($name, $description, shortcut: null, value: false, default: null)` | An option. Without `value: true` it is a flag: `true` when given, `false` otherwise |
| `argument($name)` | Read an argument |
| `option($name)` | Read an option |

The shortcuts `h`, `n`, `v` and `V` are taken by the global options.

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
