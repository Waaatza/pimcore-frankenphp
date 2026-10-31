# Migration from the Pimcore skeleton

This guide describes the infrastructure delta from `pimcore/skeleton` 2026.x.
Apply it to a clean skeleton first, then merge it into an existing project.

## 1. Preserve the application baseline

FrankenPHP does not require changes to Pimcore's `public/index.php`,
`src/Kernel.php`, Composer Runtime integration or bundle registration. Keep
those files aligned with the skeleton unless the application has an unrelated
reason to change them.

## 2. Replace the web/PHP services

In `docker-compose.yaml`:

1. Remove the `nginx` service.
2. Replace the `php` service with the `frankenphp` service shown in the example.
3. Remove the standalone `mercure` service if the built-in hub is used.
4. Change `test-php` inheritance from `php` to `frankenphp`, if test services
   from the skeleton are retained.
5. Keep `supervisord`; HTTP workers do not replace Messenger consumers,
   scheduled tasks or Pimcore maintenance jobs.

## 3. Add the image and Caddy configuration

Copy `.docker/frankenphp/` into the application. Start with classic mode.
The Caddyfile carries across Pimcore-specific behavior that is easy to miss:

- the document root is `public/`;
- all dynamic requests fall back to `public/index.php`;
- cache-buster URLs are rewritten;
- generated assets and thumbnails can be served from `public/var`;
- PHP below upload directories is never executed;
- dotfiles, source files and backups are not public;
- uploads are allowed up to the configured PHP and Caddy limits.

## 4. Configure Mercure

The example uses FrankenPHP's built-in Mercure module. Generate a strong JWT
secret and provide it to the container without committing it. Symfony/Pimcore
needs two URLs:

- browser-facing: `http://localhost/.well-known/mercure` for local development;
- container-facing: `http://frankenphp/.well-known/mercure`.

Exact variable names can differ with the installed Mercure bundle. Retain the
names used by the current Pimcore skeleton and change only their URL values.

## 5. Verify

```bash
docker compose config
docker compose build --pull frankenphp
docker compose up -d
docker compose exec frankenphp php -m
docker compose exec frankenphp bin/console pimcore:system:requirements:check
docker compose exec frankenphp bin/console cache:clear
```

Manually verify:

- frontend and Pimcore Studio login;
- assets, thumbnails and cache-buster URLs;
- a large upload near the configured limit;
- Mercure updates;
- Messenger consumers and scheduled tasks;
- `/var/assets/example.php` is not executable;
- dotfiles and backup suffixes return 404.

## Optional system tools

The official `pimcore/pimcore:*max*` image contains more media tools than the
minimal Dockerfile here. Add only what the application uses, such as FFmpeg,
Ghostscript, LibreOffice, Chromium, Poppler, Inkscape, pngquant, optipng,
jpegoptim, ExifTool or Graphviz. Run Pimcore's requirements check after every
image change.

## Local environment and CLI worker healthchecks

The application integration now loads `.env.local` into the FrankenPHP
container using Compose:

```yaml
services:
    frankenphp:
        env_file:
            - .env.local
```

Create this untracked file before starting the service. This passes variables
to the container process, including auxiliary processes started alongside
FrankenPHP. Symfony's dotenv loading alone does not configure those processes.
Explicit `environment` entries take precedence over values from `env_file`;
remove redundant entries when the file should supply their values. Keep
project-specific variables in the application repository.

Service-level `env_file` does not supply Compose `${VARIABLE}` interpolation.
For the example's interpolated values, use a shell environment, `.env`, or
`docker compose --env-file .env.local …` as appropriate.

If a separate Messenger/Scheduler service reuses a FrankenPHP image with an
HTTP healthcheck, its CLI command does not start Caddy. The application
integration disables that inherited check on its CLI worker:

```yaml
healthcheck:
    disable: true
```

Apply this only to CLI services using that image. Keep the HTTP service's
healthcheck enabled. Disabling the probe does not monitor consumer progress;
use a consumer-specific check if that is required. The standalone supervisord
image in this repository's Compose example does not need this override.
