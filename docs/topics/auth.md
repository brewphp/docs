# Authentication

BrewPHP comes with a user system modelled on `django.contrib.auth`: users,
groups and permissions stored in the database, password hashing, login and
logout, pages for logging in and changing a password, and middleware that
protects views.

```php
use BrewPHP\Http\Request;

function dashboard(Request $request): string
{
    $user = $request->user();

    return $user->isAuthenticated()
        ? 'Hello, ' . e($user->getUsername())
        : 'Hello, stranger';
}
```

## Installation

The skeleton has everything turned on. For an older project, five steps:

**1. Install the apps.** `brewphp.auth` holds the users, groups and
permissions; `brewphp.sessions` adds database sessions and the
`clearsessions` command (optional with file sessions, but recommended).

```php title="config/settings.php"
'INSTALLED_APPS' => [
    'brewphp.auth',
    'brewphp.sessions',
    'pages',
],
```

**2. Add the middleware**, in this order:

```php title="config/settings.php"
use BrewPHP\Auth\AuthenticationMiddleware;
use BrewPHP\Middleware\SecurityHeadersMiddleware;
use BrewPHP\Security\CsrfViewMiddleware;
use BrewPHP\Sessions\SessionMiddleware;

'MIDDLEWARE' => [
    SecurityHeadersMiddleware::class,
    SessionMiddleware::class,          // the session keeps the login
    CsrfViewMiddleware::class,         // protects every POST form
    AuthenticationMiddleware::class,   // makes $request->user() work
],
```

`AuthenticationMiddleware` must come after `SessionMiddleware`.
`php manage.php check` reports `auth.W001` when it is missing and
`auth.E001` when the order is wrong.

**3. Add the context processors** that give templates `user`, `perms` and
`messages`:

```php title="config/settings.php"
use BrewPHP\Templates\ContextProcessors;

'TEMPLATES' => [
    'DIRS' => ['templates'],
    'CONTEXT_PROCESSORS' => [
        [ContextProcessors::class, 'request'],
        [ContextProcessors::class, 'auth'],      // user and perms
        [ContextProcessors::class, 'messages'],  // flash messages
    ],
],
```

**4. Include the login pages** in the root URLconf:

```php title="config/urls.php"
use BrewPHP\Auth\Urls as AuthUrls;

return [
    path('', include_urls('apps/pages/urls.php', namespace: 'pages')),
    path('accounts/', include_urls(AuthUrls::patterns())),
];
```

**5. Create the tables and a first user:**

```bash
php manage.php migrate
php manage.php createsuperuser
```

```text
Username: admin
Email address []: admin@example.com
Password:
Password (again):
✔ Superuser created successfully.
```

Log in at <http://127.0.0.1:8000/accounts/login/>. With `brewphp.admin`
installed, the same superuser can also manage users, groups and your
models in the [admin](admin.md).

## Users

`BrewPHP\Auth\Models\User` is a regular [model](models.md) stored in the
`auth_user` table.

| Field | Type | Notes |
|---|---|---|
| `username` | `CharField(150)`, unique | Letters, digits and `@ . + - _` only. |
| `password` | `CharField(255)` | A hash, never the password. Not editable in forms. |
| `email` | `EmailField`, optional | |
| `first_name`, `last_name` | `CharField(150)`, optional | |
| `is_active` | `BooleanField`, default `true` | Inactive users cannot log in and have no permissions. Unset it instead of deleting accounts. |
| `is_staff` | `BooleanField`, default `false` | May log into the [admin](admin.md) (when active). |
| `is_superuser` | `BooleanField`, default `false` | Has every permission without being given any. |
| `last_login` | `DateTimeField`, nullable | Set by `Auth::login()`. |
| `date_joined` | `DateTimeField` | Set when the user is created. |

### Creating users

```php
use BrewPHP\Auth\Models\User;

$ann = User::createUser('ann', 'ann@example.com', 'a long passphrase');
$bob = User::createUser('bob', 'bob@example.com', 'another passphrase', ['first_name' => 'Bob']);
$root = User::createSuperuser('root', 'root@example.com', 'a strong passphrase');
$bot = User::createUser('importer');   // no password: cannot log in with one
```

`createUser()` hashes the password, lower-cases the domain of the email
address, validates the model (so a taken or invalid username raises a
`ValidationError`) and saves. `createSuperuser()` also sets `is_staff` and
`is_superuser`. Neither checks the password against the
[password validators](#password-validation); call
`Passwords::validate()` first when the password comes from a user.

!!! warning "Never assign the password field"
    `$user->password = 'secret'` would store the password in clear text.
    Always use `setPassword()`.

### Methods

| Method | Purpose |
|---|---|
| `setPassword($password)` | Hash and set a password (call `save()` afterwards). |
| `checkPassword($password)` | Whether the password is right. Upgrades an outdated hash ([rehash](#password-hashing)). |
| `setUnusablePassword()` / `hasUsablePassword()` | An account that cannot log in with a password. |
| `isAuthenticated()` / `isAnonymous()` | Always `true` / `false` for a `User`. |
| `isActive()`, `isStaff()`, `isSuperuser()` | The flags. |
| `getUsername()`, `getFullName()`, `getShortName()` | Names. |
| `hasPerm('blog.add_post')` | Whether the user has a permission. |
| `hasPerms(['blog.add_post', 'blog.change_post'])` | Whether the user has all of them. |
| `hasModulePerms('blog')` | Whether the user has any permission in the app. |
| `getAllPermissions()` | Every permission, as `"app.codename"` strings (own and through groups). |
| `groups()` / `userPermissions()` | QuerySets of the user's groups and own permissions. |
| `addToGroup($group, ...)`, `removeFromGroup(...)`, `setGroups($groups)` | Group membership. |
| `grant('blog.add_post', ...)`, `revoke(...)`, `setPermissions($permissions)` | Permissions given directly to the user. |

```php
$ann->setPassword('a new passphrase');
$ann->save();

$ann->checkPassword('a new passphrase');   // true
$ann->hasPerm('blog.add_post');            // false
$ann->grant('blog.add_post');
$ann->hasPerm('blog.add_post');            // true
```

Active superusers have every permission; inactive users have none, whatever
they were given.

## Groups and permissions

A **permission** is a row of `auth_permission`, named `"app_label.codename"`
such as `"blog.add_post"`. A **group** (`auth_group`) bundles permissions
for many users.

### Default permissions

After every `migrate`, the auth app creates four permissions for each model
of the installed apps (like Django): `add_<model>`, `change_<model>`,
`delete_<model>` and `view_<model>`. For a `Post` model in the `blog` app:

| Permission | Name |
|---|---|
| `blog.add_post` | Can add post |
| `blog.change_post` | Can change post |
| `blog.delete_post` | Can delete post |
| `blog.view_post` | Can view post |

These permissions mean nothing on their own: your views decide what each
one allows (see [Protecting views](#protecting-views)). The
[admin](admin.md#permissions) uses them for each model's pages, and is the
easiest place to give them to users and groups.

### Your own permissions

Create them in your app's `migrated()` hook, which runs after every
`migrate` (see [After migrating](migrations.md#after-migrating)):

```php title="apps/blog/App.php"
use BrewPHP\Auth\Models\Permission;
use BrewPHP\Core\AppConfig;
use BrewPHP\Core\Application;

final class App extends AppConfig
{
    public function migrated(Application $app): void
    {
        Permission::objects()->getOrCreate(
            ['app_label' => 'blog', 'codename' => 'publish_post'],
            ['name' => 'Can publish posts'],
        );
    }
}
```

### Groups

```php
use BrewPHP\Auth\Models\Group;
use BrewPHP\Auth\Models\Permission;

$editors = Group::create(['name' => 'Editors']);
$editors->grant('blog.add_post', 'blog.change_post');
$editors->revoke('blog.change_post');
$editors->permissions();        // QuerySet of Permission
$editors->users();              // QuerySet of User

$ann->addToGroup($editors);
$ann->hasPerm('blog.add_post'); // true, through the group

Permission::resolve('blog.add_post');   // the Permission object
```

`grant()` and `revoke()` accept `Permission` objects or `"app.codename"`
strings; an unknown name raises `ObjectDoesNotExist` ("Did you run
migrate?"). `setPermissions()` and `setGroups()` replace the whole list.

!!! note "Link tables instead of ManyToManyField"
    BrewPHP has no `ManyToManyField` yet. Memberships and permissions are
    stored through small link models: `UserGroup` (`auth_user_groups`),
    `UserPermission` (`auth_user_permissions`) and `GroupPermission`
    (`auth_group_permissions`). The methods above hide them.

## The current user

With `AuthenticationMiddleware`, `$request->user()` returns the logged-in
`User`, or a `BrewPHP\Auth\AnonymousUser` for visitors. Both implement
`BrewPHP\Auth\Authenticatable`, so the same calls work on either:

```php
$user = $request->user();

if ($user->isAuthenticated()) {
    // $user is a BrewPHP\Auth\Models\User
}
$user->hasPerm('blog.add_post');   // always false for an AnonymousUser
```

The user is loaded from the session on first use. If the user was
deactivated or deleted, or their password changed since this session logged
in, the visitor is anonymous again.

### In templates

The `auth` context processor adds `user` and `perms`:

```html+django
{% if user.is_authenticated %}
    <p>Hello, {{ user.username }}.</p>
    <form method="post" action="{% url 'logout' %}">
        {% csrf_token %}
        <button type="submit">Log out</button>
    </form>
{% else %}
    <a href="{% url 'login' %}">Log in</a>
{% endif %}

{% if perms.blog %}
    <a href="{% url 'blog:dashboard' %}">Blog</a>            {# any permission in the app #}
{% endif %}
{% if perms.blog.add_post %}
    <a href="{% url 'blog:new' %}">Write a post</a>          {# one permission #}
{% endif %}
```

Hiding a link is not protection: always check the permission in the view
as well.

## Logging in and out

The `BrewPHP\Auth\Auth` class does the work, like Django's `authenticate()`,
`login()` and `logout()`:

```php
use BrewPHP\Auth\Auth;
use BrewPHP\Http\Request;

function api_login(Request $request): \BrewPHP\Http\Response
{
    $user = Auth::authenticate($request, [
        'username' => (string) $request->post('username'),
        'password' => (string) $request->post('password'),
    ]);
    if ($user === null) {
        return json(['error' => 'Invalid credentials'], 400);
    }
    Auth::login($request, $user);

    return json(['ok' => true]);
}
```

| Method | Purpose |
|---|---|
| `Auth::authenticate($request, $credentials)` | Asks each [backend](#authentication-backends) in turn; returns the first `User` found, or `null`. Inactive users are never returned. |
| `Auth::login($request, $user)` | Logs the user in for this session: new session id, new CSRF token, user id and a password fingerprint stored in the session, `last_login` updated. |
| `Auth::logout($request)` | Empties the session (new id) and makes the request anonymous. |
| `Auth::updateSessionAuthHash($request, $user)` | Keeps the current session logged in after the user changed their own password. |

Session data from before the login is kept (a cart survives), unless
another user was logged in in the same session: then the session starts
empty.

### Password changes log out other sessions

The session stores a fingerprint of the user's password hash. When the
password changes, every session that logged in with the old one becomes
anonymous on its next request. After a user changes their **own** password,
call `Auth::updateSessionAuthHash()` so the current session stays logged in
(the built-in password change page does it):

```php
$user->setPassword($new);
$user->save();
Auth::updateSessionAuthHash($request, $user);
```

## Protecting views

### LoginRequired

`BrewPHP\Auth\Middleware\LoginRequired` sends visitors who are not logged
in to `LOGIN_URL`, with the page they wanted in `?next=`:

```php title="apps/blog/urls.php"
use BrewPHP\Auth\Middleware\LoginRequired;

use function Apps\Blog\dashboard;

return [
    path('dashboard/', dashboard(...), name: 'dashboard', middleware: [LoginRequired::class]),

    group('account/', [
        path('', account(...), name: 'account'),
        path('settings/', settings_page(...), name: 'settings'),
    ], middleware: [LoginRequired::class]),
];
```

A visit to `/dashboard/` redirects to `/accounts/login/?next=%2Fdashboard%2F`;
after logging in, the user lands on `/dashboard/`.

### PermissionRequired

`BrewPHP\Auth\Middleware\PermissionRequired` takes one or more permissions;
the user needs all of them. Visitors who are not logged in go to the login
page; logged-in users without the permission get a `403`:

```php
use BrewPHP\Auth\Middleware\PermissionRequired;

path('posts/new/', new_post(...), name: 'new', middleware: [new PermissionRequired('blog.add_post')]),
path('posts/<int:id>/publish/', publish(...), name: 'publish', middleware: [
    new PermissionRequired('blog.change_post', 'blog.publish_post'),
]),
```

### Checking in the view

For rules that depend on the object (only the author may edit a post),
check in the view:

```php
use BrewPHP\Auth\Middleware\LoginRequired;
use BrewPHP\Http\PermissionDenied;
use BrewPHP\Http\Request;

function edit_post(Request $request, int $id): \BrewPHP\Http\Response
{
    $user = $request->user();
    if (!$user->isAuthenticated()) {
        return LoginRequired::redirectToLogin($request);
    }
    $post = get_object_or_404(Post::class, pk: $id);
    if ($post->author_id !== $user->pk && !$user->hasPerm('blog.change_post')) {
        throw new PermissionDenied('Only the author can edit this post.');
    }

    // ...
}
```

`LoginRequired::redirectToLogin($request)` builds the same redirect as the
middleware.

## Built-in views

`BrewPHP\Auth\Urls::patterns()` returns four routes:

| URL (under `accounts/`) | Name | View | Template |
|---|---|---|---|
| `login/` | `login` | `LoginView` | `registration/login.html` |
| `logout/` | `logout` | `LogoutView` | `registration/logged_out.html` |
| `password_change/` | `password_change` | `PasswordChangeView` | `registration/password_change_form.html` |
| `password_change/done/` | `password_change_done` | `PasswordChangeDoneView` | `registration/password_change_done.html` |

Include them **without** a namespace: the views and templates use the plain
names (`{% url 'login' %}`, `password_change_done`). `php manage.php routes`
lists them. Every page is sent with `Cache-Control: no-cache, no-store,
must-revalidate, private`.

**LoginView** shows `AuthenticationForm` (username and password) and logs
the user in. It then redirects to `next` (from the form or the query
string) when that URL is on this site, else to `LOGIN_REDIRECT_URL`.
Failed logins all get the same message, whether the username exists or not:

```text
Please enter a correct username and password. Note that both fields may be case-sensitive.
```

**LogoutView** logs out, then redirects to `next`, or to
`LOGOUT_REDIRECT_URL`, or shows `registration/logged_out.html`.

**PasswordChangeView** asks for the old password and the new one twice,
checks it with the [password validators](#password-validation), keeps the
current session logged in and logs out the others. Visitors who are not
logged in are sent to the login page. It then redirects to
`password_change_done`.

### Logout needs a POST

`LogoutView` only accepts `POST` (with the CSRF token); a `GET` gets a
`405`. A logout link would let any page that embeds it (as an image, say)
log your users out. Use a small form, styled like a link if you like:

```html+django
<form method="post" action="{% url 'logout' %}">
    {% csrf_token %}
    <button type="submit">Log out</button>
</form>
```

### Settings

| Setting | Default | Description |
|---|---|---|
| `LOGIN_URL` | `'/accounts/login/'` | Where `LoginRequired` sends visitors. A path, a full URL or a route name (the skeleton uses `'login'`). |
| `LOGIN_REDIRECT_URL` | `'/'` | Where `LoginView` goes after login when there is no safe `next`. A path or a route name. |
| `LOGOUT_REDIRECT_URL` | `null` | Where `LogoutView` goes. `null` shows `registration/logged_out.html`. |

### Overriding the templates

The built-in templates extend `registration/base.html`, a small standalone
page. Create files with the same names in your project's `templates/`
folder to replace them, for example to use your own layout:

```html+django title="templates/registration/login.html"
{% extends "base.html" %}

{% block content %}
    <h1>Log in</h1>
    <form method="post" novalidate>
        {% csrf_token %}
        {{ form }}
        {% if next %}<input type="hidden" name="next" value="{{ next }}">{% endif %}
        <button type="submit">Log in</button>
    </form>
    <p><a href="{% url 'pages:home' %}">Back to the site</a></p>
{% endblock %}
```

The templates receive `title` and, on the form pages, `form`; the login
page also gets `next` (empty unless it is a safe URL). Keep
`{% csrf_token %}` in every form, and the `next` field on the login page.

### Configuring the views

The views are [class-based views](views.md#class-based-views), so
`asView()` changes their properties:

```php title="config/urls.php"
use BrewPHP\Auth\Views\LoginView;
use BrewPHP\Auth\Views\PasswordChangeView;

return [
    path('login/', LoginView::asView([
        'templateName' => 'accounts/login.html',
        'redirectAuthenticatedUser' => true,
        'extraContext' => ['site_name' => 'Brew Club'],
    ]), name: 'login'),
    path('password/', PasswordChangeView::asView(['successUrl' => 'pages:home']), name: 'password_change'),
    // ...
];
```

| View | Properties |
|---|---|
| all | `templateName`, `extraContext` (extra template variables) |
| `LoginView` | `formClass` (an `AuthenticationForm` subclass), `redirectAuthenticatedUser` (send logged-in users straight to the next page) |
| `PasswordChangeView` | `successUrl` (a path or a route name) |

### The forms

The forms live in `BrewPHP\Auth\Forms` and can be used in your own views:

| Form | Fields | Notes |
|---|---|---|
| `AuthenticationForm` | `username`, `password` | Pass `request: $request`; `getUser()` returns the user once valid. Applies [login throttling](#login-throttling). |
| `PasswordChangeForm` | `old_password`, `new_password1`, `new_password2` | Pass `user: $user`; `save()` sets the password. |
| `SetPasswordForm` | `new_password1`, `new_password2` | Set a password without the old one (for administrators). |
| `UserCreationForm` | `username`, `password1`, `password2` | A [ModelForm](forms.md#modelform) for sign-up pages; only the username comes from the model. |

A sign-up view with `UserCreationForm`:

```php
use BrewPHP\Auth\Auth;
use BrewPHP\Auth\Forms\UserCreationForm;
use BrewPHP\Http\Request;
use BrewPHP\Sessions\Messages;

function sign_up(Request $request): \BrewPHP\Http\Response
{
    $form = UserCreationForm::fromRequest($request);
    if ($request->isMethod('POST') && $form->isValid()) {
        $user = $form->save();
        Auth::login($request, $user);
        Messages::success($request, 'Welcome!');

        return redirect('pages:home');
    }

    return render('registration/sign_up.html', ['form' => $form]);
}
```

## Password hashing

Passwords are hashed with PHP's `password_hash()` and stored as
`<algorithm>$<hash>`:

| `PASSWORD_HASHER` | Stored as | Notes |
|---|---|---|
| `null` (default) | `argon2id$$argon2id$v=19$...` | Argon2id when PHP supports it, else bcrypt-SHA256. |
| `'argon2id'` | `argon2id$$argon2id$v=19$...` | Fails with `ImproperlyConfigured` when PHP has no Argon2. |
| `'bcrypt'` | `bcrypt_sha256$$2y$12$...` | bcrypt of the SHA-256 of the password, so passwords longer than bcrypt's 72-byte limit are not cut. |

`PASSWORD_HASHER_OPTIONS` passes options to `password_hash()` (integers
only), for example `['cost' => 12]` for bcrypt or
`['memory_cost' => 65536, 'time_cost' => 4, 'threads' => 1]` for Argon2id.
Leave it empty for PHP's defaults.

When you change the algorithm or its options, existing hashes keep
working. Each one is replaced by a new hash the next time its user logs in
with the right password.

The `BrewPHP\Auth\Passwords` class is available for your own code:

```php
use BrewPHP\Auth\Passwords;

$hash = Passwords::make('correct horse battery staple');
Passwords::check('correct horse battery staple', $hash);   // true, in constant time
Passwords::needsRehash($hash);                             // false with the current settings
```

## Password validation

`AUTH_PASSWORD_VALIDATORS` lists the rules for new passwords. The list is
empty by default; the skeleton turns on all four built-in rules:

```php title="config/settings.php"
use BrewPHP\Auth\Validators;

'AUTH_PASSWORD_VALIDATORS' => [
    new Validators\UserAttributeSimilarityValidator(),
    new Validators\MinimumLengthValidator(minLength: 10),
    new Validators\CommonPasswordValidator(),
    new Validators\NumericPasswordValidator(),
],
```

Entries are validator objects (to pass options) or class names (for the
defaults), such as `Validators\NumericPasswordValidator::class`.

| Validator | Rule | Options |
|---|---|---|
| `MinimumLengthValidator` | At least `minLength` characters. | `minLength` (`8`) |
| `CommonPasswordValidator` | Not a common password (compared in lower case). | `file`: your own list, one password per line |
| `NumericPasswordValidator` | Not only digits. | |
| `UserAttributeSimilarityValidator` | Not too similar to the username, names or email. | `attributes`, `maxSimilarity` (`0.7`, from `0.1` strict to `1.0`) |

The rules apply to the password change page, `UserCreationForm`,
`SetPasswordForm`, `createsuperuser` and `changepassword`. Their help texts
are shown under the new password field. Every failed rule is reported:

```text
This password is too common.
This password is entirely numeric.
```

Check a password yourself with `Passwords::validate($password, $user)`,
which throws a `ValidationError` listing the failed rules.

### Your own rule

Implement `BrewPHP\Auth\Validators\PasswordValidator`:

```php title="apps/accounts/NoSeasonValidator.php"
<?php

declare(strict_types=1);

namespace Apps\Accounts;

use BrewPHP\Auth\Authenticatable;
use BrewPHP\Auth\Validators\PasswordValidator;
use BrewPHP\Core\ValidationError;

final class NoSeasonValidator implements PasswordValidator
{
    public function validate(string $password, ?Authenticatable $user = null): void
    {
        if (preg_match('/(spring|summer|autumn|winter)\d*/i', $password) === 1) {
            throw new ValidationError('Your password cannot contain a season.', 'password_season');
        }
    }

    public function helpText(): string
    {
        return 'Your password can’t contain a season.';
    }
}
```

## Login throttling

`AuthenticationForm` counts login attempts per username **and** IP address.
Each attempt is counted before the password is checked, in one locked
step, so parallel requests cannot slip past the limit. After
`AUTH_LOGIN_ATTEMPTS` attempts without a successful login it refuses to
check passwords for that pair until `AUTH_LOGIN_LOCKOUT` seconds after the
first one:

```text
Too many failed login attempts. Try again in 15 minute(s).
```

| Setting | Default | Description |
|---|---|---|
| `AUTH_LOGIN_ATTEMPTS` | `5` | Attempts allowed in the window. `0` turns throttling off. |
| `AUTH_LOGIN_LOCKOUT` | `900` | Length of the window, in seconds. |

A successful login clears the counter. Attempts are counted under the
stored username when one matches, so spelling variants that the database
treats as equal (such as `Ann` and `ann`) share one counter. The counters
are small files in `storage/cache/ratelimit/`, kept by the
[RateLimiter](security.md#rate-limiting);
on several web servers, each one counts on its own. `Auth::authenticate()`
called directly is not throttled.

## Authentication backends

A backend checks credentials and loads users, like Django's authentication
backends. The default `ModelBackend` checks a username and a password
against `auth_user`. It refuses inactive users, and a wrong username takes
as long as a wrong password, so response times do not reveal which
usernames exist.

Write your own by implementing `BrewPHP\Auth\Backends\AuthBackend`. This
one lets users log in with their email address:

```php title="apps/accounts/EmailBackend.php"
<?php

declare(strict_types=1);

namespace Apps\Accounts;

use BrewPHP\Auth\Backends\AuthBackend;
use BrewPHP\Auth\Models\User;
use BrewPHP\Auth\Passwords;
use BrewPHP\Http\Request;

final class EmailBackend implements AuthBackend
{
    public function authenticate(?Request $request, array $credentials): ?User
    {
        $email = $credentials['username'] ?? null;
        $password = $credentials['password'] ?? null;
        if (!is_string($email) || !is_string($password) || $email === '' || $password === '') {
            return null;
        }
        $user = User::objects()->filter(email__iexact: $email)->first();
        if (!$user instanceof User) {
            Passwords::runDummyHash($password);   // same timing as a real check

            return null;
        }

        return $user->checkPassword($password) && $user->isActive() ? $user : null;
    }

    public function getUser(int|string $id): ?User
    {
        $user = User::objects()->filter(pk: $id)->first();

        return $user instanceof User && $user->isActive() ? $user : null;
    }
}
```

```php title="config/settings.php"
use BrewPHP\Auth\Backends\ModelBackend;

'AUTHENTICATION_BACKENDS' => [
    ModelBackend::class,
    Apps\Accounts\EmailBackend::class,
],
```

`Auth::authenticate()` tries the backends in order and the first user
found wins. The backend's class name is kept in the session, and only
backends listed in the setting are ever used to restore a login. Backends
are created by the container. The login form sends its fields as
`username` and `password`.

## Commands

### createsuperuser

```bash
php manage.php createsuperuser
php manage.php createsuperuser --username=admin --email=admin@example.com
```

It asks for the missing values and the password (typed twice, hidden where
the terminal allows it),
checks the password against `AUTH_PASSWORD_VALIDATORS` and offers to
bypass a failed rule. A password is never accepted as a command-line
argument: it would end up in the shell history and the process list.

In scripts and deployments, use `-n` and pass the password in the
`BREWPHP_SUPERUSER_PASSWORD` environment variable:

```bash
BREWPHP_SUPERUSER_PASSWORD='a long admin passphrase' \
    php manage.php createsuperuser -n --username=admin --email=admin@example.com
```

Without the variable, the account gets no usable password:

```text
✔ Superuser created successfully.
! The account has no usable password. Set one with: php manage.php changepassword ops
```

### changepassword

```bash
php manage.php changepassword ann
```

```text
Changing password for user "ann"
Password:
Password (again):
✔ Password changed successfully for user "ann".
```

It always asks (it cannot run with `-n`) and checks the password against
`AUTH_PASSWORD_VALIDATORS`. Changing the password logs out every session
of the user.

## Limitations

v0.5 covers the common needs. Not there yet:

- **No custom user model.** There is no `AUTH_USER_MODEL` setting. Keep
  extra user data in your own model with a `OneToOneField` to `User`
  (a profile).
- **No password reset by email.** BrewPHP has no mail support yet. An
  administrator can set a new password with `changepassword`.
- **No `ManyToManyField`.** Groups and permissions use [link
  tables](#groups).
