# Making queries

Once you have [models](models.md), you query them through **QuerySets**.
The examples on this page use these models:

```php
final class Author extends Model
{
    public static function fields(): array
    {
        return [
            'name' => new CharField(maxLength: 100),
            'email' => new EmailField(blank: true),
        ];
    }
}

final class Post extends Model
{
    public static function fields(): array
    {
        return [
            'title' => new CharField(maxLength: 200),
            'author' => new ForeignKey(Author::class, relatedName: 'posts'),
            'views' => new PositiveIntegerField(default: 0),
            'published' => new BooleanField(default: false),
            'created_at' => new DateTimeField(autoNowAdd: true),
        ];
    }
}
```

## Creating objects

```php
$ann = Author::create(['name' => 'Ann']);          // INSERT, returns the saved object

$post = new Post(['title' => 'Hello', 'author' => $ann]);
$post->save();                                     // same in two steps

Post::objects()->create(['title' => 'Hi', 'author' => $ann]);
```

## QuerySets

`Post::objects()` returns a `BrewPHP\Db\QuerySet` of every post. You narrow
it down with methods that each return a **new** QuerySet:

```php
$posts = Post::objects()
    ->filter(published: true)
    ->exclude(title__icontains: 'draft')
    ->orderBy('-created_at')
    ->limit(10);
```

QuerySets are:

- **lazy**: no query runs until you need the results;
- **immutable**: `filter()` never changes the QuerySet you call it on, so a
  base QuerySet can be reused safely;
- **cached**: once evaluated, the rows are kept, so iterating twice runs
  one query.

A QuerySet runs its query when you iterate over it, count it, or call a
method that returns objects or values:

```php
foreach ($posts as $post) {
    echo $post->title;
}

$list = $posts->toArray();   // list<Post>
count($posts);               // uses the cached rows if the query already ran
```

## Filtering

`filter()` keeps the rows that match **all** its lookups, `exclude()`
removes the rows that match all of its lookups. Lookups are named
arguments: a field name, optionally followed by a double underscore and a
lookup type.

```php
Post::objects()->filter(published: true);           // exact match
Post::objects()->filter(views__gte: 100);           // views >= 100
Post::objects()->filter(title__icontains: 'php', published: true);
Post::objects()->filter(['title__icontains' => 'php']);  // an array works too
Post::objects()->filter(pk: 5);                     // pk is the primary key
```

### Lookups

| Lookup | SQL (roughly) | Example |
|---|---|---|
| `exact` (default) | `= value`, or `IS NULL` for `null` | `filter(title: 'Hello')` |
| `iexact` | case-insensitive equality | `filter(title__iexact: 'hello')` |
| `contains` | `LIKE '%value%'`, case-sensitive | `filter(title__contains: 'PHP')` |
| `icontains` | case-insensitive `contains` | `filter(title__icontains: 'php')` |
| `startswith`, `istartswith` | `LIKE 'value%'` | `filter(title__startswith: 'How')` |
| `endswith`, `iendswith` | `LIKE '%value'` | `filter(email__iendswith: '@example.com')` |
| `gt`, `gte`, `lt`, `lte` | `>`, `>=`, `<`, `<=` | `filter(views__lt: 10)` |
| `in` | `IN (...)` | `filter(id__in: [1, 2, 3])` |
| `range` | `BETWEEN low AND high` | `filter(views__range: [10, 100])` |
| `isnull` | `IS NULL` / `IS NOT NULL` | `filter(published_at__isnull: true)` |
| `year`, `month`, `day` | part of a date | `filter(created_at__year: 2026)` |

`year`, `month` and `day` can be followed by another lookup:
`filter(created_at__year__gte: 2025)`.

Values are converted by the field, so you can pass PHP values:
`filter(created_at__gte: new DateTimeImmutable('-7 days'))`,
`filter(status: Status::Published)`, `filter(author: $ann)`.

Wildcards in the value are literal: `filter(title__contains: '100%')`
finds titles containing "100%". An empty list for `in` matches nothing.

`in` also accepts a QuerySet, which becomes a subquery:

```php
Post::objects()->filter(author__in: Author::objects()->filter(email__endswith: '@example.com'));
```

### Following relations

Double underscores follow foreign keys, as deep as you need. BrewPHP adds
the JOINs:

```php
Post::objects()->filter(author__name: 'Ann');
Comment::objects()->filter(post__author__name__startswith: 'A');
```

They also work backwards, from the target of a foreign key, through its
`relatedName`:

```php
Author::objects()->filter(posts__views__gt: 1000);       // authors with a popular post
Author::objects()->filter(posts__isnull: true);          // authors without posts
Author::objects()->filter(posts__published: true)->distinct();
```

A reverse relation can match several rows per object; `distinct()` removes
the duplicates.

### exclude() and NULL

`exclude(views__gt: 10)` keeps the rows where `views` is `NULL`, like
Django (plain SQL `NOT (views > 10)` would drop them).

## Q objects: OR and NOT

Lookups given to `filter()` are combined with AND. For OR and NOT, build
`BrewPHP\Db\Q` objects and pass them to `where()` (or `filter()`):

```php
use BrewPHP\Db\Q;

// title contains "php" OR more than 100 views
Post::objects()->where(Q::or(new Q(title__icontains: 'php'), new Q(views__gt: 100)));

// NOT published
Post::objects()->where(Q::not(new Q(published: true)));
Post::objects()->where((new Q(published: true))->negate());

// combine freely
$popular = Q::or(new Q(views__gt: 100), new Q(author__name: 'Ann'));
Post::objects()->where(Q::and($popular, new Q(published: true)));

// Q objects and lookups together
Post::objects()->filter($popular, published: true);
```

## F expressions

`BrewPHP\Db\F` refers to a column, so the database compares or computes
with it instead of PHP:

```php
use BrewPHP\Db\F;

// posts with more likes than views
Post::objects()->filter(likes__gt: new F('views'));

// add one view without reading the row first (no race condition)
Post::objects()->filter(pk: $post->pk)->update(['views' => (new F('views'))->add(1)]);
```

`add()`, `subtract()`, `multiply()` and `divide()` take a number or another
`F`. In filters, an `F` can follow relations: `new F('author__score')`.

## Ordering and slicing

```php
Post::objects()->orderBy('-created_at');            // newest first
Post::objects()->orderBy('author__name', 'title');  // across a relation
Post::objects()->orderBy();                         // no ordering at all
Post::objects()->orderBy('views')->reverse();       // highest views first
```

Without `orderBy()`, the model's `ordering()` is used.

```php
Post::objects()->limit(10);          // the first 10
Post::objects()->offset(20);         // all but the first 20
Post::objects()->slice(20, 10);      // 10 rows starting at row 20
```

You cannot filter or reorder a QuerySet once a limit or offset is set; do
it before slicing.

## Retrieving single objects

```php
$post = Post::objects()->get(pk: 5);               // exactly one
$post = Post::objects()->filter(published: true)->get(title: 'Hello');

$first = Post::objects()->orderBy('created_at')->first();  // or null
$last = Post::objects()->last();                           // or null
```

`get()` throws `BrewPHP\Db\ObjectDoesNotExist` when nothing matches and
`BrewPHP\Db\MultipleObjectsReturned` when several rows match. `first()` and
`last()` order by the primary key when the QuerySet has no ordering.

In a view, `get_object_or_404()` turns a missing object into a 404 page:

```php
function post_detail(Request $request, int $id): Response
{
    $post = get_object_or_404(Post::class, id: $id, published: true);

    return render('blog/post_detail.php', ['post' => $post]);
}
```

It also accepts a QuerySet: `get_object_or_404(Post::objects()->filter(published: true), slug: $slug)`.

## Counting and checking

```php
Post::objects()->filter(published: true)->count();    // SELECT COUNT(*)
Post::objects()->filter(title: 'Hello')->exists();    // cheaper than count()
```

## Values instead of objects

```php
Post::objects()->values('title', 'author__name');
// [['title' => 'Hello', 'author__name' => 'Ann'], ...]

Post::objects()->values();   // every field, by attribute name ('author_id')

Post::objects()->valuesList('id', 'title');           // [[1, 'Hello'], [2, 'Hi']]
Post::objects()->valuesList('title', flat: true);      // ['Hello', 'Hi']

Post::objects()->inBulk([1, 2, 3]);                    // [1 => Post, 2 => Post, 3 => Post]
Author::objects()->inBulk(['ann@example.com'], 'email');  // by another unique field
```

## Aggregation

`aggregate()` computes summary values over the matching rows. Name each
result with a named argument; unnamed ones get a key like `views__sum`:

```php
use BrewPHP\Db\Avg;
use BrewPHP\Db\Count;
use BrewPHP\Db\Max;
use BrewPHP\Db\Min;
use BrewPHP\Db\Sum;

Post::objects()->filter(published: true)->aggregate(
    new Count('id'),
    total: new Sum('views'),
    average: new Avg('views'),
    newest: new Max('created_at'),
);
// ['id__count' => 12, 'total' => 1520, 'average' => 126.67, 'newest' => DateTimeImmutable]

Post::objects()->aggregate(authors: new Count('author', distinct: true));
Author::objects()->aggregate(views: new Sum('posts__views'));   // across a relation
Post::objects()->aggregate(rows: new Count('*'));
```

`Count` gives an `int` and `Avg` a `float`; `Sum`, `Min` and `Max` are
converted by the field (a `Max` of a date field is a `DateTimeImmutable`).
They are `null` when there are no rows (`Count` is `0`).

## Loading related objects

`$post->author` runs a query the first time it is read. In a loop over many
posts that is one query per post. `selectRelated()` loads the related
objects in the same query with a JOIN:

```php
foreach (Post::objects()->selectRelated('author') as $post) {
    echo $post->author->name;   // no extra query
}

Comment::objects()->selectRelated('post__author');   // two levels
```

`selectRelated()` follows foreign keys forwards only.

## Updating and deleting

```php
// one object
$post->title = 'New title';
$post->save();
$post->delete();

// many objects with one statement; returns the number of rows
Post::objects()->filter(author: $ann)->update(['published' => true]);
Post::objects()->filter(views: 0)->delete();
```

`update()` runs a single `UPDATE` and does not call `save()`, model hooks or
`autoNow`. `delete()` on a QuerySet loads the objects, applies the
[`onDelete`](models.md#on-delete) rules and calls the delete hooks of every
object, in a transaction.

## Get or create

```php
[$tag, $created] = Tag::objects()->getOrCreate(['slug' => 'php'], ['name' => 'PHP']);

[$author, $created] = Author::objects()->updateOrCreate(
    ['email' => 'ann@example.com'],     // how to find it
    ['name' => 'Ann Smith'],            // what to set
);
```

The first array finds the object; when it is missing, it is created from
both arrays (lookups with `__` are not used as values).

## Empty QuerySets

`none()` returns a QuerySet that is always empty and never queries the
database, handy as a default:

```php
$posts = $user !== null ? Post::objects()->filter(author: $user) : Post::objects()->none();
```

## Pagination

`paginate()` returns one page of results. Pass the page number straight
from the query string: an invalid number gives the first page, a number
past the end gives the last page.

```php
function post_list(Request $request): Response
{
    $page = Post::objects()->filter(published: true)->orderBy('-created_at')->paginate(10, $request->query('page'));

    return render('blog/post_list.php', ['page' => $page]);
}
```

```php title="apps/blog/templates/blog/post_list.php"
<?php foreach ($page as $post): ?>
    <h2><?= e($post->title) ?></h2>
<?php endforeach ?>

<p>Page <?= e($page->number) ?> of <?= e($page->paginator->numPages()) ?></p>
<?php if ($page->hasPrevious()): ?>
    <a href="?page=<?= e($page->previousPageNumber()) ?>">Previous</a>
<?php endif ?>
<?php if ($page->hasNext()): ?>
    <a href="?page=<?= e($page->nextPageNumber()) ?>">Next</a>
<?php endif ?>
```

The `BrewPHP\Pagination\Paginator` class works with QuerySets and plain
arrays, and only loads one page of rows:

```php
use BrewPHP\Pagination\Paginator;

$paginator = new Paginator(Post::objects()->orderBy('id'), perPage: 25, orphans: 3);
$paginator->count();          // total items
$paginator->numPages();
$paginator->pageRange();      // [1, 2, 3, ...]
$paginator->elidedPageRange(10);  // [1, null, 8, 9, 10, 11, 12, null, 40]

$page = $paginator->page(2);  // strict: throws PageNotAnInteger or EmptyPage
$page = $paginator->getPage($request->query('page'));  // forgiving
$page->items;                 // the objects of the page
$page->startIndex();          // 26 ("showing 26-50")
$page->endIndex();            // 50
$page->hasOtherPages();
```

`orphans` moves a small last page into the previous one: with 53 items, 25
per page and 3 orphans there are 2 pages, the last with 28 items.

## Transactions

`transaction()` commits when the callback returns and rolls back when it
throws. Nested calls use savepoints, so a failing inner block only undoes
its own work:

```php
db()->transaction(function () use ($order): void {
    $order->save();
    Stock::objects()->filter(product: $order->product)->update(['count' => (new F('count'))->subtract(1)]);
});

$id = db()->transaction(fn () => Post::create(['title' => 'Hi', 'author' => $ann])->pk);
```

The manual form is `beginTransaction()`, `commit()` and `rollBack()`;
`inTransaction()` tells whether one is open.

!!! note
    MySQL commits implicitly after `CREATE TABLE` and other schema changes,
    so they cannot be rolled back there. SQLite and PostgreSQL can.

## Raw SQL

For anything the ORM does not cover, write SQL. Always pass values as
bindings (`?` placeholders), never by concatenating strings:

```php
$rows = db()->select('SELECT title, views FROM blog_post WHERE views > ?', [100]);
$row = db()->selectOne('SELECT * FROM blog_post WHERE id = ?', [5]);   // or null
$total = db()->scalar('SELECT SUM(views) FROM blog_post');
$changed = db()->statement('UPDATE blog_post SET views = 0 WHERE published = ?', [false]);
$id = db()->insert('blog_post', ['title' => 'Raw', 'author_id' => 1, 'views' => 0]);
```

To get model objects from your own SQL, select every column of the table:

```php
$posts = Post::objects()->raw('SELECT * FROM blog_post WHERE length(title) > ?', [20]);
```

To see the SQL of a QuerySet:

```php
[$sql, $bindings] = Post::objects()->filter(author__name: 'Ann')->toSql();
```

Database errors throw `BrewPHP\Db\QueryException`, whose message contains
the SQL; constraint violations (duplicate unique values, `NULL` in a
non-null column, a broken foreign key) throw its subclass
`BrewPHP\Db\IntegrityError`.

## Query log

With `DEBUG` on, every connection records its queries:

```php
foreach (db()->lastQueries() as $query) {
    echo $query['time'], ' ms: ', $query['sql'], PHP_EOL;
}
```

`enableQueryLog()` turns the log on in production, `flushQueryLog()`
empties it. Only the last 500 queries are kept.
