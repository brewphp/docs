# Project structure

A new project created from the skeleton looks like this:

```text
myapp/
├── manage.php              command-line entry point
├── .env                    machine-specific settings (not in git)
├── .env.example            template for .env (in git)
├── composer.json
├── phpunit.xml
├── config/
│   ├── settings.php        project settings
│   └── urls.php            root URLconf
├── apps/
│   └── pages/              the sample "pages" app
│       ├── App.php         app configuration
│       ├── views.php       views of the app
│       ├── urls.php        URLs of the app
│       ├── templates/
│       │   └── pages/
│       │       └── home.php
│       └── tests/
│           └── HomeTest.php
├── templates/              project-level templates
├── static/                 project static files (CSS, JS, images)
├── media/                  files uploaded by users
├── storage/                cache, logs, sessions, compiled templates
│   ├── cache/
│   ├── logs/
│   ├── sessions/
│   └── templates/
└── public/
    ├── index.php           front controller
    └── .htaccess           rewrite rules for Apache
```

## The files at the root

`manage.php`
:   The command-line entry point, like Django's `manage.py`. Run
    `php manage.php list` to see every command. See [manage.php](../cli/overview.md).

`.env` and `.env.example`
:   Values that change between machines: debug mode, the secret key, host
    names. `.env` is ignored by git; `.env.example` documents what it should
    contain. See [Configuration](configuration.md#the-env-file).

`public/index.php`
:   Every web request goes through this file. Point your web server's
    document root at `public/` so nothing else in the project can be
    downloaded. It contains just two statements:

    ```php title="public/index.php"
    <?php

    declare(strict_types=1);

    require __DIR__ . '/../vendor/autoload.php';

    BrewPHP\Core\Application::create(dirname(__DIR__))->run();
    ```

`config/settings.php`
:   Returns an array of settings. See [Configuration](configuration.md).

`config/urls.php`
:   The root URLconf, the list of routes the site answers. See [Routing](../topics/routing.md).

## Apps

Like in Django, a project is made of **apps**: folders that group the views,
URLs, templates and models of one part of the site, such as a blog or
a shop. Apps are listed in the `INSTALLED_APPS` setting:

```php title="config/settings.php"
'INSTALLED_APPS' => [
    'pages',
],
```

The label `'pages'` refers to the folder `apps/pages/`.

### Autoloading

BrewPHP autoloads the classes of project apps without any Composer
configuration. The rule is:

| Label | Folder | Namespace |
|---|---|---|
| `pages` | `apps/pages/` | `Apps\Pages` |
| `blog` | `apps/blog/` | `Apps\Blog` |
| `blog_posts` | `apps/blog_posts/` | `Apps\BlogPosts` |

A class is loaded from the file that matches the rest of its name, so
`Apps\Blog\Commands\PublishCommand` lives in
`apps/blog/Commands/PublishCommand.php`.

### Single-file modules

You don't need a file per class. When an app is installed, BrewPHP loads its
**single-file modules** if they exist:

- `models.php`
- `views.php`
- `forms.php`

Everything declared in them (functions and classes) is then available
everywhere. That is why the skeleton's `apps/pages/urls.php` can use the
`home()` function declared in `apps/pages/views.php`:

```php title="apps/pages/views.php"
<?php

declare(strict_types=1);

namespace Apps\Pages;

use BrewPHP\Core\Application;
use BrewPHP\Http\Request;
use BrewPHP\Http\Response;

function home(Request $request): Response
{
    return render('pages/home.php', [
        'version' => Application::VERSION,
        'debug' => (bool) settings('DEBUG'),
    ]);
}
```

!!! tip "One file or many"
    Start with `views.php`. When it grows, move classes into their own files
    (for example `apps/blog/Views/PostView.php` with the namespace
    `Apps\Blog\Views`); the autoloader finds them without any change.

### App configuration

`App.php` describes the app. It is optional; without it BrewPHP uses
sensible defaults.

```php title="apps/pages/App.php"
<?php

declare(strict_types=1);

namespace Apps\Pages;

use BrewPHP\Core\AppConfig;

final class App extends AppConfig
{
    public string $verboseName = 'Pages';
}
```

| Property | Default | Meaning |
|---|---|---|
| `$label` | the folder name, e.g. `pages` | Short unique name |
| `$name` | `apps.<label>` | Dotted name |
| `$verboseName` | label with capitals, e.g. `Blog Posts` | Human-readable name |
| `$modules` | `['models.php', 'views.php', 'forms.php']` | Files loaded at boot |

Override `ready(Application $app)` to run code once every app is registered:

```php
public function ready(\BrewPHP\Core\Application $app): void
{
    // Register things here.
}
```

An app published as a Composer package is installed by its class name
instead of a label, for example `Acme\Blog\BlogApp::class`. Its folder and
namespace are taken from the class.

### Inside an app

| Path | Contents |
|---|---|
| `App.php` | App configuration (optional) |
| `views.php` | Views |
| `urls.php` | Routes, mounted from the root URLconf with `include_urls()` |
| `templates/<label>/` | Templates; the extra folder avoids name clashes between apps |
| `models.php` | [Models](../topics/models.md) |
| `Models/` | Models, one class per file (loaded at boot like `models.php`) |
| `Commands/` | `manage.php` commands, discovered automatically |
| `tests/` | Tests of the app |
| `migrations/`, `forms.php`, `admin.php`, `static/<label>/` | Coming in later releases |

## Templates

Template folders are searched in this order, and the first match wins:

1. the folders in the `TEMPLATES` `DIRS` setting (by default the project's `templates/`);
2. each installed app's `templates/` folder, in `INSTALLED_APPS` order;
3. the framework's own templates (welcome page, error pages).

So a project can override any app or framework template by creating a file
with the same name in `templates/`. For example, `templates/404.html`
replaces the production "page not found" page. A project template can even
extend the template it replaces (see
[Template inheritance](../topics/templates.md#template-inheritance)).

Files ending in `.php` are plain PHP templates; every other extension
(`.html`, `.txt`...) uses the [template language](../topics/templates.md).
Compiled templates are cached in `storage/templates/`.

## storage/

The framework writes runtime files here: cache, logs, sessions and compiled
templates. The web server needs write access to it. Its contents are ignored
by git. Change its location with the `STORAGE_PATH` setting.
