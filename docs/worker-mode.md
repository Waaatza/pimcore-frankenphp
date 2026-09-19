# Worker mode checklist

Classic mode is the safe migration target. Worker mode keeps the Pimcore kernel
and service container alive across requests, so request state can accidentally
leak even when Symfony resets its own services.

Before enabling `Caddyfile.worker`:

- load-test the real request mix in a non-production environment;
- audit custom singletons, static properties, global variables and in-memory
  caches for request-specific state;
- implement `Symfony\Contracts\Service\ResetInterface` where a custom service
  retains request state;
- ensure code never writes request-specific values to `$_ENV` or process-wide
  state;
- test authentication, locale switching, permissions and tenant/site context
  across alternating users;
- monitor memory growth and restart behavior over thousands of requests;
- set the worker count from measured CPU and memory usage rather than copying
  the example value;
- keep `max_requests` finite as a safety valve;
- remove development `watch` directives in immutable production images.

If any check is uncertain, remain in classic mode. FrankenPHP still replaces
Nginx/PHP-FPM and provides Caddy features without requiring long-lived workers.
