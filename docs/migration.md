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
