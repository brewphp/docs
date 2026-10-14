# Installation

## Create a project with Composer

The quickest way to start is the official skeleton:

```bash
composer create-project brewphp/skeleton myapp
cd myapp
```

Composer installs BrewPHP, copies `.env.example` to `.env` and runs
`php manage.php key:generate`, which writes a random `SECRET_KEY` to `.env`.

Start the development server:

```bash
php manage.php serve
```

```text
Performing system checks...
System check identified no issues.

BrewPHP development server at http://127.0.0.1:8000/
Quit the server with CTRL-C.
```

Open <http://127.0.0.1:8000/> and you will see the welcome page.

!!! tip "Another address or port"
    Pass an address, a port, or both: `php manage.php serve 8080` or
    `php manage.php serve 0.0.0.0:8000` (to reach the server from other
    devices on your network).

Finally, make sure everything is configured correctly:

```bash
php manage.php check
```

```text
✔ System check identified no issues.
```

## From a cloned repository

When you clone an existing BrewPHP project, `.env` is not in git (it holds
secrets), so create it yourself:

```bash
composer install
cp .env.example .env
php manage.php key:generate
```

## Global installer (planned)

A global installer is planned as a separate package, `brewphp/installer`,
providing the `brewphp` command:

```bash
composer global require brewphp/installer
brewphp new myapp
```

!!! note "Coming soon"
    The installer is not released yet; use `composer create-project` above.
    It will do exactly the same thing. The command is called `brewphp`, never
    `brew`, so it does not clash with Homebrew.

## Docker

If you prefer not to install PHP locally, run the project in a container.
Create a `Dockerfile` in the project folder:

```dockerfile title="Dockerfile"
FROM php:8.2-cli

RUN apt-get update \
    && apt-get install -y --no-install-recommends git unzip \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY . .
RUN composer install --no-interaction \
    && if [ ! -f .env ]; then cp .env.example .env && php manage.php key:generate; fi

EXPOSE 8000
CMD ["php", "manage.php", "serve", "0.0.0.0:8000"]
```

Build and run it:

```bash
docker build -t myapp .
docker run --rm -p 8000:8000 -e ALLOWED_HOSTS=localhost,127.0.0.1 myapp
```

Then open <http://127.0.0.1:8000/>.

!!! warning
    `manage.php serve` uses PHP's built-in server, which is for development
    only. The deployment guides (coming in a later release) cover production
    setups with nginx and PHP-FPM.

## Next steps

- Learn how a project is laid out in [Project structure](project-structure.md).
- See every setting in [Configuration](configuration.md).
- Map your first URLs in [Routing](../topics/routing.md).
