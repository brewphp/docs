# Requirements

## PHP

BrewPHP needs **PHP 8.2 or newer**. It is tested on PHP 8.2, 8.3 and 8.4.

Check your version:

```bash
php -v
```

## Extensions

The framework core has no runtime dependencies beyond PHP itself. These
extensions are bundled with almost every PHP build and are recommended:

| Extension | Used for | Required? |
|---|---|---|
| `json`, `ctype`, `filter` | Core features | Always enabled in PHP 8.2+ |
| `mbstring` | Correct widths and truncation of non-ASCII text | Recommended |
| `fileinfo` | Detecting the real MIME type of uploads | Recommended |
| `iconv` | Transliteration in `Str::slug()` | Recommended |
| `openssl` / `sodium` | Cryptography (sessions and signing, v0.5) | Recommended |
| `pdo` + a driver | Databases (v0.2) | When you use the ORM |

`php manage.php about` shows which ones are available:

```text
Extensions
  PDO drivers ........................... mysql, pgsql, sqlite
  mbstring ............................................... yes
  intl ................................................... yes
  openssl ................................................ yes
  sodium ................................................. yes
  OPcache ................................................ off
```

## Composer

Projects are created and updated with [Composer](https://getcomposer.org/) 2.

```bash
composer --version
```

## Databases

The ORM arrives in v0.2 and will support:

| Database | Minimum version | PDO driver |
|---|---|---|
| SQLite | 3.35 | `pdo_sqlite` |
| MySQL / MariaDB | 8.0 / 10.6 | `pdo_mysql` |
| PostgreSQL | 13 | `pdo_pgsql` |

SQLite is the default for new projects because it needs no server.

## Web server

For development, `php manage.php serve` runs PHP's built-in server. In
production, use any web server that can run PHP (nginx with PHP-FPM, Apache,
Caddy...) with the document root set to the project's `public/` folder.
