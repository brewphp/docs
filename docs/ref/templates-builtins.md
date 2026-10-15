# Built-in template tags and filters

Every tag and filter of the [template language](../topics/templates.md)
that ships with BrewPHP. To add your own, see
[Custom tags and filters](../topics/templates.md#custom-tags-and-filters).

## Built-in tags

### autoescape

Turns auto-escaping `on` or `off` for the template code between the tags.
Included templates keep their own escaping.

```django
{% autoescape off %}{{ trusted_html }}{% endautoescape %}
```

### block

Defines a block that child templates can replace. See
[Template inheritance](../topics/templates.md#template-inheritance).
`{{ block.super }}` prints the parent's content of the block.

```django
{% block sidebar %}{{ block.super }}<p>More links</p>{% endblock sidebar %}
```

### comment

Ignores everything up to `{% endcomment %}`, tags included. An optional
note may follow the tag name.

```django
{% comment "Disabled until the API is ready" %}{% include "widgets/weather.html" %}{% endcomment %}
```

### csrf_token

Prints a hidden `csrfmiddlewaretoken` form field holding the `csrf_token`
context variable, or nothing when there is no token. `csrf_token` is
available in every template rendered during a request; a new masked token
is printed on each page. Put the tag in every form that posts to your own
site. See [CSRF protection](../topics/security.md#csrf-protection).

```django
<form method="post">{% csrf_token %}...</form>
```

### cycle

Prints the next of its values each time it runs; inside a loop it
alternates. `as name` also stores the current value in a variable, and
`silent` stores it without printing. Later, `{% cycle name %}` moves the
named cycle on.

```django
{% for row in rows %}
    <tr class="{% cycle 'odd' 'even' %}">...</tr>
{% endfor %}

{% for row in rows %}
    {% cycle 'odd' 'even' as rowclass silent %}
    <tr class="{{ rowclass }}">...</tr>
{% endfor %}
```

### extends

Makes this template a child of another. It must be the first tag. The
argument is a string or a variable. See
[Template inheritance](../topics/templates.md#template-inheritance).

```django
{% extends "base.html" %}
```

### filter

Applies filters to the output of its content.

```django
{% filter lower|truncatewords:10 %}This TEXT is lowercased and cut after ten words{% endfilter %}
```

### firstof

Prints the first of its arguments that is true (escaped), or nothing.
`as name` stores it instead.

```django
{% firstof user.nickname user.name "Anonymous" %}
```

### for

Loops over a list, an array, a QuerySet, a page or any iterable.
`{% empty %}` is shown when there is nothing to loop over; `reversed`
loops backwards.

```django
{% for post in posts reversed %}
    {{ forloop.counter }}. {{ post.title }}
{% empty %}
    No posts.
{% endfor %}
```

With two names, an associative array gives key and value, and a list of
pairs is unpacked:

```django
{% for code, label in countries %}<option value="{{ code }}">{{ label }}</option>{% endfor %}
{% for name, score in results.items %}{{ name }}: {{ score }}{% endfor %}
```

The `forloop` variable:

| Attribute | Value |
|---|---|
| `forloop.counter` | current iteration, from 1 |
| `forloop.counter0` | current iteration, from 0 |
| `forloop.revcounter` | iterations left, ending at 1 |
| `forloop.revcounter0` | iterations left, ending at 0 |
| `forloop.first` | true on the first iteration |
| `forloop.last` | true on the last iteration |
| `forloop.length` | number of items |
| `forloop.parentloop` | the `forloop` of the enclosing loop |

### if

Shows its content when a condition is true, with optional `{% elif %}`
and `{% else %}` branches.

```django
{% if not posts %}
    Nothing yet.
{% elif posts|length == 1 and posts.0.pinned %}
    One pinned post.
{% elif user.role in staff_roles or (user.is_admin and not user.locked) %}
    Staff view.
{% else %}
    {{ posts|length }} posts.
{% endif %}
```

| Operator | Meaning |
|---|---|
| `and`, `or`, `not` | boolean logic (`or` binds loosest, then `and`, then `not`) |
| `==`, `!=` | equal / not equal: numbers by value (`1 == 1.0`), other values must have the same type; models are equal when they have the same class and primary key |
| `<`, `>`, `<=`, `>=` | compare numbers, strings or dates (other combinations are false) |
| `in`, `not in` | substring of a string, item of a list or iterable, key of an associative array |
| `is`, `is not` | identity: `{% if value is None %}` |
| `( )` | grouping |

Truth follows Python: `null`, `false`, `0`, `0.0`, `''`, `[]` and empty
countables (an empty QuerySet or page) are false; everything else,
including `"0"`, is true. Missing variables are false (even with
`TEMPLATES.STRICT`).

### include

Renders another template with the current context. `with` adds variables;
`only` passes those variables and nothing else. The name can be a
variable or a list of names (the first that exists is used).

```django
{% include "partials/pagination.html" %}
{% include "partials/card.html" with item=post compact=True only %}
```

### load

Does nothing. Template libraries are always available in BrewPHP; the tag
is accepted so templates copied from Django still work.

### now

Prints the current date and time with a PHP `date()` format, or stores it
with `as`.

```django
&copy; {% now "Y" %} My company
{% now "Y-m-d" as today %}
```

### spaceless

Removes whitespace between HTML tags (not inside text).

```django
{% spaceless %}
    <p>
        <a href="/">Home</a>
    </p>
{% endspaceless %}
{# <p><a href="/">Home</a></p> #}
```

### static

Prefixes a path with the `STATIC_URL` setting.

```django
<link rel="stylesheet" href="{% static 'css/site.css' %}">
{% static 'img/logo.svg' as logo %}
```

### url

The path of a [named route](../topics/routing.md#naming-routes-and-reverse), like
`reverse()`. Give route parameters in pattern order, or by name; do not
mix both.

```django
<a href="{% url 'blog:post-detail' post.id %}">{{ post.title }}</a>
<a href="{% url 'blog:post-detail' id=post.id %}">{{ post.title }}</a>
{% url 'blog:archive' year month as archive_url %}
```

An unknown route raises an error, except in the `as name` form, which
stores an empty string instead.

### verbatim

Prints its content without interpreting tags or variables (for
JavaScript templates that also use `{{ }}`). `{% verbatim name %}` ends at
`{% endverbatim name %}`.

```django
{% verbatim %}<script type="text/x-template">{{ message }}</script>{% endverbatim %}
```

### with

Stores values in variables until `{% endwith %}`.

```django
{% with total=cart.items|length shipping=cart.shipping_cost %}
    {{ total }} item{{ total|pluralize }}, shipping {{ shipping|floatformat:2 }}
{% endwith %}
{% with business.employees.count as count %}{{ count }}{% endwith %}
```

## Built-in filters

Filters receive the value on the left of `|` and at most one argument.

### add

Adds the argument: numbers (also numeric strings) are summed, lists
merged, anything else concatenated as text.

```django
{{ page.number|add:1 }}   {{ first_list|add:second_list }}
```

### capfirst

Uppercases the first character. `{{ "élan"|capfirst }}` → `Élan`.

### cut

Removes every occurrence of the argument. `{{ "a b c"|cut:" " }}` → `abc`.

### date

Formats a date with PHP's [`date()` characters](https://www.php.net/manual/en/datetime.format.php).
Accepts `DateTimeInterface` objects, timestamps and date strings; anything
else gives an empty string. Default format: `M j, Y` (`Mar 5, 2026`).

```django
{{ post.published_at|date:"l, F jS Y \a\t H:i" }}
```

### default

The argument when the value is false (`null`, `''`, `0`, empty list...).
`{{ post.subtitle|default:"No subtitle" }}`

### default_if_none

The argument only when the value is `null`.

### dictsort

Sorts a list of arrays or objects by a key (dotted paths work).
`dictsortreversed` sorts in reverse order.

```django
{% for user in users|dictsort:"profile.age" %}...{% endfor %}
```

### divisibleby

True when the value is divisible by the argument.
`{% if forloop.counter|divisibleby:3 %}<br>{% endif %}`

### escape

Escapes HTML now. A value already escaped (or safe) is not escaped again.

### first

The first item of a list, iterable or string; empty string when there is none.

### floatformat

Rounds a number.

| Template | Value `34.23234` | Value `34.0` |
|---|---|---|
| `{{ v\|floatformat }}` | `34.2` | `34` |
| `{{ v\|floatformat:2 }}` | `34.23` | `34.00` |
| `{{ v\|floatformat:"-2" }}` | `34.23` | `34` |
| `{{ v\|floatformat:"2g" }}` (thousands separators) | `34.23` | `34.00` |

A positive argument always shows that many decimals; a negative one only
when needed.

### join

Joins a list with the argument (default `", "`). With auto-escaping on,
items and separator are escaped. `{{ tags|join:" · " }}`

### json_script

Prints the value as JSON inside `<script type="application/json">`,
optionally with an `id`. `<`, `>` and `&` are encoded, so the data cannot
end the script element. See
[Data for JavaScript](../topics/templates.md#data-for-javascript).

```django
{{ settings|json_script:"app-settings" }}
```

### last

The last item of a list, iterable or string.

### length

The number of items (arrays, QuerySets, countables, iterables) or
characters (strings). `null` gives 0.

### linebreaks

Turns plain text into paragraphs: a blank line starts a new `<p>`, a
single newline becomes `<br>`. The text is escaped first (unless it is
safe or auto-escaping is off).

### linebreaksbr

Turns every newline into `<br>`, escaping the text first.

### lower

Lowercases the text.

### pluralize

A plural suffix when the value (a number, or a list's length) is not 1.
The argument is the suffix (default `s`) or `singular,plural`.

```django
{{ count }} comment{{ count|pluralize }}
{{ count }} box{{ count|pluralize:"es" }}
{{ count }} cherr{{ count|pluralize:"y,ies" }}
```

### safe

Marks the value as safe HTML, so it is not escaped. Only for HTML you
trust.

### slice

Part of a list or string, with Python slice syntax.
`{{ items|slice:":3" }}` (first three), `"2:"`, `"-2:"`, `"1:-1"`, `"::2"`.

### slugify

`"Hello, World!"` → `hello-world`. Accented letters are transliterated
when the iconv extension is available.

### striptags

Removes HTML tags. The result is not HTML-safe: it is still escaped when
printed.

### time

Formats the time part of a date (default `H:i`). Same format characters as
[date](#date). `{{ meeting.starts_at|time:"g:i A" }}` → `2:30 PM`.

### timesince

The time elapsed since a date, up to two units: `3 days, 6 hours`. The
argument is the date to compare with (default: now).
`{{ comment.created_at|timesince }} ago`

### title

Uppercases the first letter of each word.

### truncatechars

Cuts the text to that many characters, the last one being `…`.
`{{ "Hello world"|truncatechars:5 }}` → `Hell…`

### truncatewords

Keeps that many words and adds ` …` when the text was longer.

### upper

Uppercases the text.

### urlencode

Percent-encodes the value for a URL. Characters in the argument are kept
(default `/`); use `urlencode:""` to encode `/` too.

```django
<a href="/search/?q={{ query|urlencode }}">
```

### wordcount

The number of words.

### yesno

Maps true, false and `null` to words. The argument is
`"yes_word,no_word,null_word"` (default `yes,no,maybe`); without a third
word, `null` uses the second.

```django
{{ post.published|yesno:"Published,Draft" }}
```
