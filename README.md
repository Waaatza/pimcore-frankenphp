> [!WARNING]
> Everything in this repository is experimental and has not been used on production systems. Do not use it in production.

# Pimcore on FrankenPHP

This repository documents the infrastructure-only changes required to run a
project created from [`pimcore/skeleton`](https://github.com/pimcore/skeleton)
with FrankenPHP instead of Nginx and PHP-FPM.

The examples target the `2026.x` skeleton line, PHP 8.5 and FrankenPHP 1.x.
They deliberately contain no application bundles, data object definitions,
customer names, domains, credentials or other project-specific material.

## What changes

The official skeleton runs separate `nginx`, `php` and `mercure` services.
The example in this repository replaces those three services with one custom
`frankenphp` service:

```text
Browser -> FrankenPHP/Caddy -> Pimcore public/index.php
                    |
                    +-------> built-in Mercure hub
```

MariaDB, Redis, RabbitMQ, OpenSearch and the Pimcore messenger workers remain
separate services. This keeps the migration small and follows the topology of
the upstream skeleton.

## Files to copy

From this repository, copy:

```text
docker/frankenphp/       -> .docker/frankenphp/
examples/docker-compose.yaml -> docker-compose.yaml
.env.example             -> use as a reference for your local environment
```

Do not overwrite a real `.env`. Transfer only the missing variable names and
generate fresh secrets.

## Start a development environment

```bash
cp .env.example .env.local
docker compose build frankenphp
docker compose up -d
docker compose exec frankenphp composer install
docker compose exec frankenphp bin/console pimcore:system:requirements:check
```

For a new skeleton, run its normal Pimcore installer after the containers are
healthy. The application remains available at `http://localhost`, and Pimcore
Studio at `http://localhost/pimcore-studio`.

## Modes

- `docker/frankenphp/Caddyfile` uses classic mode. It boots Pimcore for every
  request and is the recommended migration baseline.
- `docker/frankenphp/Caddyfile.worker` enables Symfony Runtime worker mode. Use
  it only after completing the [worker-mode checklist](docs/worker-mode.md).

Switch to the worker example by changing the Caddyfile mount in Compose.

## Production

The included Compose file is a development reference, not a production image.
For production, build an immutable image containing the application and its
Composer dependencies, run background workers separately, use managed secrets,
pin image versions or digests, configure real TLS hostnames, and size PHP
threads/workers against available memory.

See the [migration notes](docs/migration.md).

## Upstream references

- [Pimcore skeleton, branch 2026.x](https://github.com/pimcore/skeleton/tree/2026.x)
- [Pimcore system requirements](https://docs.pimcore.com/platform/Getting_Started/Installation/System_Requirements/)
- [FrankenPHP migration guide](https://frankenphp.dev/docs/migrate/)
- [FrankenPHP worker mode](https://frankenphp.dev/docs/worker/)

## License

The original material in this repository is available under the MIT License.
Pimcore and FrankenPHP remain subject to their respective licenses.
