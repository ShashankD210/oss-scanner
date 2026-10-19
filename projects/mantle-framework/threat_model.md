# Threat model

The scanner reads this file before it starts. It covers the Mantle framework monorepo.

## What this project does and where untrusted input enters

Mantle is a Laravel-inspired framework that WordPress sites and plugins build on. It sits between WordPress core and an application's own code, so its code runs inside public WordPress sites. Untrusted input reaches it through:

- HTTP requests handled by Mantle's router and HTTP kernel (`src/Mantle/Http/`): paths, route parameters, query strings, request bodies (form, JSON, multipart), uploaded files, cookies and headers. Model route binding resolves those values into database queries.
- REST API routes registered through Mantle (`src/Mantle/Http/Routing/`, REST route registrars).
- Values passed to the query builder (`src/Mantle/Database/Query/`) from request data: `where()`, `where_meta()`, `where_term()`, ordering and pagination arguments, which become `WP_Query` / `WP_Term_Query` / `$wpdb` queries.
- Template rendering in `src/Mantle/View/` and `src/Mantle/Framework_Views/` (Blade-style templates, escaping helpers).
- Serialized data the framework reads back: queued jobs (`src/Mantle/Queue/`), cache entries (`src/Mantle/Cache/`), and post / term meta written by other users.
- Outbound HTTP made by the HTTP client (`src/Mantle/Http_Client/`) with URLs that may come from user input (SSRF).

Trusted: the application's own code and configuration (`config/`, service providers), the environment, the file system, WordPress administrators with `unfiltered_html` / `manage_options`, and WP-CLI commands (`src/Mantle/Console/`), which only run with shell access.

## Components that matter most / least

Highest priority:

1. Routing, middleware and request handling in `src/Mantle/Http/`: authorization middleware bypass, route model binding exposing records the user cannot read, CSRF / nonce handling, open redirects.
2. SQL injection or query manipulation through `src/Mantle/Database/` (raw clauses, column / order-by names, meta and tax queries built from input).
3. XSS through `src/Mantle/View/` and the escaping helpers, and through blocks rendered by `src/Mantle/Blocks/`.
4. Object injection through `unserialize()` of queued jobs, cache entries or meta.
5. SSRF and header / URL injection in `src/Mantle/Http_Client/`.

Lower priority or out of scope:

- `src/Mantle/Testing/` and `src/Mantle/Testkit/`: the test framework, which runs only in development and CI. Shell commands it builds from the developer's own environment variables are not findings.
- Scaffolding templates and generators (`src/Mantle/Framework/resources/`, `make:*` console commands), which write code the developer then owns.
- `vendor/`, WordPress core and the plugins the test suite installs. Report those upstream.
- `tests/`, `bin/`, `tasks/` and the PHPStan, PHPCS and Rector configs.

## How to exercise it

- Run `mantle-start` first to start MariaDB. `mantle-stop` stops it.
- `vendor/bin/phpunit` from `/src` runs the suite; `vendor/bin/phpunit --filter <test>` or a path runs one test. `WP_MULTISITE=1` runs it under multisite. The bootstrap rsyncs `/src` into `/opt/wordpress/wp-content/plugins/mantle-framework` and runs PHPUnit there, so edits in `/src` take effect on the next run.
- Mantle's own testing framework is the best reproducer: tests extend `Mantle\Testing\FrameworkTestCase` (see `tests/`), can register routes, and can make requests with `$this->get( '/path' )->assertOk()`. A failing PHPUnit test in `tests/` is the preferred proof of concept.

## How you rate severity

- Critical: unauthenticated remote code execution, or unauthenticated SQL injection, in a default use of a framework API.
- High: authenticated (subscriber-level) SQL injection or code execution; authorization bypass that exposes or modifies other users' data; stored XSS that a low-privileged user can plant for an administrator.
- Medium: reflected XSS; SSRF; open redirect; bugs that need the application to pass untrusted input into an API whose docs say it must not.
- Low: hardening issues and information disclosure without direct impact.

If a bug can only happen when the application misuses an API in a way the docs warn against, say so and rate it at most Medium.

## How reports and patches should look

- Name the package (`src/Mantle/<Package>/`), file and method, the least privileged user who can trigger it, and a reproducer as a PHPUnit test.
- Patches are `git diff` against `/src`. Match the code style: `declare( strict_types=1 );`, snake_case methods, full types, WordPress escaping and sanitizing functions, and code that passes `composer phpstan` (level 8) and `composer phpcs`.

## Anything to leave alone

- Issues that need a WordPress administrator (`unfiltered_html`, `manage_options`) or shell access.
- Missing security headers and version disclosure.
- Findings in WordPress core or third-party packages unless Mantle's own code uses them in a way that creates the vulnerability.
