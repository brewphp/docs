# Models

A **model** is a PHP class that describes one database table: its fields
(columns), their types and how they are validated. BrewPHP's ORM follows
Django closely: you declare fields once, and you get a table, queries,
validation and (later) forms and the admin from them.

```php title="apps/blog/models.php"
<?php

declare(strict_types=1);

namespace Apps\Blog;

use BrewPHP\Db\Fields\BooleanField;
use BrewPHP\Db\Fields\CharField;
use BrewPHP\Db\Fields\DateTimeField;
use BrewPHP\Db\Fields\EmailField;
use BrewPHP\Db\Fields\ForeignKey;
use BrewPHP\Db\Fields\TextField;
use BrewPHP\Db\Model;
use BrewPHP\Db\OnDelete;

final class Author extends Model
{
    public static function fields(): array
    {
        return [
            'name' => new CharField(maxLength: 100),
            'email' => new EmailField(unique: true),
        ];
    }

    public function __toString(): string
    {
        return (string) $this->name;
    }
}

final class Post extends Model
{
    public static function fields(): array
    {
        return [
            'title' => new CharField(maxLength: 200),
            'body' => new TextField(blank: true),
            'author' => new ForeignKey(Author::class, onDelete: OnDelete::Cascade, relatedName: 'posts'),
            'published' => new BooleanField(default: false),
            'created_at' => new DateTimeField(autoNowAdd: true),
        ];
    }

    public static function ordering(): array
    {
        return ['-created_at'];
    }

    public function __toString(): string
    {
        return (string) $this->title;
    }
}
```

This gives two tables, `blog_author` and `blog_post`:

```sql
CREATE TABLE "blog_post" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" varchar(200) NOT NULL,
    "body" text NOT NULL,
    "author_id" bigint NOT NULL,
    "published" bool NOT NULL,
    "created_at" datetime NOT NULL,
    CONSTRAINT "blog_post_author_id_..._fk" FOREIGN KEY ("author_id") REFERENCES "blog_author" ("id") DEFERRABLE INITIALLY DEFERRED
)
```

and you can use them right away:

```php
use Apps\Blog\Author;
use Apps\Blog\Post;

$ann = Author::create(['name' => 'Ann', 'email' => 'ann@example.com']);
$post = Post::create(['title' => 'Hello', 'author' => $ann]);

$post->title = 'Hello, world';
$post->save();

$posts = Post::objects()->filter(author__name: 'Ann', published: true);
```

[Making queries](queries.md) covers querying in detail.

## Where models live

Put models in the `models.php` file of an app, or one class per file in the
app's `Models/` folder (`apps/blog/Models/Post.php` with the namespace
`Apps\Blog\Models`). BrewPHP loads both when the app boots, so every model
of the project is known.

A model belongs to the installed app whose namespace contains it, so
`Apps\Blog\Post` and `Apps\Blog\Models\Post` both belong to the app `blog`.
The registry lists them:

```php
use BrewPHP\Db\Registry;

Registry::models();          // every model class
Registry::models('blog');    // the models of the blog app
Registry::get('blog.Post');  // one model class by label
```

## Creating the tables

Migrations (`makemigrations` and `migrate`) arrive in v0.4. Until then,
create tables with the **schema editor**, for example from a small
`manage.php` command:

```php title="apps/blog/Commands/CreateTablesCommand.php"
<?php

declare(strict_types=1);

namespace Apps\Blog\Commands;

use Apps\Blog\Author;
use Apps\Blog\Post;
use BrewPHP\Console\Command;

final class CreateTablesCommand extends Command
{
    public string $name = 'blog:create-tables';

    public string $description = 'Create the tables of the blog app';

    public function handle(): int
    {
        $editor = db()->schemaEditor();
        $editor->createModel(Author::class);  // referenced tables first
        $editor->createModel(Post::class);
        $this->success('Tables created.');

        return self::SUCCESS;
    }
}
```

```bash
php manage.php blog:create-tables
```

The schema editor can also change existing tables; see
[The schema editor](#the-schema-editor) below.

## Fields

`fields()` returns the fields by name. Field names are lowercase with
underscores (`created_at`), cannot contain a double underscore, and `pk` is
reserved.

Every model gets an auto-increment primary key named `id` (a
`BigAutoField`), unless one of its fields has `primaryKey: true`.

### Field options

All fields accept these named arguments:

| Option | Default | Meaning |
|---|---|---|
| `null` | `false` | Store empty values as `NULL`. |
| `blank` | `false` | Allow empty values in validation (`fullClean()`, forms). |
| `default` | none | Value for new objects. A closure is called for each object: `default: fn () => []`. |
| `unique` | `false` | Add a `UNIQUE` constraint. |
| `dbIndex` | `false` | Add an index on the column. |
| `choices` | `null` | Allowed values: an array `['s' => 'Small']` or a backed enum class. |
| `verboseName` | field name with spaces | Human-readable name, e.g. for form labels. |
| `helpText` | `''` | Extra help shown with form widgets. |
| `editable` | `true` | `false` hides the field from forms and the admin. |
| `primaryKey` | `false` | Make this field the primary key instead of `id`. |
| `dbColumn` | field name | Name of the database column. |
| `validators` | `[]` | Callables that throw `ValidationError` for invalid values. |

!!! note "`null` and `blank`"
    `null` is about the database, `blank` about validation. For text fields,
    prefer `blank: true` alone: an empty value is then stored as `''`, so
    there is only one kind of "empty".

### Field types

All field classes are in the `BrewPHP\Db\Fields` namespace.

| Field | PHP value | Extra options | Column (SQLite / MySQL / PostgreSQL) |
|---|---|---|---|
| `AutoField` | `int` | | auto-increment `integer` |
| `BigAutoField` | `int` | | auto-increment `bigint` (the implicit `id`) |
| `CharField` | `string` | `maxLength` (255) | `varchar(n)` |
| `TextField` | `string` | | `text` / `longtext` / `text` |
| `SlugField` | `string` | `maxLength` (50); indexed | `varchar(n)` |
| `EmailField` | `string` | `maxLength` (254) | `varchar(n)` |
| `URLField` | `string` | `maxLength` (200) | `varchar(n)` |
| `IntegerField` | `int` | | `integer` |
| `BigIntegerField` | `int` | | `bigint` |
| `PositiveIntegerField` | `int` (≥ 0) | | `integer` |
| `FloatField` | `float` | | `real` / `double precision` |
| `DecimalField` | numeric `string`, e.g. `"19.90"` | `maxDigits` (10), `decimalPlaces` (2) | `decimal(m, d)` / `decimal(m, d)` / `numeric(m, d)` |
| `BooleanField` | `bool` | | `bool` / `bool` / `boolean` |
| `DateField` | `DateTimeImmutable` (midnight) | `autoNow`, `autoNowAdd` | `date` |
| `DateTimeField` | `DateTimeImmutable` | `autoNow`, `autoNowAdd` | `datetime` / `datetime` / `timestamp` |
| `TimeField` | `string`, e.g. `"09:30:00"` | | `time` |
| `JsonField` | array or any JSON value | | `text` / `json` / `jsonb` |
| `UUIDField` | `string` | | `char(36)` / `char(36)` / `uuid` |
| `ForeignKey` | the related object | `to`, `onDelete`, `relatedName` | type of the target key |
| `OneToOneField` | the related object | same as `ForeignKey`; unique | type of the target key |

Notes:

- `DecimalField` values are strings so that no precision is lost to floats.
- `DateTimeField` stores `Y-m-d H:i:s` in the project's `TIME_ZONE`, to the
  second; there is no time zone conversion.
- `autoNowAdd: true` sets the date when the object is created, `autoNow: true`
  on every save. Both make the field non-editable and blank.
- For a random UUID primary key:
  `'id' => new UUIDField(primaryKey: true, default: UUIDField::uuid4(...))`.
- Strings are converted when values are validated, so `"42"` becomes `42`
  for an `IntegerField` and `"on"` becomes `true` for a `BooleanField`.

### Choices

`choices` limits a field to some values. With an array, keys are the stored
values and values are labels:

```php
'size' => new CharField(maxLength: 1, choices: ['s' => 'Small', 'm' => 'Medium', 'l' => 'Large']),
```

With a backed enum, the PHP value is the enum case and the database stores
its backing value:

```php
enum Status: string
{
    case Draft = 'draft';
    case Published = 'published';

    // Optional: the label shown in forms (default: the case name).
    public function label(): string
    {
        return $this === self::Draft ? 'Draft' : 'Published';
    }
}

'status' => new CharField(maxLength: 20, choices: Status::class, default: Status::Draft),
```

```php
$post->status === Status::Draft;                    // true
Post::objects()->filter(status: Status::Published); // enum cases work in queries
Post::objects()->filter(status: 'published');       // and so do raw values
```

## Relations

### ForeignKey

A `ForeignKey` is a many-to-one relation: each post has one author, and an
author has many posts.

```php
'author' => new ForeignKey(Author::class, onDelete: OnDelete::Cascade, relatedName: 'posts'),
```

| Argument | Meaning |
|---|---|
| `to` | The target model: a class name, `'self'` (the same model), `'blog.Author'` (app label and model name), or `'Author'` (a model of the same app). |
| `onDelete` | What happens to this object when the target is deleted (below). Default `OnDelete::Cascade`. |
| `relatedName` | The name of the reverse relation on the target. Default `<model>_set` (e.g. `post_set`). |

The column is the field name plus `_id` (`author_id`), and it is indexed.
On an object:

```php
$post->author;               // the Author object, loaded on first access and cached
$post->author_id;            // the raw id, without a query
$post->author = $bob;        // assign an object...
$post->author_id = 7;        // ...or an id
$post->category = null;      // a nullable foreign key can be cleared
```

The reverse relation is a [QuerySet](queries.md) of the objects that point
to this one:

```php
$ann->posts;                                   // QuerySet of Ann's posts (relatedName)
$ann->related('posts')->filter(published: true);
$ann->posts->create(['title' => 'New']);       // sets the author automatically
```

In queries, follow relations with double underscores, in both directions:
`Post::objects()->filter(author__name: 'Ann')`,
`Author::objects()->filter(posts__title__icontains: 'php')`. Without a
`relatedName`, the reverse name in queries is the model name in snake case
(`post`).

### On delete

When an object is deleted, BrewPHP applies the `onDelete` rule of every
foreign key that points to it. It does this in PHP (not with `ON DELETE`
clauses), so it works the same on every database and the model hooks of
every deleted object run.

| `OnDelete` | Effect |
|---|---|
| `Cascade` | Delete the pointing objects too (recursively). |
| `Protect` | Refuse with a `BrewPHP\Db\ProtectedError` listing the objects that block it. |
| `Restrict` | Same as `Protect` in BrewPHP. |
| `SetNull` | Set the foreign key to `NULL` (the field needs `null: true`). |
| `SetDefault` | Set the foreign key to its default. |
| `DoNothing` | Nothing; the database constraint decides. |

```php
try {
    $category->delete();
} catch (ProtectedError $e) {
    // $e->protectedObjects are the posts still in this category
}
```

### OneToOneField

A `OneToOneField` is a unique foreign key. Its reverse relation returns a
single object instead of a QuerySet:

```php
final class Profile extends Model
{
    public static function fields(): array
    {
        return [
            'author' => new OneToOneField(Author::class, relatedName: 'profile'),
            'bio' => new TextField(blank: true),
        ];
    }
}

$ann->profile->bio;   // throws ObjectDoesNotExist when Ann has no profile
```

## Model options

Override these static methods to change how a model is stored and shown:

| Method | Default | Meaning |
|---|---|---|
| `table()` | `"<app label>_<model>"`, e.g. `blog_post` | Table name. |
| `ordering()` | `[]` | Default ordering of queries, e.g. `['-created_at', 'title']`. |
| `indexes()` | `[]` | Extra indexes, including multi-column unique ones (below). |
| `verboseName()` | class name with spaces, e.g. `blog post` | Human-readable name. |
| `verboseNamePlural()` | verbose name + `s` | Plural name. |
| `appLabel()` | the app whose namespace contains the class | App the model belongs to. |

Indexes on several fields use `BrewPHP\Db\Index`; a unique index is
Django's `unique_together`:

```php
use BrewPHP\Db\Index;

public static function indexes(): array
{
    return [
        new Index(['author', 'created_at']),
        new Index(['author', 'slug'], unique: true),
        new Index(['title'], name: 'post_title_idx'),
    ];
}
```

Also define `__toString()`: it names the object in messages, forms and the
admin. The default is `Post object (5)`.

## Working with objects

```php
$post = new Post(['title' => 'Draft']);  // not saved yet; defaults are applied
$post->author = $ann;
$post->save();                           // INSERT; $post->pk is now set

$post->title = 'Final';
$post->save();                           // UPDATE

$post = Post::create(['title' => 'Quick', 'author' => $ann]);  // new + save

$post->pk;                // the primary key, whatever the field is called
$post->isPersisted();     // true once saved or loaded
$post->refresh();         // reload the values from the database
$post->toArray();         // ['id' => 1, 'title' => 'Quick', 'author_id' => 1, ...]
json_encode($post);       // the same, with dates as ISO 8601 strings
$post->delete();          // returns the number of deleted rows (with cascades)
```

Properties that are not fields can be set freely (for example a value you
computed in a view), but reading an unknown property throws a
`BrewPHP\Db\FieldError`, which catches typos.

### Hooks

Override these methods to run code around saving and deleting:

```php
final class Post extends Model
{
    // ...

    public function beforeSave(): void
    {
        $this->title = trim((string) $this->title);
    }

    public function afterSave(bool $created): void
    {
        if ($created) {
            // notify subscribers...
        }
    }

    public function beforeDelete(): void {}

    public function afterDelete(): void {}
}
```

The delete hooks also run for objects deleted by a cascade or by
`QuerySet::delete()`. `QuerySet::update()` runs a single SQL statement and
does not call any hook.

## Validation

`save()` does not validate: like Django, validation is a separate step.
`fullClean()` checks every field, then the model's `clean()` method, then
unique constraints, and throws a `BrewPHP\Core\ValidationError` with
messages per field:

```php
use BrewPHP\Core\ValidationError;

$post = new Post(['title' => '', 'author_id' => 999]);

try {
    $post->fullClean();
    $post->save();
} catch (ValidationError $e) {
    $e->errors();
    // ['title' => ['This field cannot be blank.'],
    //  'author' => ['Author instance with id 999 does not exist.']]
}
```

The built-in checks are: not null, not blank, valid choice, maximum length,
the format of emails, URLs, slugs, UUIDs, numbers and dates, decimal digits,
positive integers, that a related object exists, and uniqueness (fields with
`unique: true` and unique indexes). Values are converted on the way (`"42"`
becomes `42`). Empty values of `blank` fields are not validated.

Validate the whole object in `clean()`, for example to compare two fields.
A message for one field uses its name as the key; other messages go under
`__all__` (`ValidationError::NON_FIELD`):

```php
public function clean(): void
{
    if ($this->ends_at < $this->starts_at) {
        throw new ValidationError(['ends_at' => 'The end must be after the start.']);
    }
}
```

Validators are callables that receive the cleaned value and throw a
`ValidationError`:

```php
$even = static function (mixed $value): void {
    if ($value % 2 !== 0) {
        throw new ValidationError('Enter an even number.');
    }
};

'seats' => new IntegerField(validators: [$even]),
```

`fullClean($exclude)` skips the fields listed in `$exclude`, and
`fullClean(validateUnique: false)` skips the uniqueness queries.

## Metadata

`Post::meta()` returns a `BrewPHP\Db\ModelMeta` with everything BrewPHP
knows about the table (Django's `_meta`):

```php
$meta = Post::meta();
$meta->table;               // "blog_post"
$meta->appLabel;            // "blog"
$meta->label();             // "blog.Post"
$meta->pk;                  // the primary key Field
$meta->fields;              // fields by name, primary key first
$meta->field('author');     // one Field (throws FieldError if unknown)
$meta->field('author')->column;  // "author_id"
$meta->reverseRelations();  // ForeignKeys of other models pointing here
```

## The schema editor

The schema editor creates and changes tables from model definitions. It is
what migrations will use in v0.4, and what you use until then.

```php
$editor = db()->schemaEditor();
```

| Method | What it does |
|---|---|
| `createModel(Post::class)` | `CREATE TABLE` with constraints and indexes |
| `deleteModel(Post::class)` | `DROP TABLE` |
| `renameModel('blog_post', 'blog_article')` | Rename a table |
| `addField(Post::class, $field)` | Add a column |
| `removeField(Post::class, $field)` | Drop a column |
| `alterField(Post::class, $old, $new)` | Change a column: name, type, `null`, `unique`, index, foreign key |
| `renameField(Post::class, $field, 'headline')` | Rename a column |
| `addIndex(Post::class, new Index([...]))` | Create an index |
| `removeIndex(Post::class, new Index([...]))` | Drop an index |

Fields passed to these methods must belong to a model, for example
`Post::meta()->field('subtitle')` after adding `subtitle` to `fields()`:

```php
$editor->addField(Post::class, Post::meta()->field('subtitle'));
```

A new non-null field needs a default (not a closure) when the table already
has rows; the default becomes the value of the existing rows.

SQLite cannot alter most column properties, so on SQLite `alterField()`,
`removeField()` and some `addField()` calls rebuild the table: create a new
table, copy the rows, drop the old table and rename the new one. That is
why the model must describe the table **as it is in the database**: a
column the model does not know about would be lost in a rebuild.

To see the SQL without running it, collect it:

```php
$editor = db()->schemaEditor(collectOnly: true);
$editor->createModel(Post::class);
print_r($editor->collectedSql());
```
