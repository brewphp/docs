# Forms

A form describes the fields a page asks for. It turns submitted text into
clean PHP values, reports what is wrong in words a user understands, and
renders itself as escaped HTML. BrewPHP forms work like Django's
`django.forms`.

## A first form

Declare the fields in `fields()`:

```php title="apps/pages/forms.php"
<?php

declare(strict_types=1);

namespace Apps\Pages;

use BrewPHP\Forms\Fields;
use BrewPHP\Forms\Form;
use BrewPHP\Forms\Widgets;

final class ContactForm extends Form
{
    public function fields(): array
    {
        return [
            'name' => new Fields\CharField(maxLength: 100),
            'email' => new Fields\EmailField(helpText: 'We never share it.'),
            'topic' => new Fields\ChoiceField(['support' => 'Support', 'sales' => 'Sales']),
            'message' => new Fields\CharField(widget: new Widgets\Textarea(['rows' => 4])),
            'cc_myself' => new Fields\BooleanField(required: false),
        ];
    }
}
```

`forms.php` is loaded at boot like `models.php` and `views.php`, so the
class is available in every view of the app.

Use it in a view:

```php title="apps/pages/views.php"
use BrewPHP\Http\Request;
use BrewPHP\Sessions\Messages;

function contact(Request $request): \BrewPHP\Http\Response
{
    $form = ContactForm::fromRequest($request);
    if ($request->isMethod('POST') && $form->isValid()) {
        $data = $form->cleanedData();
        // send_message($data['email'], $data['message']) ...
        Messages::success($request, 'Thanks, we will answer soon.');

        return redirect('pages:contact');
    }

    return render('pages/contact.html', ['form' => $form]);
}
```

And render it in the template:

```html+django title="apps/pages/templates/pages/contact.html"
{% extends "base.html" %}

{% block content %}
    <form method="post" novalidate>
        {% csrf_token %}
        {{ form }}
        <button type="submit">Send</button>
    </form>
{% endblock %}
```

On `GET`, the form is empty. On `POST`, it is filled with what was sent;
when something is wrong, the same page is shown again with the values and
the error messages. After a successful `POST`, redirect, so reloading the
page does not send the form twice.

## Bound and unbound forms

A form with data is **bound**; a form without is **unbound** and only
shows its initial values.

```php
$form = new ContactForm();                                 // unbound
$form = new ContactForm(['name' => 'Ann', 'email' => 'ann@example.com', ...]);  // bound
$form = ContactForm::fromRequest($request);                // bound on POST, PUT and PATCH
```

`fromRequest()` reads `$request->post()` and `$request->files()`. Other
constructor arguments can be given by name:
`ContactForm::fromRequest($request, ['topic' => 'sales'], prefix: 'contact')`.

| Constructor argument | Purpose |
|---|---|
| `data` | The submitted values (`null` for an unbound form). |
| `files` | Uploaded files by field name. |
| `initial` | Values shown by an unbound form, by field name. |
| `prefix` | Prefix of the HTML names (`billing-name`), to put several forms in one `<form>`. |
| `autoId` | Format of the HTML ids, `'id_%s'` by default; `null` for none. |

## Validation

| Method | Returns |
|---|---|
| `isBound()` | Whether the form has data. |
| `isValid()` | Whether it is bound and every value is valid. Runs the validation once. |
| `cleanedData()` | The clean values: strings trimmed, numbers as `int` or `float`, dates as `DateTimeImmutable`, choices as enum cases... Only fields that passed validation are there. |
| `errors()` | The messages by field name; `__all__` holds the errors of the whole form. |
| `hasError('email')` | Whether a field has an error. |
| `addError('email', 'Already registered.')` | Add an error yourself (`null` as the field for the whole form). |
| `nonFieldErrors()` | The errors of the whole form, as an `ErrorList`. |

```php
$form = new ContactForm(['name' => 'Ann', 'email' => 'nope', 'topic' => 'evil', 'message' => 'Hi']);
$form->isValid();    // false
$form->errors();
// ['email' => ['Enter a valid email address.'],
//  'topic' => ['Select a valid choice. evil is not one of the available choices.']]
```

### Cleaning one field: clean&lt;Name&gt;()

Add a method named `clean` followed by the field name in StudlyCase
(`clean_email` → `cleanEmail()`, `cc_myself` → `cleanCcMyself()`). It
receives the value already cleaned by the field, and returns it (changed if
you like) or throws a `ValidationError`:

```php
use BrewPHP\Core\ValidationError;

public function cleanName(string $name): string
{
    if (str_contains(strtolower($name), 'spam')) {
        throw new ValidationError('No spam, please.');
    }

    return $name;
}
```

The hook only runs when the field itself is valid. Declare the parameter
type to match what the field returns, and allow `null` for optional fields
(`?string`).

### Cleaning several fields: clean()

Override `clean(array $data)` for rules that involve several fields. It
receives the values that are valid so far, and returns them (changed if
you like) or throws a `ValidationError`:

```php
public function clean(array $data): array
{
    if (($data['topic'] ?? null) === 'sales' && !str_ends_with((string) ($data['email'] ?? ''), '.com')) {
        throw new ValidationError('Sales only answers .com addresses.');
    }

    return $data;
}
```

A message thrown by `clean()` goes to the whole form (`__all__`). To attach
messages to fields, throw an array:
`throw new ValidationError(['email' => 'Use your work address.'])`.

## Rendering forms

Forms and fields print escaped HTML; in templates they are printed as they
are, without double escaping. You still write the `<form>` element, the
`{% csrf_token %}` and the button yourself.

### The whole form

`{{ form }}` prints the errors of the whole form, then every field in a
`<div>` with its label, help text, errors and widget, then the hidden
fields. For the contact form above, unbound:

```html
<div><label for="id_name">Name:</label><input type="text" name="name" maxlength="100" id="id_name" required></div>
<div><label for="id_email">Email:</label><div class="helptext" id="id_email_helptext">We never share it.</div><input type="email" name="email" maxlength="320" id="id_email" required aria-describedby="id_email_helptext"></div>
<div><label for="id_topic">Topic:</label><select name="topic" id="id_topic" required>
<option value="">---------</option>
<option value="support">Support</option>
<option value="sales">Sales</option>
</select></div>
<div><label for="id_message">Message:</label><textarea cols="40" rows="4" name="message" id="id_message" required>
</textarea></div>
<div><label for="id_cc_myself">Cc myself:</label><input type="checkbox" name="cc_myself" id="id_cc_myself"></div>
```

With errors, each field gets `<ul class="errorlist">`,
`aria-invalid="true"` and an `aria-describedby` pointing at its messages:

```html
<div><label for="id_email">Email:</label><div class="helptext" id="id_email_helptext">We never share it.</div><ul class="errorlist" id="id_email_error"><li>Enter a valid email address.</li></ul><input type="email" name="email" value="nope" maxlength="320" id="id_email" required aria-invalid="true" aria-describedby="id_email_helptext id_email_error"></div>
```

### Field by field

For your own markup, print each part:

```html+django
<form method="post" novalidate>
    {% csrf_token %}
    {{ form.non_field_errors }}

    <div class="field">
        {{ form.email.label_tag }}
        {{ form.email }}
        {% if form.email.help_text %}<small>{{ form.email.help_text }}</small>{% endif %}
        {{ form.email.errors }}
    </div>

    <button type="submit">Send</button>
</form>
```

Or loop over the fields:

```html+django
{% for field in form %}
    <div class="field{% if field.errors %} has-error{% endif %}">
        {{ field.label_tag }}
        {{ field }}
        {% for error in field.errors %}<p class="error">{{ error }}</p>{% endfor %}
    </div>
{% endfor %}
```

| In a template | Gives |
|---|---|
| `{{ form.email }}` | The widget. |
| `{{ form.email.label_tag }}` | `<label for="id_email">Email:</label>` |
| `{{ form.email.label }}`, `{{ form.email.help_text }}` | The texts. |
| `{{ form.email.errors }}` | A `<ul class="errorlist">`, or nothing; loop over it for single messages. |
| `{{ form.email.id_for_label }}`, `{{ form.email.html_name }}` | The id and the name in the HTML. |
| `{{ form.email.value }}` | The value shown in the widget. |
| `{{ form.email.is_hidden }}` | Whether it is a hidden input. |
| `{{ form.non_field_errors }}` | The errors of the whole form, as `<ul class="errorlist nonfield">`. |
| `{% for field in form %}` | Every field; also `form.visible_fields` and `form.hidden_fields`. |
| `{{ form.is_multipart }}` | Whether the form has a file field. |

In PHP code the same parts are methods: `$form->field('email')->labelTag()`,
`->errors()`, `->helpText()`, `->value()`; `$form->asDiv()` renders the
whole form.

### Files need multipart

A form with a [file field](#file-uploads) must be sent as
`multipart/form-data`:

```html+django
<form method="post" {% if form.is_multipart %}enctype="multipart/form-data"{% endif %}>
    {% csrf_token %}
    {{ form }}
    <button type="submit">Upload</button>
</form>
```

### Labels, initial values and widgets

Each field has a label from its name (`cc_myself` → "Cc myself"), which
`label:` replaces. Widgets accept HTML attributes:

```php
'email' => new Fields\EmailField(
    label: 'Your email',
    initial: 'you@example.com',
    widget: new Widgets\EmailInput(['placeholder' => 'you@example.com', 'class' => 'input']),
),
```

## Fields

Every field accepts these options, by name:

| Option | Default | Meaning |
|---|---|---|
| `required` | `true` | The field must not be empty. |
| `widget` | depends on the field | How it is shown. See [Widgets](#widgets). |
| `label` | from the name | The label. |
| `initial` | `null` | The value of an unbound form. A closure is called. |
| `helpText` | `''` | Shown next to the widget, linked with `aria-describedby`. |
| `errorMessages` | `[]` | Messages by code, such as `['required' => 'Please tell us your name.']`. |
| `validators` | `[]` | Callables that receive the clean value and throw `ValidationError`. |
| `disabled` | `false` | Shown but not editable; submitted values are ignored and the initial value is used. |

The field classes live in `BrewPHP\Forms\Fields`:

| Field | Cleaned to | Own options | Default widget |
|---|---|---|---|
| `CharField` | `string` (trimmed) | `maxLength`, `minLength`, `strip` (`true`), `emptyValue` (`''`) | `TextInput` |
| `EmailField` | `string` | `maxLength` (`320`) | `EmailInput` |
| `URLField` | `string` | `maxLength` (`2048`), `schemes` (`['http', 'https']`) | `URLInput` |
| `SlugField` | `string` of letters, digits, `_` and `-` | as `CharField` | `TextInput` |
| `IntegerField` | `int` | `minValue`, `maxValue` | `NumberInput` |
| `FloatField` | `float` | `minValue`, `maxValue` | `NumberInput` |
| `DecimalField` | `string` such as `"12.50"` (no precision lost) | `maxDigits`, `decimalPlaces`, `minValue`, `maxValue` | `NumberInput` |
| `BooleanField` | `bool` | | `CheckboxInput` |
| `ChoiceField` | `string`, or an enum case | `choices`, `coerce` | `Select` |
| `MultipleChoiceField` | list of `string` or enum cases | `choices`, `coerce` | `SelectMultiple` |
| `DateField` | `DateTimeImmutable` at midnight | `inputFormats` (`['Y-m-d']`) | `DateInput` |
| `DateTimeField` | `DateTimeImmutable` | `inputFormats` (`Y-m-d\TH:i:s`, `Y-m-d\TH:i`, `Y-m-d H:i:s`, `Y-m-d H:i`, `Y-m-d`) | `DateTimeInput` |
| `TimeField` | `string` `"HH:MM:SS"` | | `TimeInput` |
| `UUIDField` | lower-case `string` `"8-4-4-4-12"` | | `TextInput` |
| `JsonField` | decoded value (arrays, not objects) | | `Textarea` |
| `FileField` | `UploadedFile` | `maxSize` (bytes), `extensions` | `FileInput` |
| `ModelChoiceField` | a model object | `queryset`, `toFieldName` (`'pk'`), `labelFrom` | `Select` |
| `ModelMultipleChoiceField` | list of model objects | as `ModelChoiceField` | `SelectMultiple` |

Empty optional fields clean to `null` (`''` for the text fields, `[]` for
the multiple-choice fields, `false` for `BooleanField`).

!!! note "BooleanField is required by default"
    A required `BooleanField` must be checked, as for "I accept the terms".
    Use `required: false` for a plain on/off value.

### Choices

`choices` is an array of `value => label`, or the class name of a backed
enum:

```php
enum Size: string
{
    case Small = 's';
    case Large = 'l';

    public function label(): string
    {
        return ucfirst(strtolower($this->name));
    }
}

'size' => new Fields\ChoiceField(Size::class),          // cleaned to Size::Small...
'drink' => new Fields\ChoiceField(['tea' => 'Tea', 'coffee' => 'Coffee']),
'rating' => new Fields\ChoiceField(['1' => 'Poor', '5' => 'Great'], coerce: intval(...)),
'toppings' => new Fields\MultipleChoiceField(['milk' => 'Milk', 'sugar' => 'Sugar'], required: false),
```

An enum's labels come from a `label()` method when it has one, else from
the case names. Only listed values are accepted, whatever the browser
sends. A `<select>` gets an empty `---------` first option.

### Validators

`validators` takes callables that receive the clean value and throw a
`ValidationError`; every message is collected:

```php
use BrewPHP\Core\ValidationError;

'code' => new Fields\CharField(validators: [
    static function (string $value): void {
        if (!str_starts_with($value, 'BREW-')) {
            throw new ValidationError('Codes start with BREW-.');
        }
    },
]),
```

## Widgets

Widgets live in `BrewPHP\Forms\Widgets`. Their first argument is an array
of HTML attributes, escaped when rendered.

| Widget | HTML |
|---|---|
| `TextInput` | `<input type="text">` |
| `EmailInput` | `<input type="email">` |
| `URLInput` | `<input type="url">` |
| `NumberInput` | `<input type="number">` (with `min`, `max` and `step` from the field) |
| `PasswordInput` | `<input type="password">`; the value is never written back, unless `renderValue: true` |
| `HiddenInput` | `<input type="hidden">` |
| `DateInput` | `<input type="date">` |
| `DateTimeInput` | `<input type="datetime-local">` |
| `TimeInput` | `<input type="time">` |
| `Textarea` | `<textarea>` (`cols="40" rows="10"` unless you pass others) |
| `CheckboxInput` | `<input type="checkbox">` |
| `Select` | `<select>` |
| `SelectMultiple` | `<select multiple>` (the name ends with `[]`) |
| `RadioSelect` | a list of radio buttons |
| `CheckboxSelectMultiple` | a list of checkboxes (the name ends with `[]`) |
| `FileInput` | `<input type="file">` |

```php
'password' => new Fields\CharField(strip: false, widget: new Widgets\PasswordInput(['autocomplete' => 'new-password'])),
'plan' => new Fields\ChoiceField(['free' => 'Free', 'pro' => 'Pro'], widget: new Widgets\RadioSelect()),
'tags' => new Fields\MultipleChoiceField($tags, widget: new Widgets\CheckboxSelectMultiple()),
'token' => new Fields\CharField(widget: new Widgets\HiddenInput()),
```

## File uploads

`FileField` cleans to a `BrewPHP\Http\UploadedFile`. It checks the size and
the extension; the file name and the type sent by the browser are not
trusted.

```php
final class AvatarForm extends Form
{
    public function fields(): array
    {
        return [
            'avatar' => new Fields\FileField(maxSize: 2 * 1024 * 1024, extensions: ['png', 'jpg', 'jpeg']),
        ];
    }
}

function upload_avatar(Request $request): \BrewPHP\Http\Response
{
    $form = AvatarForm::fromRequest($request);
    if ($request->isMethod('POST') && $form->isValid()) {
        $file = $form->cleanedData()['avatar'];
        $file->moveTo(base_path('media', 'avatars'), $file->hashName());

        return redirect('pages:profile');
    }

    return render('pages/avatar.html', ['form' => $form]);
}
```

The `extensions` also become the widget's `accept` attribute
(`accept=".png,.jpg,.jpeg"`). Store files under a new name with
`hashName()`, never under the name the browser sent, and never in a folder
where the web server runs PHP. See
[Uploaded files](requests-responses.md#uploaded-files).

## ModelForm

A `ModelForm` builds its fields from a [model](models.md), validates them
with the model's own rules, and saves the object:

```php title="apps/blog/forms.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Forms\ModelForm;
use BrewPHP\Forms\Widgets;

final class PostForm extends ModelForm
{
    protected function model(): string
    {
        return Post::class;
    }

    protected function fieldNames(): array
    {
        return ['title', 'body', 'category', 'published_at'];
    }

    protected function widgets(): array
    {
        return ['body' => new Widgets\Textarea(['rows' => 20])];
    }
}
```

```php title="apps/blog/views.php"
use BrewPHP\Http\Request;

function edit_post(Request $request, int $id): \BrewPHP\Http\Response
{
    $post = get_object_or_404(Post::class, pk: $id);
    $form = PostForm::fromRequest($request, instance: $post);   // without instance: a new post
    if ($request->isMethod('POST') && $form->isValid()) {
        $post = $form->save();

        return redirect('blog:detail', ['id' => $post->pk]);
    }

    return render('blog/post_form.html', ['form' => $form]);
}
```

| Method | Purpose |
|---|---|
| `model()` | The model class (required). |
| `fieldNames()` | The model fields of the form, in order (required). `['__all__']` takes every editable field. |
| `exclude()` | Fields to leave out, handy with `['__all__']`. |
| `widgets()` | Widgets by field name, replacing the defaults. |
| `fields()` | Override it and add to `parent::fields()` for extra fields or to replace one. |
| `instance()` | The object being edited (a new, unsaved one when none was given). |
| `save(commit: true)` | Saves the object and returns it. `commit: false` returns it unsaved, to set more values first. Only call it after `isValid()`. |

```php
$post = $form->save(commit: false);
$post->author = $request->user();
$post->save();
```

### Fields must be listed

`fieldNames()` has no default: a `ModelForm` without it raises
`ImproperlyConfigured`. Only the listed fields are read from the request,
so a visitor cannot set other columns (`author`, `is_staff`) by adding
them to the POST data. `['__all__']` must be asked for on purpose, and
remember that it includes fields added to the model later. Fields with
`editable: false` and auto-increment ids can never be listed.

### Model fields to form fields

| Model field | Form field |
|---|---|
| `CharField`, `TextField` (`Textarea`) | `CharField` |
| `EmailField`, `URLField`, `SlugField` | the same form field |
| `IntegerField`, `BigIntegerField` | `IntegerField` |
| `PositiveIntegerField` | `IntegerField(minValue: 0)` |
| `FloatField`, `DecimalField` | `FloatField`, `DecimalField` |
| `BooleanField` | `BooleanField(required: false)` |
| `DateField`, `DateTimeField`, `TimeField` | the same form field |
| `UUIDField`, `JsonField` | the same form field |
| `ForeignKey`, `OneToOneField` | `ModelChoiceField` with every object of the target model |
| any field with `choices` | `ChoiceField` |

`blank: true` makes the form field optional; the model field's
`verboseName`, `helpText` and `default` become the label, help text and
initial value.

### Model validation

After the form fields are clean, the values are copied to the object and
its `fullClean()` runs for the fields of the form: field validators, the
model's `clean()`, unique fields and unique indexes. Its messages are shown
next to the fields:

```text
Post with this slug already exists.
```

## ModelChoiceField

`ModelChoiceField` offers objects of a QuerySet and cleans to the chosen
object. Only objects of the QuerySet are accepted: a visitor who sends the
id of another row gets "Select a valid choice. That choice is not one of
the available choices."

This makes the QuerySet an access-control tool. A `ModelForm` offers every
object of the target model; limit it to what the user may choose:

```php
use BrewPHP\Auth\Models\User;
use BrewPHP\Forms\Fields\ModelChoiceField;

final class PostForm extends ModelForm
{
    public function __construct(?array $data = null, array $files = [], array $initial = [], string $prefix = '', ?string $autoId = 'id_%s', ?\BrewPHP\Db\Model $instance = null, private ?User $user = null)
    {
        parent::__construct($data, $files, $initial, $prefix, $autoId, $instance);
    }

    // model() and fieldNames() as before

    public function fields(): array
    {
        $fields = parent::fields();
        $fields['category'] = new ModelChoiceField(
            Category::objects()->filter(owner: $this->user?->pk),
            labelFrom: static fn (Category $category): string => $category->name,
        );

        return $fields;
    }
}

$form = PostForm::fromRequest($request, instance: $post, user: $request->user());
```

The QuerySet only runs when the widget is rendered or a value is cleaned.

## Security

Forms are written to be safe with hostile input:

- **Strict shapes.** A field accepts only the shape it expects: a string,
  or a list for multiple choices. An array sent where a string is expected
  (`name[]=x`) is rejected with "Enter a valid value." instead of causing a
  PHP error. Text fields reject NUL characters.
- **Choices are enforced.** `ChoiceField`, `MultipleChoiceField` and the
  model choice fields accept only their listed values, whatever the
  browser sends.
- **Everything is escaped.** Labels, help texts, values, choice labels,
  error messages and attributes are HTML-escaped, and attribute names are
  checked, so no submitted value can inject markup. Messages that repeat
  user input (such as an invalid choice) are escaped too.
- **Safe URLs.** `URLField` accepts only `http` and `https` (or the
  `schemes` you give), so a `javascript:` link can never be saved and shown
  later. Without a scheme, `https://` is added.
- **Passwords are not sent back.** `PasswordInput` never writes the value
  into the page, so a password does not travel back after a failed submit.
  Use `strip: false` on password fields so spaces are kept.
- **No mass assignment.** A `ModelForm` sets only its listed fields.
- **CSRF.** Every `POST` form needs `{% csrf_token %}`; see
  [Security](security.md#csrf-protection).

!!! note "Coming from Django"
    `form.cleaned_data` is `$form->cleanedData()`, `form.is_valid()` is
    `$form->isValid()`, and `clean_<field>()` is `clean<Field>()`. The
    `Meta` class of a `ModelForm` becomes the `model()`, `fieldNames()`,
    `exclude()` and `widgets()` methods. Formsets are not available yet.
