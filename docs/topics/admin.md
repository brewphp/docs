# The admin

BrewPHP comes with an automatic admin, modelled on `django.contrib.admin`.
It reads your models and builds a site where trusted staff users list,
search, filter, add, change and delete objects, with a history of every
change. You describe each model's page with a few properties:

```php title="apps/blog/admin.php"
#[Register(Post::class)]
final class PostAdmin extends ModelAdmin
{
    public array $listDisplay = ['title', 'author', 'status', 'published_at'];
    public array $listFilter = ['status', 'published_at'];
    public array $searchFields = ['title', 'author__username'];
}
```

The admin is meant for the people who run the site, not for visitors. It
is a quick, safe back office; for anything that needs its own workflow,
write views.

## Enabling the admin

The skeleton has the admin turned on: log in at
<http://127.0.0.1:8000/admin/> with a user made by `createsuperuser`. For an
older project, four steps.

**1. Install the apps.** The admin needs `brewphp.auth` (users and
permissions). `brewphp.sessions` stores sessions in the database; it is
optional with file sessions, but the skeleton installs it. Put
`brewphp.admin` **before your own apps**, so it is ready when their
`ready()` hooks run:

```php title="config/settings.php"
'INSTALLED_APPS' => [
    'brewphp.admin',
    'brewphp.auth',
    'brewphp.sessions',
    'pages',
    'blog',
],
```

**2. Add the middleware.** The admin needs the session, the logged-in user
and CSRF protection:

```php title="config/settings.php"
use BrewPHP\Auth\AuthenticationMiddleware;
use BrewPHP\Middleware\SecurityHeadersMiddleware;
use BrewPHP\Security\CsrfViewMiddleware;
use BrewPHP\Sessions\SessionMiddleware;

'MIDDLEWARE' => [
    SecurityHeadersMiddleware::class,
    SessionMiddleware::class,
    CsrfViewMiddleware::class,
    AuthenticationMiddleware::class,
],
```

`php manage.php check` reports what is missing:

| ID | Problem |
|---|---|
| `admin.E001` | `brewphp.auth` is not in `INSTALLED_APPS` |
| `admin.E002` | `SessionMiddleware` is not in `MIDDLEWARE` |
| `admin.E003` | `AuthenticationMiddleware` is not in `MIDDLEWARE` |
| `admin.E004` | `CsrfViewMiddleware` is not in `MIDDLEWARE` |
| `admin.E108` | a name in `listDisplay` or `listFilter` is neither a field of the model nor a method of the ModelAdmin |
| `admin.E116` | a name in `listFilter` is not a field of the model |

```text
$ php manage.php check
✘ [admin.E004] brewphp.admin needs BrewPHP\Security\CsrfViewMiddleware in MIDDLEWARE.
✘ [admin.E108] Apps\Blog\PostAdmin: "nope" is not a field of Apps\Blog\Post nor a method of the admin.

System check identified 2 issues.
```

**3. Mount the admin** in the root URLconf, under the `admin` namespace:

```php title="config/urls.php"
use BrewPHP\Admin\AdminSite;

return [
    path('admin/', include_urls(AdminSite::default()->urls(), namespace: 'admin')),
    path('', include_urls('apps/pages/urls.php', namespace: 'pages')),
];
```

Any prefix works (`path('backoffice/', ...)`), but the namespace must be
`admin`: the admin builds its links with `reverse('admin:...')`.

**4. Create the tables and a first user.** `migrate` creates the
`admin_log_entry` table (the change history) and the permissions of every
model; `createsuperuser` makes an active staff user with every permission:

```bash
php manage.php migrate
php manage.php createsuperuser
```

Then log in at <http://127.0.0.1:8000/admin/>.

## Registering models

A model shows up in the admin once it is registered with a `ModelAdmin`.
Put the ModelAdmin classes in the app's `admin.php` and mark them with
`#[Register]`. The admin loads the `admin.php` of every installed app at
boot and registers each marked class; `startapp` creates the file for you.

```php title="apps/blog/admin.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Admin\ModelAdmin;
use BrewPHP\Admin\Register;

#[Register(Post::class)]
final class PostAdmin extends ModelAdmin
{
    public array $listDisplay = ['title', 'status', 'published_at'];
    public array $searchFields = ['title'];
}

// One ModelAdmin for several models:
#[Register(Category::class, Tag::class)]
final class NameAdmin extends ModelAdmin
{
    public array $ordering = ['name'];
}
```

A model whose app is not installed is skipped. A model can only be
registered once; registering it twice raises `ImproperlyConfigured`.

Without options, the plain `ModelAdmin` lists objects by their
`__toString()` and edits every editable field. You can also register in
code, from the `ready()` hook of any app:

```php title="apps/blog/App.php"
use BrewPHP\Admin\AdminSite;
use BrewPHP\Core\AppConfig;
use BrewPHP\Core\Application;

final class App extends AppConfig
{
    public function ready(Application $app): void
    {
        $site = AdminSite::default();
        $site->register([Category::class, Tag::class]);       // the default ModelAdmin
        $site->register(Post::class, PostAdmin::class);       // a PostAdmin without #[Register]
    }
}
```

!!! note "One site per application"
    `AdminSite::default()` (or `AdminSite::of($app)`) always returns the same
    site, created on first use, so the order of `INSTALLED_APPS` does not
    matter: an app listed before `brewphp.admin` registers on the same site.

`AdminSite` also has `unregister($model)`, `isRegistered($model)` and
`registry()` (model class => ModelAdmin). To replace the built-in user
admin, unregister `User` and register your own subclass of `UserAdmin`.

## ModelAdmin options

Set these public properties on your subclass:

| Property | Default | Meaning |
|---|---|---|
| `listDisplay` | `['__str__']` | The columns of the list: field names, `'__str__'` (the object's `__toString()`), or methods of the ModelAdmin. |
| `listDisplayLinks` | `[]` (the first column) | The columns that link to the change page. |
| `listFilter` | `[]` | Fields to filter by in the sidebar: booleans, fields with choices, foreign keys, dates, or other values. |
| `searchFields` | `[]` (no search box) | Fields searched by the search box. Prefix with `^` for "starts with", `=` for "equals"; the default is "contains". `__` follows foreign keys. |
| `ordering` | `[]` (the model's ordering) | The default order of the list, e.g. `['-published_at']`. |
| `listPerPage` | `100` | Objects per page of the list. |
| `fields` | `null` (every editable field) | The fields of the form, in order. |
| `exclude` | `[]` | Fields left out of the form. |
| `readonlyFields` | `[]` | Fields or methods shown on the change page but not editable. |
| `labels` | `[]` | Column and read-only field labels by name, e.g. `['word_count' => 'Words']`. |
| `formClass` | `null` | A `ModelForm` class to use instead of the generated form. Its constructor must accept `instance:`. |
| `actions` | `[]` | Custom actions on the selected rows: method name => description. |
| `deleteSelectedAction` | `true` | Whether the "Delete selected" action is offered. |

A full example:

```php title="apps/blog/admin.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Admin\ModelAdmin;
use BrewPHP\Admin\Register;
use BrewPHP\Db\QuerySet;
use BrewPHP\Http\Request;

#[Register(Post::class)]
final class PostAdmin extends ModelAdmin
{
    public array $listDisplay = ['title', 'author', 'status', 'featured', 'published_at', 'word_count'];
    public array $listFilter = ['status', 'featured', 'author', 'published_at'];
    public array $searchFields = ['title', '=status', 'author__username'];
    public array $ordering = ['-published_at'];
    public int $listPerPage = 50;
    public array $exclude = ['author'];
    public array $readonlyFields = ['created_at', 'word_count'];
    public array $labels = ['word_count' => 'Words'];
    public array $actions = ['publish' => 'Publish selected posts'];

    public function wordCount(Post $post): int
    {
        return str_word_count((string) $post->body);
    }

    public function publish(Request $request, QuerySet $posts): void
    {
        $posts->update(['status' => 'published']);
    }
}
```

### listDisplay

Each column is one of:

- **a field name.** Values are shown the way people read them: the label
  of a choice (`Draft`, not `draft`), a ✔/✘ icon for booleans, dates as
  `Oct 15, 2026, 18:06`, `-` for `null`, and the `__toString()` of a
  foreign key. Foreign keys in the list are loaded with `selectRelated()`,
  so the list runs one query, not one per row.
- **`'__str__'`**, the object's `__toString()`.
- **a method of the ModelAdmin**, which receives the object. Name it in
  `snake_case` or `camelCase`: `'word_count'` finds `wordCount()` (or
  `word_count()`). Its label comes from the name (`Word count`) unless
  `labels` says otherwise.

Everything is escaped. To show HTML, return a `SafeString` (or any
`HtmlSafe` object) from the method, and escape what goes into it yourself:

```php
use BrewPHP\Templates\SafeString;

public function statusBadge(Post $post): SafeString
{
    return new SafeString('<strong>' . e($post->status) . '</strong>');
}
```

Field columns can be sorted by clicking their heading (again for
descending); method and `'__str__'` columns cannot. Only the methods of
your subclass count, never those of `ModelAdmin` itself, and they must be
public.

### listFilter

Each name must be a field of the model (`admin.E116`). The choices in the
sidebar depend on the field:

| Field | Choices |
|---|---|
| `BooleanField` | All, Yes, No (and Unknown when it is nullable) |
| a field with `choices` | All and each choice |
| `ForeignKey` | All, each related object (up to 200), and None when it is nullable |
| `DateField`, `DateTimeField` | All, Today, Past 7 days, This month, This year (in `TIME_ZONE`) |
| any other field | All and each distinct value in the table (up to 200) |

Filters are plain query-string parameters such as `?status__exact=draft`
or `?published_at__gte=2026-10-01&published_at__lt=2026-11-01`, so a
filtered list can be bookmarked. Parameters for fields that are not in
`listFilter` are ignored.

The values of an "other field" filter come from `getQueryset()`, so they
never reveal rows the user cannot see.

!!! note
    The choices of a foreign-key filter list the related objects of the
    whole related table (up to 200), whatever `getQueryset()` returns. Only
    filter on a foreign key when every staff user who sees the list may see
    the names of all related objects.

### searchFields

The search box splits the text into words. Every word must match at least
one of the search fields, case-insensitively:

| Entry | Lookup | Matches "tea" |
|---|---|---|
| `'title'` | `title__icontains` | "Green tea", "Steam" |
| `'^title'` | `title__istartswith` | "Tea time" |
| `'=title'` | `title__iexact` | "Tea" |
| `'author__username'` | `author__username__icontains` | posts by "teatime" |

A search over a related field (`__`) returns each object once. The result
count and a link back to the full list are shown next to the box.

### fields, exclude and readonlyFields

Without `fields`, the form has every editable field of the model (not the
auto-increment primary key, nor fields with `editable: false` such as
`autoNowAdd` dates). `exclude` and `readonlyFields` are then taken out.
A ModelAdmin with no editable field left raises `ImproperlyConfigured`.

`readonlyFields` are shown below the form, with the same formatting as the
list. They may be methods, like list columns:

```php
public array $readonlyFields = ['created_at', 'word_count'];
```

Fields that are not in the form are never read from the POST data, so
excluded and read-only fields cannot be changed by a crafted request. Set
them in [`saveModel()`](#saving-and-deleting) instead.

### formClass

To control validation and widgets, give your own [ModelForm](forms.md#modelform):

```php title="apps/blog/forms.php"
final class PostAdminForm extends ModelForm
{
    protected function model(): string
    {
        return Post::class;
    }

    protected function fieldNames(): array
    {
        return ['title', 'body', 'status', 'published_at'];
    }

    protected function widgets(): array
    {
        return ['body' => new Widgets\Textarea(['rows' => 20])];
    }
}
```

```php title="apps/blog/admin.php"
public ?string $formClass = PostAdminForm::class;
```

The admin creates it as `new PostAdminForm($data, $files, instance: $object)`,
with `$object` `null` on the add page. With `formClass`, `fields` and
`exclude` are not used; `readonlyFields` are still shown. To pass more
arguments, such as the user, override [`getForm()`](#hooks).

## Actions

Actions run on the rows ticked in the list. Every list offers "Delete
selected" (to users with the delete permission), which shows what would be
deleted, cascaded objects included, and asks for confirmation. Turn it off
with `public bool $deleteSelectedAction = false;`.

Add your own in `actions`, as method name => description. The method
receives the request and a QuerySet of the selected objects:

```php
use BrewPHP\Db\QuerySet;
use BrewPHP\Http\Request;
use BrewPHP\Http\Response;

public array $actions = [
    'publish' => 'Publish selected posts',
    'exportCsv' => 'Export selected posts as CSV',
];

public function publish(Request $request, QuerySet $posts): void
{
    $posts->update(['status' => 'published']);
}

public function exportCsv(Request $request, QuerySet $posts): ?Response
{
    $rows = ['id,title'];
    foreach ($posts as $post) {
        $rows[] = $post->pk . ',"' . str_replace('"', '""', (string) $post->title) . '"';
    }

    return new Response(implode("\n", $rows), 200, [
        'Content-Type' => 'text/csv; charset=utf-8',
        'Content-Disposition' => 'attachment; filename="posts.csv"',
    ]);
}
```

The signature is `(Request $request, QuerySet $queryset): ?Response`.
Return a `Response` to show your own page or file; return nothing (or
`null`) to go back to the list, which then shows the message "Publish
selected posts: done." (add your own with `Messages::success()` for more
detail). Unlike `listDisplay`, the key in `actions` must be the exact
method name: `'exportCsv'` for `exportCsv()`.

Custom actions are offered to users with the change permission. The
QuerySet only holds selected objects that `getQueryset()` returns **and**
that pass the per-object permission check: `hasChangePermission($request,
$object)` for custom actions, `hasDeletePermission($request, $object)` for
"Delete selected". So an action never touches a row the user cannot see or
may not change. When none of the selected objects pass, nothing runs and
the list shows "You do not have permission to do that to the selected
items. No items have been changed." An action name that the ModelAdmin
does not offer to this user is refused with "No action selected."

## Hooks

Override these methods for behaviour that depends on the request or the
object:

| Method | Purpose |
|---|---|
| `getQueryset(Request $request): QuerySet` | The objects this user may see. Every page and action works on it. |
| `getFields(Request $request, ?Model $object = null): array` | The form's fields (without the read-only ones). |
| `getReadonlyFields(Request $request, ?Model $object = null): array` | The read-only fields; `$object` is `null` on the add page. |
| `getForm(Request $request, ?Model $object, ?array $data = null, array $files = []): Form` | The form of the add and change pages. |
| `saveModel(Request $request, Model $object, Form $form, bool $change): void` | Saves the object of a valid form. |
| `saveRelated(Request $request, Form $form, bool $change): void` | Called after `saveModel()`, for what depends on the saved object. |
| `deleteModel(Request $request, Model $object): void` | Deletes one object (also for "Delete selected", once per object). |
| `label(string $name): string` | The heading of a column or read-only field. |
| `display(Model $object, string $name): string\|HtmlSafe` | The value of a column or read-only field. |
| `hasViewPermission()`, `hasAddPermission()`, `hasChangePermission()`, `hasDeletePermission()` | Who may do what; see [Permissions](#permissions). |

### Saving and deleting

`saveModel()` is the place to set values that do not come from the form,
such as the author. `$change` is `false` on the add page:

```php
use BrewPHP\Db\Model;
use BrewPHP\Forms\Form;
use BrewPHP\Http\Request;

public function saveModel(Request $request, Model $object, Form $form, bool $change): void
{
    if (!$change) {
        $object->author = $request->user();
    }
    parent::saveModel($request, $object, $form, $change);
}
```

The default saves a `ModelForm` with `$form->save()`. `saveRelated()` runs
next, with the saved object in `$form->instance()`.

### Fields and forms per request

```php
public function getReadonlyFields(Request $request, ?Model $object = null): array
{
    // Published posts keep their slug.
    return $object?->status === 'published' ? ['slug', 'created_at'] : ['created_at'];
}

public function getForm(Request $request, ?Model $object, ?array $data = null, array $files = []): Form
{
    return new PostAdminForm($data, $files, instance: $object, user: $request->user());
}
```

The generated form offers every row of a foreign key's model (for example
every user as the author). To limit the choices, use your own form with a
`ModelChoiceField` over a filtered QuerySet, as shown in
[ModelChoiceField](forms.md#modelchoicefield).

## Permissions

Only **active staff** users (`is_active` and `is_staff`) get into the
admin; everyone else is sent to the admin login page, and a logged-in
visitor who is not staff is told the account is not authorized. Being a
superuser is not enough on its own: `createsuperuser` sets `is_staff` too.

Inside, each model checks the default [permissions](auth.md#default-permissions)
of the user, `<app_label>.<action>_<modelname>`:

| Permission | Lets the user |
|---|---|
| `blog.view_post` | see the list and the objects, read-only |
| `blog.add_post` | use the add page |
| `blog.change_post` | edit objects, run custom actions (includes viewing) |
| `blog.delete_post` | delete objects and use "Delete selected" |

Superusers have them all. A model the user has no permission for is not
on the dashboard, and its pages answer 403. The list needs the view (or
change) permission: with only add or delete permission, the model is on
the dashboard without a link to its list, and the list answers 403. Grant permissions to users or
groups in the admin itself, or in code with `$user->grant('blog.view_post')`.

**View-only mode.** A user with the view permission but not the change
permission sees "View post" pages: the values without a form (the form's
fields and the `readonlyFields`), and a Close button instead of Save.
Posting to the page answers 403.

**Cascades.** Deleting an object also deletes the objects that point to it
with `OnDelete::Cascade`. The confirmation page lists them by model (up to
50 of each), and the deletion is refused when the user lacks the delete
permission of any of those models, or when a `Protect` foreign key blocks
it.

**Your own rules.** Override the `has*Permission()` methods. On the change,
delete and history pages, and for each object selected for an action,
they also receive the object:

```php
use BrewPHP\Db\Model;
use BrewPHP\Http\Request;

public function hasChangePermission(Request $request, ?Model $object = null): bool
{
    // Authors edit their own posts; editors edit every post.
    if ($object !== null && $object->author_id === $request->user()->pk) {
        return true;
    }

    return parent::hasChangePermission($request, $object);
}

public function hasDeletePermission(Request $request, ?Model $object = null): bool
{
    return $request->user()->isSuperuser();
}
```

The list and the dashboard call them without an object (to decide what
to show and which actions to offer), so return `true` there when the user
may act on some objects. For rules about which rows a user may reach at
all, use `getQueryset()`.

### Row-level security with getQueryset()

`getQueryset()` returns the objects a user can work with. Every admin page
starts from it: the list and its count, the change, delete and history
pages, and the actions.

```php
use BrewPHP\Db\QuerySet;
use BrewPHP\Http\Request;

public function getQueryset(Request $request): QuerySet
{
    $posts = parent::getQueryset($request);

    return $request->user()->isSuperuser()
        ? $posts
        : $posts->filter(author: $request->user()->pk);
}
```

A row outside the QuerySet does not exist for that user: its change,
delete and history URLs answer 404, ids of such rows sent with an action
are dropped, and "other field" filters do not list its values. Call `parent::getQueryset()` to keep the `ordering`.

## Users and groups

With `brewphp.auth` installed, the admin registers `UserAdmin` for users
and `GroupAdmin` for groups (unless you registered your own).

- **Adding a user** asks for a username and the password twice, checked
  against `AUTH_PASSWORD_VALIDATORS`. The new user is not staff; tick
  "Staff status" on the next page.
- **Passwords are never shown**, not even their hash: the change page
  says how the password is stored (e.g. "Stored as a bcrypt_sha256
  hash") and
  points to `php manage.php changepassword <username>` to set a new one.
- The user list shows username, email, names, staff and active status,
  filters by staff, superuser and active status, and searches usernames,
  names and emails.
- Groups are listed by name; superusers choose each group's permissions.

Staff users who are **not superusers** can be trusted with user accounts
without being able to promote themselves:

- they never see superuser accounts (not in the list, and their pages are
  404);
- the user form has no superuser status, groups or permissions fields,
  and the group form has no permissions field; values posted for them are
  ignored.

## History and recent actions

Every add, change and delete made in the admin is recorded as a
`BrewPHP\Admin\Models\LogEntry` in the `admin_log_entry` table: the time,
the user, the object (`content_type` such as `blog.post`, its id and its
text), the action and a message such as "Changed Title, Body.".

- The dashboard shows the user's 10 latest actions under "Recent actions".
- The History button of a change page lists the object's entries, oldest
  first.

Changes made outside the admin (in views, with `migrate`, in the shell)
are not recorded, and neither are custom actions; log them yourself if
you need to:

```php
use BrewPHP\Admin\Models\LogEntry;

LogEntry::log($request->user(), $post, LogEntry::CHANGE, 'Published by the action.');
```

## Customizing the admin site

`AdminSite::default()` is the site of the application. Change its texts
from the `ready()` hook of any app:

```php title="apps/pages/App.php"
public function ready(Application $app): void
{
    $site = AdminSite::default();
    $site->siteHeader = 'Brew & Co administration';   // the top bar and the login box
    $site->siteTitle = 'Brew & Co admin';             // the end of every <title>
    $site->indexTitle = 'Dashboard';                  // the heading of the start page
    $site->siteUrl = '/shop/';                        // where "View site" goes
}
```

| Property | Default |
|---|---|
| `siteHeader` | `BrewPHP administration` |
| `siteTitle` | `BrewPHP site admin` |
| `indexTitle` | `Site administration` |
| `siteUrl` | `/` |

### URLs

The admin's routes have names in the `admin` namespace, so you can link to
them:

| Name | URL (under `admin/`) | Arguments |
|---|---|---|
| `admin:index` | the prefix itself | |
| `admin:login`, `admin:logout` | `login/`, `logout/` | |
| `admin:password_change`, `admin:password_change_done` | `password_change/`, `password_change/done/` | |
| `admin:app_list` | `<app_label>/` | app label |
| `admin:changelist` | `<app_label>/<model_name>/` | app label, model name |
| `admin:add` | `<app_label>/<model_name>/add/` | app label, model name |
| `admin:change`, `admin:delete`, `admin:history` | `<app_label>/<model_name>/<id>/change/` (`delete/`, `history/`) | app label, model name, id |

```php
reverse('admin:change', ['blog', 'post', $post->pk]);   // /admin/blog/post/7/change/
```

```html+django
{# with the auth context processor #}
{% if user.is_staff %}<a href="{% url 'admin:index' %}">Admin</a>{% endif %}
```

The model name is the lower-case class name.

### Overriding templates

The admin's templates are in the framework's
`resources/templates/admin/`. Templates in your project's `templates/`
folder win, so to change one, copy it to `templates/admin/` with the same
name and edit it:

| Template | Page |
|---|---|
| `admin/base.html` | the layout of every page (blocks `extrahead`, `bodyclass`, `content_title`, `content`) |
| `admin/index.html` | the dashboard with the app list and recent actions |
| `admin/app_index.html` | the page of one app |
| `admin/_app.html` | one app's box with its models (included by the two above) |
| `admin/change_list.html` | the list with search, filters, actions and pages |
| `admin/change_form.html` | the add, change and view pages |
| `admin/delete_confirmation.html` | deleting one object |
| `admin/delete_selected_confirmation.html` | "Delete selected" |
| `admin/_deleted_objects.html` | the list of objects a deletion removes |
| `admin/object_history.html` | an object's history |
| `admin/login.html` | the login page |
| `admin/password_change_form.html`, `admin/password_change_done.html` | the password change pages |

You don't have to copy the whole file: an override may
[extend the template it replaces](templates.md#template-inheritance) and
change one block. For example, to add your own stylesheet to every page:

```html+django title="templates/admin/base.html"
{% extends "admin/base.html" %}

{% block extrahead %}<link rel="stylesheet" href="/static/admin-brand.css">{% endblock %}
```

Every page receives `title`, `site_header`, `site_title`, `site_url`,
`admin_user`, `breadcrumbs` and `admin_messages`. An override applies to
every model; there are no per-model templates.

## Security

The admin can change any data, so it is locked down by default:

- **Staff only.** Every page except the login page needs an active staff
  user, then the model's permission. The login form gives the same error
  for a wrong password and for an account that is not staff ("Please
  enter the correct username and password for a staff account. Note that
  both fields may be case-sensitive."), so it never confirms that a
  guessed password is right, and login throttling applies (see
  [Authentication](auth.md#login-throttling)).
  The `next` parameter of the login page only accepts URLs of this site.
- **CSRF.** Every form, logout included, posts a CSRF token.
- **Logout needs a POST**, so a link or image on another page cannot log
  staff out; `GET /admin/logout/` answers 405.
- **Never cached.** Admin pages and redirects send
  `Cache-Control: no-cache, no-store, must-revalidate, private`, so a
  shared computer's back button or a proxy does not keep them.
- **Only what the ModelAdmin declares.** Filters not in `listFilter`, sort
  columns not in `listDisplay` and actions the user is not offered are
  ignored, so the URL cannot be used to probe other fields.
- **Row-level security.** `getQueryset()` limits every page and action.
- **Escaped output.** Every value is escaped, unless your method returns
  an `HtmlSafe` value.
- **Static files from a fixed list.** `admin.css` and `admin.js` are
  served from `/admin/static/`; no other file can be reached through that
  URL. There is no inline script or style, so the admin works under a
  strict Content Security Policy such as
  `'SECURE_CSP' => "default-src 'self'"` (see
  [Security headers](middleware.md#security-headers)).
- **No framing.** Admin pages send `X-Frame-Options: DENY` against
  clickjacking, even without `SecurityHeadersMiddleware` (which sends it
  for the rest of the site).
- **No search engines.** Pages carry `<meta name="robots" content="noindex, nofollow">`.

On a live site, serve the admin over HTTPS only (`SECURE_COOKIES=true` in
the skeleton's `.env`), give staff accounts strong passwords, and grant
staff users only the permissions they need. See [Security](security.md).

## Limitations

v0.6 covers the common back-office needs. Not there yet:

- **No inlines.** Related objects (the comments of a post) are edited on
  their own pages.
- **No many-to-many widgets** except the groups and permissions of users
  and groups. BrewPHP has no `ManyToManyField` yet.
- **No date hierarchy**, no `list_editable`, and no autocomplete for
  foreign keys (they are plain selects).
- **No per-model templates**; template overrides apply to the whole admin.
- **No password reset for other users** in the admin. Use
  `php manage.php changepassword <username>`.
- **No admin documentation pages** and no custom admin views; write normal
  views protected with
  [PermissionRequired](auth.md#permissionrequired) for those.
