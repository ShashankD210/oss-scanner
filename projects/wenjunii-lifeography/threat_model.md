# Lifeography

Lifeography is a static interactive art website. Root HTML pages contain main
views and experiments, `scripts/` and `shape/` contain JavaScript, and local data
and media drive browser visualizations. There is no server-side application,
package manifest, build step, or documented automated test suite.

## Inputs and boundaries

Review query/hash parameters, browser events/messages, DOM insertion, external
resource loading, imported JSON/CSV, and locally bundled third-party libraries.
Focus on DOM/script injection through a realistic untrusted input, dangerous
URL schemes, prototype pollution reachable from data, and cross-origin trust
assumptions. Treat trusted, repository-authored creative content separately from
an attacker-controlled remote value. Read the existing README security notes
about pinned CDN scripts, SRI, UUID generation, and deep-merge protections.

## Offline environment

The complete checkout is `/src`. Serve it with:

```sh
python3 -m http.server 8000 --bind 127.0.0.1
```

Node.js and Chromium are installed for creating reproducers. Script/style URLs
linked directly from HTML are cached at `/opt/oss-scanner-web-cache/manifest.json`.
Each entry maps the original URL to local bytes and a SHA-256. Use browser request
routing or a local fixture to supply those bytes; the original pages and their
SRI values are unchanged. Third-party APIs, dynamically loaded resources, remote
fonts/media and live hosting are not reproduced by that cache. Do not claim full
page functionality merely because the static server responds. Build validation
is a source/cache/HTTP smoke check; there is no existing full browser test suite.

## Severity and reporting

Prioritize exploitable browser execution or information disclosure with a clear
attacker input and affected origin. Explain required navigation, interaction,
data source and browser behavior. A vulnerable vendor version alone needs a
reachable example in this project. Use a minimal local HTML/data fixture and
avoid altering original artworks, personal datasets, or historical assets.
Send reports privately rather than opening public vulnerability issues.
