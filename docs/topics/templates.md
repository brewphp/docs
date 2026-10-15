# Templates

BrewPHP has its own template language, modelled on Django's. A template is
a text file (usually HTML) with **variables** `{{ ... }}`, **tags**
`{% ... %}` and **comments** `{# ... #}`:

```html+django title="apps/blog/templates/blog/post_list.html"
{% extends "base.html" %}

{% block title %}Blog{% endblock %}

{% block content %}
    <h1>Latest posts</h1>
    {% for post in posts %}
        <article>
            <h2><a href="{% url 'blog:post-detail' post.id %}">{{ post.title }}</a></h2>
            <p>By {{ post.author.name }}, {{ post.published_at|date:"M j, Y" }}</p>
            {{ post.body|truncatewords:30|linebreaks }}
        </article>
    {% empty %}
        <p>No posts yet.</p>
    {% endfor %}
{% endblock %}
```

Render it from a view with `render()`:

```php title="apps/blog/views.php"
function post_list(Request $request): Response
{
    return render('blog/post_list.html', [
        'posts' => Post::objects()->filter(published: true)->orderBy('-published_at'),
    ]);
}
```

The template language is designed for designers as much as for
programmers: it reads data and presents it, but it cannot run arbitrary
PHP. Everything printed is HTML-escaped unless you say otherwise.

## Template files and engines

Templates are found by name in the template folders, first match wins
(see [Project structure](../getting-started/project-structure.md#templates)):
the `TEMPLATES` `DIRS` folders, then each app's `templates/` folder, then
the framework's own templates. Name templates after their app
(`blog/post_list.html`) to avoid clashes between apps.

The **file extension** picks the engine:

| Extension | Engine |
|---|---|
| `.php` | plain PHP templates (context values become variables, escape with `e()`) |
| anything else: `.html`, `.txt`, `.xml`, `.md`... | the BrewPHP template language |

So `render('blog/post.php')` keeps working exactly as in v0.1, and you can
move pages to the template language one at a time.

The `Templates` object (`app()->templates()`) is the single entry point:

```php
$templates = app()->templates();

$html = $templates->render('emails/welcome.txt', ['user' => $user]);
$html = $templates->render(['blog/custom.html', 'blog/default.html'], $context); // first that exists
$html = $templates->renderString('Hello {{ name }}!', ['name' => 'Ann']);
$templates->exists('blog/post.html');   // bool
$templates->find('blog/post.html');     // absolute path
```

`render_to_string($template, $context)` is a shortcut for the first line
that also runs the [context processors](#context-processors) when called
during a request.

## Variables

`{{ name }}` prints a variable from the context. A dot looks inside it:

```django
{{ user.name }}          {# array key, property, method or model field #}
{{ post.author.email }}  {# follow a foreign key #}
{{ items.0 }}            {# first item of a list or QuerySet #}
{{ page.has_next }}      {# calls $page->hasNext() #}
```

For `{{ a.b }}`, BrewPHP tries in this order:

1. array key `b` (or `ArrayAccess` offset);
2. public property `b` (or `$a->b` through `__get()` when `__isset()` says it exists);
3. public method `b()` that needs no arguments;
4. for a model, field or relation `b`.

A `snake_case` name also tries its `camelCase` form, so templates read like
Django's: `{{ page.paginator.num_pages }}` calls `numPages()`,
`{{ post.created_at }}` reads the field.

Some rules keep templates safe and simple:

- Names starting with `_` are not allowed (`{{ post._secret }}` is a syntax error).
- Static methods, magic methods and methods with required parameters are never called.
- Methods that change data are never called: `{{ post.delete }}` prints
  nothing and deletes nothing. These methods carry the `#[AltersData]`
  attribute (`BrewPHP\Core\AltersData`): `Model::save()`, `delete()`,
  `refresh()`, the model hooks, `QuerySet::delete()`, `update()`,
  `create()`, `getOrCreate()` and `updateOrCreate()`. Overriding such a
  method keeps the protection. Add the attribute to your own methods that
  change data.
- A context value that is a `Closure` is called (without arguments) when
  the template uses it, so you can pass expensive values lazily.

A missing variable or attribute prints an empty string. Set
`TEMPLATES.STRICT` to `true` to get a `VariableDoesNotExist` error
instead, which catches typos during development. Even in strict mode,
`{% if %}` conditions, `{% firstof %}` and the `default` and
`default_if_none` filters may test missing values.

How values are printed:

| Value | Printed as |
|---|---|
| `null` | nothing |
| `true` / `false` | `True` / `False` |
| numbers, strings, `Stringable` objects (models!) | their text |
| backed enum / pure enum | its value / its name |
| `DateTimeInterface` | `Y-m-d H:i:s` (use the `date` filter for other formats) |
| arrays, other objects | error: loop over them or pick an attribute |

Literals can be used anywhere a variable can: `"text"` or `'text'`,
numbers (`3`, `-1.5`) and `True`, `False`, `None` (or `true`, `false`,
`null`).

## Filters

Filters change a value before it is printed. Apply them with `|`, give
them one argument with `:`, and chain them:

```django
{{ name|lower }}
{{ bio|truncatewords:20 }}
{{ post.title|default:"Untitled"|upper }}
{{ value|date:date_format }}   {# the argument can be a variable #}
```

The [built-in filters](../ref/templates-builtins.md#built-in-filters) cover
text (`lower`, `truncatechars`, `slugify`...), lists (`join`, `length`,
`first`, `dictsort`...), numbers (`floatformat`, `add`), dates (`date`,
`timesince`), HTML (`linebreaks`, `striptags`, `json_script`) and escaping
(`escape`, `safe`).

## Tags

Tags add logic: loops, conditions, inheritance, URLs. Some have an end tag.

```django
{% if user.is_staff %}
    <a href="{% url 'admin:index' %}">Admin</a>
{% elif user %}
    Hello {{ user.name }}
{% else %}
    <a href="/login/">Log in</a>
{% endif %}

<ul>
{% for item in cart.items %}
    <li class="{% cycle 'odd' 'even' %}">{{ forloop.counter }}. {{ item.name }}</li>
{% empty %}
    <li>Your cart is empty.</li>
{% endfor %}
</ul>
```

`{% if %}` understands `and`, `or`, `not`, `==`, `!=`, `<`, `>`, `<=`,
`>=`, `in`, `not in`, `is`, `is not` and parentheses. Values are tested
like in Python: `null`, `false`, `0`, `''`, empty arrays and empty
countables (an empty QuerySet or page) are false; everything else,
including the string `"0"`, is true.

Inside `{% for %}`, `forloop` gives `counter` (from 1), `counter0` (from
0), `revcounter`, `revcounter0`, `first`, `last`, `length` and
`parentloop` (the `forloop` of the enclosing loop). Unpack pairs with
`{% for key, value in dict %}` and loop backwards with
`{% for x in items reversed %}`.

Every tag is described in the
[built-in tags reference](../ref/templates-builtins.md#built-in-tags).

## Comments

```django
{# a one-line comment, never printed #}

{% comment "why this is disabled" %}
    {% for x in old_stuff %}...{% endfor %}
{% endcomment %}
```

## Template inheritance

Inheritance builds every page from one layout. The **parent** template
defines blocks:

```html+django title="templates/base.html"
<!doctype html>
<html lang="en">
<head>
    <title>{% block title %}My site{% endblock %}</title>
</head>
<body>
    <nav>{% block nav %}<a href="/">Home</a>{% endblock %}</nav>
    <main>{% block content %}{% endblock %}</main>
</body>
</html>
```

A **child** template starts with `{% extends %}` and replaces the blocks it
wants. `{{ block.super }}` prints the parent's version of the block:

```html+django title="apps/blog/templates/blog/post_detail.html"
{% extends "base.html" %}

{% block title %}{{ post.title }} | {{ block.super }}{% endblock %}

{% block content %}
    <h1>{{ post.title }}</h1>
    {{ post.body|linebreaks }}
{% endblock content %}
```

The rules:

- `{% extends %}` must be the first tag of the template. Its argument is a
  string or a variable (`{% extends layout %}`).
- In a child template only the blocks count; text outside blocks is ignored.
- Inheritance can go as deep as you like (`base.html` ← `blog/base.html` ←
  `blog/post_detail.html`); the most derived block wins.
- A block name may appear once per template. `{% endblock %}` may repeat
  the name for readability: `{% endblock content %}`.
- **Overriding a template you also extend**: a project template may extend
  the app or framework template with the same name, for example
  `templates/admin/base.html` containing `{% extends "admin/base.html" %}`.
  The search for the parent then starts after the folder of the child, so
  you change one block of a template without copying the whole file.

## Including templates

`{% include %}` renders another template in place, with the current context:

```django
{% include "blog/_post_card.html" %}
{% include "blog/_post_card.html" with post=featured show_author=True %}
{% include "blog/_post_card.html" with post=featured only %}
{% include template_name %}
```

`with` adds variables; `only` gives the included template those variables
and nothing else. The name can be a variable or a list of names (the first
that exists is used).

## Automatic HTML escaping

Every variable is HTML-escaped when printed: `<`, `>`, `&`, `"` and `'`
become entities. So `{{ comment.text }}` is safe even if the comment
contains `<script>`.

To print HTML that you trust:

- `{{ article.html_body|safe }}` marks one value as safe;
- `{% autoescape off %}...{% endautoescape %}` turns escaping off for the
  template code between the tags (included templates keep their own
  escaping);
- a `BrewPHP\Templates\SafeString` value is never escaped. Build one in PHP
  with `mark_safe($html)` or `new SafeString($html)`. Filters and tags that
  produce HTML return one.

`{{ value|escape }}` escapes immediately (useful inside
`{% autoescape off %}`); a value is never escaped twice.

!!! warning "Only mark trusted HTML as safe"
    `|safe`, `mark_safe()` and `{% autoescape off %}` switch off the XSS
    protection. Never use them on data a user can control.

In PHP templates, `e()` also leaves a `SafeString` unchanged, so
`<?= e($form) ?>` prints safe HTML once.

### HtmlSafe objects

An object whose string form is already escaped HTML can implement the
`BrewPHP\Templates\HtmlSafe` interface (a `Stringable`). Auto-escaping and
`e()` then print it as it is. [Forms](forms.md), their fields and error
lists implement it, so `{{ form }}` and `{{ form.email.errors }}` print
markup while every value inside is escaped. `SafeString` implements it
too.

```php
use BrewPHP\Templates\HtmlSafe;

final class Badge implements HtmlSafe
{
    public function __construct(private readonly string $label)
    {
    }

    public function __toString(): string
    {
        return '<span class="badge">' . e($this->label) . '</span>';
    }
}
```

!!! warning
    Only implement `HtmlSafe` on classes that escape every value they put
    in their HTML.

### Data for JavaScript

Do not build JavaScript from template variables. Use `json_script` and read
the data back with `JSON.parse`:

```django
{{ chart_data|json_script:"chart-data" }}
<script>
    const data = JSON.parse(document.getElementById('chart-data').textContent);
</script>
```

`json_script` encodes `<`, `>` and `&`, so the data can never close the
`<script>` element.

### Templates cannot run PHP

Templates are compiled to PHP, but template text, string literals,
variable names and filter names never become PHP code: text and literals
are written as escaped PHP strings, and names must match
`[A-Za-z_][A-Za-z0-9_]*`. A template containing `<?php ... ?>` prints it
as text.

## Context processors

A **context processor** adds variables to every template rendered by
`render()` (or `render_to_string()`) during a request. It is any callable
that receives the `Request` and returns an array:

```php title="apps/shop/context_processors.php"
<?php

declare(strict_types=1);

namespace Apps\Shop;

use BrewPHP\Http\Request;

final class CartProcessor
{
    /**
     * @return array<string, mixed>
     */
    public function __invoke(Request $request): array
    {
        return ['cart_count' => (int) $request->cookie('cart_count', '0')];
    }
}
```

List processors in `TEMPLATES.CONTEXT_PROCESSORS`, as callables or names
of invokable classes (built by the container, so their constructors can
ask for services):

```php title="config/settings.php"
use BrewPHP\Templates\ContextProcessors;

'TEMPLATES' => [
    'DIRS' => ['templates'],
    'CONTEXT_PROCESSORS' => [
        [ContextProcessors::class, 'request'],   // adds `request`
        Apps\Shop\CartProcessor::class,
    ],
],
```

The built-in processors are methods of
`BrewPHP\Templates\ContextProcessors`:

| Processor | Adds | Needs |
|---|---|---|
| `[ContextProcessors::class, 'request']` | `request`: `{{ request.path }}`, `{{ request.method }}`. On by default. | |
| `[ContextProcessors::class, 'auth']` | `user` (the logged-in `User` or an `AnonymousUser`) and `perms` (`{% if perms.blog.add_post %}`). See [Authentication](auth.md#in-templates). | `AuthenticationMiddleware` |
| `[ContextProcessors::class, 'messages']` | `messages`, the [flash messages](sessions.md#flash-messages), removed once a template loops over them. | `SessionMiddleware` |

The skeleton turns on all three. Without the middleware they need, `auth`
and `messages` add nothing.

`csrf_token` is always added, whatever the list contains, so
`{% csrf_token %}` works in every template rendered with `render()` or
`render_to_string()` during a request. The token (and its cookie) is only
created when a template prints it. See
[CSRF protection](security.md#csrf-protection).
The variables a view passes win over those of the processors. Wrap an
expensive value in a closure (`'stats' => fn () => compute_stats()`): it
is only computed if a template uses it.

## Custom tags and filters

Register filters and tags with a `Library` in the `ready()` method of one
of your apps. Libraries are global: every template can use them, and
`{% load %}` is accepted but does nothing. A name registered later
replaces an earlier one, including built-ins.

```php title="apps/blog/App.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Core\AppConfig;
use BrewPHP\Core\Application;
use BrewPHP\Templates\Library;

final class App extends AppConfig
{
    public function ready(Application $app): void
    {
        $app->templates()->library(
            (new Library())
                ->filter('reading_time', fn (mixed $text): string => max(1, intdiv(str_word_count((string) $text), 200)) . ' min')
                ->simpleTag('greeting', fn (string $name, string $word = 'Hello'): string => "$word, $name!"),
        );
    }
}
```

```django
{{ post.body|reading_time }}
{% greeting user.name %}
{% greeting user.name word="Welcome" %}
{% greeting user.name as message %}{{ message|upper }}
```

### Filters

`filter(string $name, callable $filter, bool $isSafe = false, bool $needsAutoescape = false)`

The callable receives the value and, when the template gives one, the
argument. Values arrive as they are (a `SafeString` stays a `SafeString`;
cast it with `(string)`). `BrewPHP\Templates\Runtime` has helpers:
`Runtime::toString()`, `Runtime::escape()`, `Runtime::truthy()`.

- `isSafe: true` says the filter adds no HTML special characters, so when
  the input was a `SafeString` the result is marked safe too (like `lower`).
- `needsAutoescape: true` passes a named `autoescape` argument: filters
  that return HTML must escape their input themselves when it is `true`,
  and return a `SafeString`:

```php
use BrewPHP\Templates\Runtime;
use BrewPHP\Templates\SafeString;

$library->filter(
    'mark',
    fn (mixed $text, bool $autoescape = true): SafeString => new SafeString(
        '<mark>' . ($autoescape ? Runtime::escape($text) : Runtime::toString($text)) . '</mark>',
    ),
    needsAutoescape: true,
);
```

### Simple tags

`simpleTag(string $name, callable $function, bool $takesContext = false)`

The tag's arguments are evaluated and passed to the function: positional
ones in order, `name=value` ones as named arguments. The result is printed,
escaped unless it is a `SafeString`; with `as name` it is stored in a
variable instead. With `takesContext: true`, the function first receives
the `BrewPHP\Templates\Context`:

```php
use BrewPHP\Templates\Context;

$library->simpleTag(
    'active',
    fn (Context $context, string $path): string => $context->get('request')?->path() === $path ? 'active' : '',
    takesContext: true,
);
```

```django
<a href="/blog/" class="{% active '/blog/' %}">Blog</a>
```

### Tags with their own syntax

For a tag with an end tag or a special syntax, register a **compiler**
with `tag(string $name, callable $compiler)`. Templates are compiled to
PHP, so a compiler reads the tag's tokens and returns a node that writes
PHP code. This example repeats its content:

```php
use BrewPHP\Templates\Compiler;
use BrewPHP\Templates\Nodes\Node;
use BrewPHP\Templates\Nodes\TagNode;
use BrewPHP\Templates\Parser;
use BrewPHP\Templates\Token;

$library->tag('repeat', function (Parser $parser, Token $token): Node {
    $times = $parser->compileExpression($token->tagArguments()); // PHP code of "3" or "n|add:1"
    $body = $parser->parse(['endrepeat'], $token);                // nodes up to {% endrepeat %}
    $parser->nextToken();                                         // consume {% endrepeat %}

    return new TagNode(function (Compiler $compiler) use ($times, $body): void {
        $i = $compiler->variable('i');
        $compiler->write(sprintf('for (%s = 0; %s < (int) %s; %s++) {', $i, $i, $times, $i))
            ->indent()
            ->subcompile($body)
            ->outdent()
            ->write('}');
    });
});
```

```django
{% repeat 3 %}<span class="star">★</span>{% endrepeat %}
```

In the generated code `$c` is the `Context` and `$r` the `Runtime`. The
tools you need:

| API | Purpose |
|---|---|
| `$parser->arguments($token)` | the tag's arguments as a `TokenStream` (`next()`, `nextIfName('as')`, `expectName()`, `expectEnd()`...) |
| `$parser->filterExpression($stream)` / `compileExpression($string)` | compile a value with filters to PHP code |
| `$parser->condition($stream)` | compile an `{% if %}`-style condition to PHP code giving a bool |
| `$parser->parse(['endtag'], $token)` and `nextToken()` | parse up to an end tag |
| `$parser->autoescape` | whether auto-escaping is on at this point |
| `Compiler::string($text)`, `Compiler::literal($value)` | write values as PHP literals: **never** paste template text into code |
| `new TemplateSyntaxError('message')` | report a mistake; the template name and line are added for you |

!!! warning "Compilers write PHP"
    A tag compiler's output runs as PHP code. Only build it from what the
    parser gives you and from `Compiler::string()` / `Compiler::literal()`,
    so templates can never inject code.

Compiled templates are cached (see below). The cache key includes the
names of the registered tags, but not their code: after changing a tag
compiler, delete `storage/templates/`.

## Caching and compiled templates

The first time a template is rendered it is compiled to a PHP file in
`storage/templates/`; later renders just run that file (and OPcache keeps
it in memory). The compiled file remembers the modification time and size
of its source and is rebuilt as soon as either changes, so editing a
template needs no extra step. `TEMPLATES.CACHE = false` compiles each
template again on every request (only useful when working on custom tag
compilers). The web server needs write access to `storage/templates/`.

## Errors

Mistakes in a template raise `BrewPHP\Templates\TemplateSyntaxError` when
it is compiled: an unknown tag or filter, an unclosed block, a bad
expression. The message names the template and the line:

```text
Unknown filter "titel". (in template "blog/post_detail.html", line 12)
```

Errors raised while rendering (an unknown URL name in `{% url %}`, a
missing template in `{% include %}`, a failing filter) are wrapped in a
`TemplateError` with the same information; the original exception is
available with `getPrevious()`. HTTP exceptions such as `Http404` pass
through unchanged. With `DEBUG` on, the error page shows the template
source around the failing line.

Every error class lives in `BrewPHP\Templates`: `TemplateError` (with
`templateName`, `templateLine` and `templateFile`), its subclasses
`TemplateSyntaxError` and `VariableDoesNotExist`, and `TemplateNotFound`.

## Settings

| Setting | Default | Description |
|---|---|---|
| `TEMPLATES.DIRS` | `['templates']` | Project template folders, searched before app templates. |
| `TEMPLATES.CACHE` | `true` | Reuse compiled templates until their source changes. |
| `TEMPLATES.STRICT` | `false` | Raise `VariableDoesNotExist` for missing variables. |
| `TEMPLATES.CONTEXT_PROCESSORS` | `[[ContextProcessors::class, 'request']]` | Callables or invokable classes adding variables to every template rendered during a request (the skeleton adds `auth` and `messages`). |

`STATIC_URL` is used by `{% static %}`.
