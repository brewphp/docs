# Requests and responses

Every view receives a `BrewPHP\Http\Request` and returns a
`BrewPHP\Http\Response`. Both are small, plain classes.

## The request

Ask for the request in your view by typing a parameter with `Request`:

```php
use BrewPHP\Http\Request;

function search(Request $request): string
{
    $term = (string) $request->query('q', '');

    return 'You searched for ' . e($term);
}
```

### URL and method

For a request to `https://example.com:8443/posts/?page=2`:

| Method | Returns |
|---|---|
| `method()` | `'GET'` (upper-case) |
| `isMethod('post')` | `false` (case-insensitive) |
| `path()` | `'/posts/'` (URL-decoded, always starts with `/`) |
| `fullPath()` | `'/posts/?page=2'` |
| `scheme()` | `'https'` |
| `isSecure()` | `true` |
| `host()` | `'example.com'` (no port, lower-case) |
| `port()` | `8443` |
| `httpHost()` | `'example.com:8443'` (port only when not the default) |
| `url()` | `'https://example.com:8443/posts/'` |
| `fullUrl()` | `'https://example.com:8443/posts/?page=2'` |
| `absoluteUrl('/about/')` | `'https://example.com:8443/about/'` |
| `ip()` | the client's IP address (`REMOTE_ADDR`) |

!!! note "Method override"
    HTML forms can only send `GET` and `POST`. A `POST` with a `_method`
    field (or an `X-HTTP-Method-Override` header) set to `PUT`, `PATCH` or
    `DELETE` is treated as that method:
    `<input type="hidden" name="_method" value="DELETE">`.

### Input

| Method | Reads |
|---|---|
| `query('page', '1')` | a query-string value (`$_GET`); `query()` returns them all |
| `post('title')` | a form-body value (`$_POST`); `post()` returns them all |
| `input('title')` | the body (form or JSON) first, then the query string |
| `all()` | every input value, body merged over query string |
| `has('title')` | whether `all()` contains the key |
| `json()` | the decoded JSON body as an array (empty if invalid) |
| `content()` | the raw body |

```php
use BrewPHP\Http\Request;

function create_post(Request $request): array
{
    $title = trim((string) $request->input('title', ''));
    if ($title === '') {
        return ['error' => 'A title is required.'];
    }

    return ['created' => $title];
}
```

`input()` and `all()` read a JSON body automatically when the request's
`Content-Type` contains `json`.

!!! warning "Input is untrusted"
    Values come from the user. Validate them, cast them to the type you need,
    and escape them with `e()` when you print them. Forms with validation
    arrive in v0.5.

### Headers, cookies and server values

| Method | Returns |
|---|---|
| `header('Accept')` | a header value (case-insensitive), or `null` |
| `headers()` | all headers, with lower-case names |
| `cookie('theme')` | a cookie value, or `null` |
| `cookies()` | all cookies |
| `server('REQUEST_TIME')` | a `$_SERVER` value; `server()` returns them all |
| `isJson()` | the body is JSON |
| `wantsJson()` | the body is JSON or the client accepts `application/json` |
| `isAjax()` | the `X-Requested-With: XMLHttpRequest` header is present |

### Attributes

Attributes carry values attached while the request is handled. The router
stores the matched route as `route`:

```php
$match = $request->attribute('route');   // BrewPHP\Routing\RouteMatch
$request->setAttribute('started_at', microtime(true));
```

### Building requests by hand

`Request::create()` builds a request without a web server, which is handy in
tests. For `GET` requests the parameters become the query string, otherwise
the form body:

```php
use BrewPHP\Core\Application;
use BrewPHP\Http\Request;

$app = Application::create(__DIR__);

$response = $app->handle(Request::create('GET', '/posts/?page=2'));
$response = $app->handle(Request::create('POST', '/contact/', ['message' => 'Hi']));
$response = $app->handle(Request::create('POST', '/api/posts/', server: [
    'CONTENT_TYPE' => 'application/json',
], content: '{"title":"Hello"}'));
```

## Responses

### Response

```php
use BrewPHP\Http\Response;

return new Response('<h1>Created</h1>', 201, ['Cache-Control' => 'no-store']);
```

The default content type is `text/html; charset=utf-8`. Setters return the
response, so calls can be chained:

```php
return (new Response('Hello'))
    ->setStatus(202)
    ->header('X-Powered-By', 'coffee');
```

| Method | Purpose |
|---|---|
| `content()` / `setContent($html)` | the body |
| `status()` / `setStatus($code)` / `statusText()` | the status code and its text |
| `header($name, $value)` | set (replace) a header |
| `getHeader($name)`, `hasHeader($name)`, `removeHeader($name)` | inspect and remove headers (case-insensitive) |
| `headers()` | all headers |
| `cookie(Cookie $cookie)` / `deleteCookie($name)` / `cookies()` | cookies |

Header values with line breaks are rejected, which prevents header
injection.

### JsonResponse

```php
use BrewPHP\Http\JsonResponse;

return new JsonResponse(['id' => 1, 'title' => 'Café'], 201);
```

The body is `{"id":1,"title":"Café"}` with the `Content-Type:
application/json` header. Slashes and Unicode are not escaped. Returning an
array from a view does the same with status 200.

### RedirectResponse

```php
use BrewPHP\Http\RedirectResponse;

return new RedirectResponse('/login/');          // 302 Found
return new RedirectResponse('/posts/5/', 301);   // permanent
```

The status must be a `3xx` code. Use `303` after a successful form `POST`.

### StreamedResponse

For large downloads, write the body while it is sent instead of building it
in memory:

```php
use BrewPHP\Http\StreamedResponse;

return new StreamedResponse(function (): void {
    echo "id,title\n";
    foreach ([[1, 'Hello'], [2, 'World']] as [$id, $title]) {
        echo $id, ',', $title, "\n";
    }
}, headers: [
    'Content-Type' => 'text/csv',
    'Content-Disposition' => 'attachment; filename="posts.csv"',
]);
```

## Helpers

These global functions are shortcuts for the classes above:

| Helper | Returns |
|---|---|
| `response($content, $status, $headers)` | `Response` |
| `json($data, $status, $headers)` | `JsonResponse` |
| `redirect($to, $parameters, $status)` | `RedirectResponse` |
| `render($template, $context, $status, $headers)` | `Response` with a rendered template |
| `abort($status, $message)` | never returns: throws an HTTP exception |
| `e($value)` | the value escaped for HTML |

`redirect()` accepts a URL (anything containing `/`, or starting with `?` or
`#`) or a [route name](routing.md#naming-routes-and-reverse):

```php
return redirect('/login/');
return redirect('post-detail', ['id' => 5]);
return redirect('https://brewphp.dev', status: 301);
```

`e()` converts `&`, `<`, `>`, `"` and `'` to HTML entities. `null` and
`false` print as an empty string, `true` as `1`, and backed enums as their
value.

## Cookies

```php
use BrewPHP\Http\Cookie;
use BrewPHP\Http\Response;

function remember_theme(): Response
{
    return (new Response('Saved'))
        ->cookie(new Cookie('theme', 'dark', expires: '+1 year', secure: true));
}
```

| Argument | Default | Meaning |
|---|---|---|
| `name`, `value` | | name and value (the value is URL-encoded) |
| `expires` | `0` (session cookie) | timestamp, `strtotime()` string like `'+1 week'`, or `DateTimeInterface` |
| `path` | `'/'` | URL path the cookie applies to |
| `domain` | `null` | domain the cookie applies to |
| `secure` | `false` | only send over HTTPS |
| `httpOnly` | `true` | hide the cookie from JavaScript |
| `sameSite` | `'Lax'` | `'Lax'`, `'Strict'`, `'None'` (requires `secure`) or `null` |

Delete a cookie with `$response->deleteCookie('theme')`. Read cookies with
`$request->cookie('theme')`.

## Uploaded files

```php
use BrewPHP\Http\Request;

function upload_avatar(Request $request): string
{
    $avatar = $request->file('avatar');
    if ($avatar === null || !$avatar->isValid()) {
        return 'Upload failed: ' . e($avatar?->errorMessage() ?? 'no file');
    }
    if (!in_array($avatar->mimeType(), ['image/png', 'image/jpeg'], true)) {
        return 'Please upload a PNG or JPEG image.';
    }

    $path = $avatar->moveTo(base_path('media', 'avatars'), $avatar->hashName());

    return 'Saved to ' . e(basename($path));
}
```

| Member | Meaning |
|---|---|
| `isValid()` | the upload succeeded and is a real uploaded file |
| `errorMessage()` | a readable reason when it failed |
| `clientName`, `clientExtension()` | the name sent by the browser, and its extension |
| `clientMimeType` | the type sent by the browser (do not trust it) |
| `mimeType()` | the type detected from the file's contents |
| `size` | size in bytes |
| `hashName()` | a random file name keeping the extension, safe to store |
| `content()` | the file's contents |
| `moveTo($directory, $name)` | move it into a folder (created if needed); returns the new path |

Fields with several files, such as `photos[]`, are arrays of
`UploadedFile` in `$request->files()`; `file()` only returns single files.

!!! tip "Never trust the client's file name"
    Use `hashName()` (or your own name) when storing files. `moveTo()` keeps
    only the base name of the name you pass, so `../` tricks cannot escape
    the folder.

## HTTP exceptions

Throw these from anywhere to answer with an error page:

| Exception | Status |
|---|---|
| `BrewPHP\Http\BadRequest` | 400 |
| `BrewPHP\Http\PermissionDenied` | 403 |
| `BrewPHP\Http\Http404` | 404 |
| `BrewPHP\Http\MethodNotAllowed` | 405 (with an `Allow` header) |
| `BrewPHP\Http\HttpException` | any status: `new HttpException(410, 'Gone for good')` |

```php
use BrewPHP\Http\PermissionDenied;

throw new PermissionDenied('Only the author can edit this post.');
```

The message is shown on the production error page for `4xx` statuses, so
keep it user-friendly. (A 404 raised because no route matched shows the
generic "Page not found" instead; the list of patterns tried only appears
with `DEBUG` on.) `abort(404)` and `abort(403, 'Members only')` do the
same in one call. See [Error pages](../getting-started/configuration.md#error-pages)
to customise the pages.
