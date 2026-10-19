# Threat model

The scanner reads this file before it starts. It covers Burst Statistics, the free, privacy-friendly analytics plugin
for WordPress (wordpress.org/plugins/burst-statistics). The repository is generated from the maintainers' private
Pro repository with the `Pro/` directory left out, so code that references `Burst\Pro\…` is inert here unless a guard
is missing (see below).

## What this project does and where untrusted input enters

Burst records page views, sessions, goals and e-commerce conversions in its own database tables
(`wp_burst_*`) and shows them in a React dashboard inside wp-admin. It runs on every front-end page of the sites that
use it, so treat all of the following as untrusted:

- **The tracking beacon.** Every visitor's browser sends hits to `/src/endpoint.php` (a standalone file that loads
  WordPress with `SHORTINIT`) or, as a fallback, to the REST route `burst/v1/track`. The payload (URL, referrer,
  user agent, screen size, UID / fingerprint, goals, parameters, timings) is fully attacker-controlled and is
  written to the statistics tables and later rendered in the dashboard. Any anonymous visitor can send it.
- **`wp_ajax_nopriv_burst_tracking_error`**, an unauthenticated AJAX action.
- **Values that come back out of the statistics tables.** Page URLs, referrers, UTM parameters, search terms,
  device and browser strings stored by the beacon are later shown to administrators. Stored XSS from a visitor
  into wp-admin, the dashboard widget, email reports or a shared dashboard is in scope.
- **Shared dashboards.** A `burst_share_token` in the URL grants read access to a subset of statistics without a
  WordPress login.
- **The admin REST API** (`burst/v1/...`): settings, goals, data tables, reports, import uploads, archive and
  export, MainWP authentication.
- **Imports.** Uploaded import files (chunked upload, then finalize) are parsed into the statistics tables.

### Where the code is

| Path | What it is |
| --- | --- |
| `/src` | The burst-statistics checkout (branch `master`). Patches belong here. |
| `/wordpress` | A WordPress install (latest stable) served by Apache on MariaDB, with Burst active. `/var/www/html` links to it. |
| `/wordpress/wp-content/plugins/burst-statistics` | A symlink to `/src`, so a patch in `/src` is live immediately. |

### Trust model

- **Anonymous visitors** are untrusted. They can send any beacon payload, any number of times.
- **Logged-in users** must stay within their capabilities. Burst adds three: `view_burst_statistics` (read the
  dashboard), `view_sales_burst_statistics` (read e-commerce / revenue data) and `manage_burst_statistics`
  (change settings, goals, import, export, archive). Administrators get all three on activation; other roles get
  none by default.
- **A share token holder** may read only what that share link exposes, and nothing after it expires or is revoked.
- Administrators with `manage_options` or `unfiltered_html`, `wp-config.php`, the file system and the database are
  trusted. An issue that needs one of these is not a finding unless it crosses into something WordPress itself
  does not already allow (for example, reading another site's data on multisite).

## Components that matter most / least

Highest priority:

1. **The beacon path**: `endpoint.php`, `includes/Frontend/Tracking/`. SQL injection, PHP errors or fatals an
   anonymous visitor can trigger, writes outside the statistics tables, and anything that lets a visitor read data.
2. **Authorization on REST and AJAX routes**: a `permission_callback` that is missing, `__return_true` for admin
   data, checks the wrong capability (sales data must require `view_sales_burst_statistics`), or trusts request
   data. State-changing routes also need a nonce. `has_admin_access()` is a load gate, not an authorization check.
   Data requests are meant to resolve through `data/datatable/{id}` with a per-table capability
   (`App::get_datatable_config()`); a way to query arbitrary metrics or tables is a finding.
3. **SQL injection** in the statistics query builder (`Statistics_Query`, `Query_Executor`, filters, group-by,
   order-by and metric names): table and column names cannot be parameterized and must come from an allowlist.
4. **Stored XSS** from tracked data into wp-admin, the dashboard widget, email reports and shared dashboards.
5. **Share tokens**: guessable or leaked tokens, access beyond the shared scope, tokens that keep working after
   expiry.
6. **Import and export**: path traversal or arbitrary file write in the import upload, export files reachable
   without authorization, and any secret (share tokens, license or API tokens, `wp_options`) ending up in an export.
7. **Generated files in `wp-content/uploads/burst/`** (the tracking script and its settings): anything that lets a
   visitor or a low-privilege user write or execute code there.

Lower priority or out of scope:

- Third-party code: `vendor/`, `lib/`, and anything bundled under `assets/` or `includes/Admin/App/build/` that is
  not Burst's own source. Report these upstream unless Burst's own use creates the vulnerability.
- Built JavaScript (`*/build/`): report against the source in `includes/Admin/App/src/` instead.
- Tests (`tests/`), `automation/`, and the CI and docker-compose files.
- The WooCommerce and Easy Digital Downloads plugins themselves. Burst's integrations with them
  (`includes/Integrations/`) are in scope.

## How to exercise it

- Run `wp-start` first. It starts MariaDB and Apache and is safe to run again. The site is at `http://127.0.0.1/`;
  `wp-stop` stops both.
- Users: `admin` / `admin` (administrator, all Burst capabilities), and `editor`, `author`, `contributor`,
  `subscriber` with the password equal to the name. Anonymous needs no login.
- `wp --path=/wordpress …` runs WP-CLI (as root, which is allowed). Useful: `wp option get burst_options_settings`,
  `wp db query "SELECT * FROM wp_burst_statistics ORDER BY ID DESC LIMIT 5"`,
  `wp user add-cap subscriber view_burst_statistics` to give a low-privilege user dashboard access.
- A beacon hit can be sent with `curl -X POST http://127.0.0.1/wp-content/plugins/burst-statistics/endpoint.php`
  and a JSON body shaped like the one `assets/js/src/burst.js` sends.
- WooCommerce and Easy Digital Downloads are installed but inactive: `wp plugin activate woocommerce`.
- `vendor/bin/phpunit` (from `/src`) runs the PHPUnit suite against the running site; `wp-start` must run first.
- PHP errors go to Apache's error log on stderr, not into responses.

## How you rate severity

- **Critical**: unauthenticated SQL injection, remote code execution, or arbitrary file write; unauthenticated
  stored XSS that runs in an administrator's session in a default configuration.
- **High**: an unauthenticated visitor or a subscriber reading statistics or settings they should not see
  (including sales data and share tokens); privilege escalation to `manage_burst_statistics`; SQL injection that
  needs `view_burst_statistics`; a share token that grants more than its scope.
- **Medium**: CSRF on a state-changing route; stored XSS that needs `manage_burst_statistics`; a sales-data leak to
  a user with `view_burst_statistics` but not `view_sales_burst_statistics`; a single unauthenticated request that
  causes a fatal error or an expensive query.
- **Low**: information disclosure without personal or business data; issues that need an uncommon configuration.
- Polluting the statistics with fake hits is the nature of client-side analytics and is not a finding on its own;
  it is a finding when the payload escapes its fields (SQL, XSS, writes to other tables) or bypasses bot filtering
  by design flaw rather than by spoofing a browser.

## How reports and patches should look

- Name the affected file and function, the least privileged role or token that triggers it, and a self-contained
  reproducer (WP-CLI setup plus `curl`, or a PHPUnit test).
- Write patches as `git diff` against `/src`, following the WordPress Coding Standards and WordPress APIs
  (`$wpdb->prepare()`, `current_user_can()`, `wp_verify_nonce()`, `esc_html()` / `esc_attr()` / `esc_url()`,
  `sanitize_*()`), with an allowlist for identifiers that cannot be prepared.

## Anything to leave alone

- Do not suggest replacing `navigator.sendBeacon` with `fetch()` in `assets/js/src/burst.js`. `sendBeacon` is
  deliberate: many JavaScript-rendering crawlers run `fetch()` but not `sendBeacon()`, so the change lets bots count
  as visitors.
- Missing security headers, version disclosure in `readme.txt`, and rate limiting of the beacon in general.
- Findings that need `display_errors`, `WP_DEBUG_DISPLAY` or `BURST_DEBUG` to be switched on.
- References to `Burst\Pro\…` classes behind a `class_exists()`, `file_exists()` or `is_pro()` guard: they are inert
  in this repository. An *unguarded* call to a `Burst\Pro\…` symbol is a real bug (it fatals), but report it as a
  reliability issue, not a security one, unless an anonymous request reaches it.
