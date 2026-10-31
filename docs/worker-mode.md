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

## Development file watching

The worker example watches selected source extensions rather than entire
directories. Broad watches can restart workers when Git metadata, dependencies,
temporary files or generated builds change.

For local bundles, add explicit source paths to the worker block, for example:

```caddyfile
watch /var/www/html/packages/example-bundle/src/**/*.{env,php,twig,yaml,yml}
watch /var/www/html/packages/example-bundle/config/**/*.{env,php,twig,yaml,yml}
watch /var/www/html/packages/example-bundle/translations/**/*.{env,php,twig,yaml,yml}
```

Use concrete package directory names, following the application integration's
pattern. Do not watch the whole `packages/` tree. Changes outside these paths
or extensions require a manual worker restart; restart after changing process
environment variables as well.

## Pimcore Studio CSP state

The locally installed Studio UI `ContentSecurityPolicyHandler` retains
`allowedUrls`, `cspHeaderOptions` and a cached `nonce` in the service instance.
With a persistent kernel, allowed URLs can accumulate and the nonce can be
reused across requests. Recycling workers only limits the lifetime of this
state; it does not reset it between requests.

The application integration currently has an uncommitted workaround in
`src/EventListener/ResetCspHandlerSubscriber.php`, registered in
`config/services.yaml` with the `kernel.event_subscriber` tag. On the first main
request it captures the handler's baseline. On subsequent main requests it
restores the URL/header options, generates a fresh nonce and replaces the old
nonce in the script policy. It runs on `kernel.request` at priority `1024` and
skips subrequests.

This is application-side code, not something the Caddyfile can fix. The
subscriber uses reflection because the installed final handler has no reset
API. It depends on private property/method names and logs a warning on
`ReflectionException`; it does not stop the worker on that failure. Treat it
as a version-dependent workaround, recheck it after Studio updates, and remove
it when upstream provides request-safe state handling. This repository does
not ship the subscriber as a supported generic implementation.

Before enabling worker mode with Studio, verify repeated Studio responses on
the same worker: CSP headers must not grow, each response must get a fresh
nonce, and the header nonce must match the rendered script attributes. Also
check configured CSP sources and alternating authenticated users. These notes
document the local changes; they are not evidence of a completed load or
security test.

## Local runtime verification (2026-10-31)

Checked against the running application container with FrankenPHP 1.13.1,
PHP 8.5.11 and Caddy 2.11.7:

- 100 consecutive requests to `/pimcore-studio/` returned HTTP 200. CSP header
  length stayed at 683 bytes and all 100 response nonce sets were different.
- A further 100 responses contained 300 script nonce attributes; all matched
  a nonce in the response CSP header. The debug toolbar stylesheet had a
  separate nonce; the current style policy also allows inline styles.
- The CLI worker's effective Docker healthcheck was `Test: ["NONE"]`.
- The three locally configured terminal environment variables were present
  in the FrankenPHP container; their secret values were not printed.
- The example worker Caddyfile adapted successfully, but full validation
  failed while provisioning Mercure: the installed module requires the legacy
  JWT directives to move into an `issuer` block or explicitly opt into
  `protocol_version_compatibility 8`. The example is therefore not validated
  for this installed module version.

These HTTP checks used the application's running configuration, not a server
started with this repository's example. They did not identify individual PHP
workers, exercise authenticated users, test watcher restart behavior or verify
Mercure delivery. No running service was restarted or reconfigured.
