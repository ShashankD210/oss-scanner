# Threat model: PerfLocale

## What this project does and where untrusted input enters

PerfLocale is a multilingual plugin for WordPress (GPL-2.0-or-later, distributed on wordpress.org). It adds languages
to a site, routes every front-end request to a language, links each post, page, term and menu to its translations,
translates theme and plugin strings, can translate content through machine-translation providers, and integrates with
WooCommerce, page builders and SEO plugins. It runs on every front-end request of the sites that use it.

This enrollment covers the plugin as released: the `main` branch of https://github.com/perflocale/perflocale, which
holds the shipped tree (no tests or build tooling; there is no build step). Code lives in `src/` (PHP, PSR-4 under the
`PerfLocale\` namespace), `addons/` (integrations, each loaded only when its plugin or theme is active), `blocks/`
(the Language Switcher and "If language" blocks) and `assets/` (admin and front-end JavaScript and CSS).

Untrusted input enters through:

- Front-end requests from anonymous visitors: the language part of the URL (path prefix, subdomain or domain, and the
  query var), translated slugs and the redirects built from them (`src/Router/`, `src/Frontend/SlugRedirector.php`),
  the `perflocale_lang` cookie, the `Accept-Language` header and the geolocation redirect (`src/Router/GeoRedirect.php`;
  CDN country headers are honoured only from allow-listed proxy addresses), and everything printed into pages: the
  language switcher (widget, block, menu items, shortcode), `hreflang` tags, the `Content-Language` header,
  conditional content and the "If language" block (`src/Frontend/`).
- The REST API under `/wp-json/perflocale/v1/` (`src/Api/`): languages, translations, strings, jobs, XLIFF, machine
  translation, block translation, configuration and webhooks. Every route has a permission callback; a route an
  actor below its capability can reach is in scope.
- `admin-ajax.php`, `admin-post.php` and the admin screens, when reached by users who are not fully trusted.
- Content from lower-privileged users that is later rendered to visitors or to higher-privileged users: translated
  post content, titles, excerpts, terms and menu labels; block markup and attributes; string translations.
- String translations entered on PerfLocale's Strings screen (or through REST and XLIFF) by users holding
  `perflocale_translate`. PerfLocale compiles them into `.l10n.php` translation files that WordPress loads
  (`src/Strings/TranslationFileGenerator.php`). Unlike WordPress's own language packs, these strings come from site
  users and are untrusted: nothing in them may ever execute as PHP or reach a page unescaped.
- Files imported by users: XLIFF (`src/Xliff/`), PO files, and PerfLocale's own export/import format.
- Data PerfLocale reads from other multilingual plugins when migrating a site from WPML, Polylang or TranslatePress
  (`src/Migration/`): their tables and options were written by those plugins and by the site's users, not by
  PerfLocale.
- Responses from remote servers: machine-translation providers (DeepL, Google, Microsoft, LibreTranslate, an external
  agency, the WordPress AI client; `src/MachineTranslation/`), exchange-rate sources for WooCommerce multi-currency
  (`src/WooCommerce/ExchangeRateSync.php`), and webhook targets. Treat provider output as untrusted text.
- Outgoing requests to URLs a user supplies (webhook registration in `src/Api/WebhookController.php`).

### Actors

Severity depends first on who the attacker is. Say in every report which actor the finding needs, and whether it needs
a site option that is off by default (self-registration, a specific language URL mode, multi-currency, geolocation).

| Actor | How they get the account | Trust |
|-------|--------------------------|-------|
| Anonymous visitor | none needed | Untrusted. High band |
| Subscriber, WooCommerce customer | self-registration or checkout, when the site allows it | Untrusted. High band |
| Translator (`perflocale_translator`: edits others' posts and pages, uploads files, `perflocale_translate`, `perflocale_use_mt`; no `unfiltered_html`) | only an administrator can create them | Untrusted beyond these capabilities. Low band |
| Contributor, Author, Shop Manager | admin-created | Untrusted beyond their capabilities. Low band |
| Editor (single site) | admin-created | Trusted with raw HTML (`unfiltered_html`). XSS from an Editor is not rated; any other boundary an Editor crosses (PHP execution, file write, a capability check that should have stopped them) is in scope, rated in the low band |
| Administrator (single site) | admin-created | Fully trusted. Not rated |
| Administrator of a site on a multisite network | created by a Super Admin | Partly trusted: no `unfiltered_html`, no plugin or theme installs. Reaching either crosses a boundary |
| Super Admin (multisite) | n/a | Fully trusted |

**Not trusted, even though they look like configuration:** string translations and translated content written by
Translators and Authors, imported files, and anything read from another plugin's tables during a migration.

**Trusted:** PerfLocale's own code and its language packs from translate.wordpress.org, WordPress core, and the
settings only an Administrator can change (provider API keys, URL modes, proxy allow-lists).

## Components that matter most / least

Most important:

- Capability, nonce and REST permission checks, and `map_meta_cap` filters (`src/Api/`, `src/Admin/`): a lower actor
  reading or changing another user's private or draft content through a translation, a language, a job or a string.
- Routing and redirects (`src/Router/`, `src/Frontend/SlugRedirector.php`, language switcher URLs): open redirects,
  host or header injection, a request routed to content the visitor may not see.
- Output escaping in everything PerfLocale prints on the front end (`src/Frontend/`, `blocks/`, `addons/`).
- The `.l10n.php` generator and loader (`src/Strings/`): a string translation that executes, breaks out of its PHP
  array, or overwrites another file.
- Content PerfLocale writes on a user's behalf (creating a translation by copying the source, machine translation,
  block translation, Quick Edit, REST): any path that lets a user without `unfiltered_html` store markup that core's
  kses would have removed for them.
- SQL built for PerfLocale's own tables (`src/Database/`), including the SQLite code paths.
- Caching (`src/Cache/`, `src/Frontend/CacheTagEmitter.php`): a visitor causing another language's page, or another
  user's data, to be cached and served to others.
- WooCommerce: prices, currency and stock across languages (`src/WooCommerce/`): a customer paying a different amount
  than the store configured, or changing another product's stock.
- Imports (`src/Xliff/`, PO import, the export/import format, `src/Migration/`): XML and file parsing, path handling,
  and imported values reaching pages unescaped.
- Outgoing requests to user-supplied URLs (webhooks): server-side request forgery.

Less important but in scope: the `addons/` integrations (each runs only with its plugin or theme active), admin-only
settings screens, WP-CLI commands (run by the server operator, so only a path that mixes in untrusted stored data
matters).

## How to exercise it

The image (see the Dockerfile header) has WordPress with PerfLocale active, English (default), German and French,
pretty permalinks, a published page and post, and one user per role including a Translator. Start it with `wp-start`.

- Front end: `http://127.0.0.1/` (English), `http://127.0.0.1/de/`, `http://127.0.0.1/fr/`.
- REST: `http://127.0.0.1/wp-json/perflocale/v1/` (log in with the users above; application passwords work with
  `wp user application-password create`).
- WP-CLI: `wp --path=/var/www/html perflocale --help` (languages, translations, strings, export, import, and more).
- WooCommerce is installed but inactive: `wp --path=/var/www/html plugin activate woocommerce`.
- The plugin code is `/src`, symlinked into `wp-content/plugins/perflocale`, so a patch in `/src` is live at once.

## How you rate severity

- Anonymous visitor or customer: stored or reflected XSS, SQL injection, PHP execution, reading private content, an
  open redirect usable for phishing, cache poisoning served to others, or a price or stock change: high, and critical
  when it leads to code execution or account takeover.
- Translator, Author or Contributor: crossing a capability boundary (stored XSS without `unfiltered_html`, PHP
  execution through `.l10n.php`, editing or reading content their role does not allow, privilege escalation): medium,
  high when it reaches code execution or an administrator account.
- Denial of service: an anonymous request that exhausts memory or time on every page: medium; the same from a
  logged-in lower actor against their own save: low.
- Server-side request forgery through webhooks: rate by who can register a webhook (by default an Administrator, so
  not rated unless a lower actor can reach it).

## Anything to leave alone

- Findings that need an Administrator on a single site (or a Super Admin), or a malicious PerfLocale language pack.
- Bugs in WordPress core, WooCommerce or another plugin that PerfLocale does not cause or worsen.
- Missing security headers, rate limiting, or other hardening that WordPress leaves to the site.
- Self-XSS, and issues that need the attacker to already control the server, the database or a provider's API.
- Timing or performance differences that are not a denial of service.
