# Threat model

## Purpose and untrusted inputs

Decipher MS is an Astro SSR site deployed to Cloudflare Workers with static assets. Its public `POST /api/briefing` accepts JSON describing a prospective engagement, verifies a Turnstile token, inserts the submission into D1, and schedules email delivery through Microsoft Graph. Public OIDC discovery and JWKS endpoints support an Entra federated identity. Worker and browser code also emit Application Insights telemetry.

Untrusted inputs include JSON bodies, submitted names/email addresses/topics/details, Turnstile tokens, HTTP headers and URLs, and upstream verification/token responses. Review `src/lib/server/briefing.ts`, `graph.ts`, `jwt.ts`, `oidc.ts`, `telemetry.ts`, the API routes, middleware, form component, and client telemetry. D1 inserts use prepared bound statements; verify that property across changes. Public JWKS must never expose private RSA material. The private signing key and Turnstile secret are runtime bindings, not build inputs.

## Security boundaries and severity

The important boundaries are unauthenticated requests to validation/Turnstile, validated data to persistent D1 storage, the Worker signing key to Entra/Graph authorization, user content to email HTML/reply-to fields, and request/form-derived data to telemetry. High-severity findings include private-key disclosure, unauthorized mail/identity use, SQL injection, stored script injection with a meaningful execution context, or access to confidential submissions. Assess denial of service, validation bypass, and abuse amplification by demonstrated impact. Treat telemetry privacy leakage according to the sensitivity actually exposed; public instrumentation identifiers and intentionally public discovery/JWKS metadata alone are not secrets.

## Image contents and offline exercise

The image retains the complete checkout at `/src`, all locked npm dependencies including development tools, Wrangler/Miniflare/workerd, and the Astro build in `dist/`. Node 24 satisfies the locked Astro minimum (22.12) and Wrangler minimum (22). The build downloads self-hosted Google font assets while networking is available; do not rebuild offline assuming that provider can fetch fonts. No `.dev.vars` file or real secrets are supplied. `public/` and `src/assets/` remain available for source audit.

There is no existing test script or tracked test suite. These are proposed post-build checks, not claimed executed results:

```sh
cd /src
npm run lint
./node_modules/.bin/wrangler dev --config dist/server/wrangler.json --local --ip 127.0.0.1 --port 8787
```

In a second shell in the same container, check the local HTTP boundary without a browser (browser scripts load external Turnstile/telemetry):

```sh
curl --fail --silent --show-error http://127.0.0.1:8787/ >/tmp/decipher-home.html
curl --fail --silent --show-error http://127.0.0.1:8787/.well-known/openid-configuration
code=$(curl --silent --show-error -o /tmp/decipher-invalid.json -w '%{http_code}' -H 'Content-Type: application/json' --data '{' http://127.0.0.1:8787/api/briefing)
test "$code" = 400
node --input-type=module -e 'import fs from "node:fs"; import assert from "node:assert/strict"; assert.equal(JSON.parse(fs.readFileSync("/tmp/decipher-invalid.json", "utf8")).error, "Invalid JSON")'
code=$(curl --silent --show-error -o /tmp/decipher-empty.json -w '%{http_code}' -H 'Content-Type: application/json' --data '{}' http://127.0.0.1:8787/api/briefing)
test "$code" = 400
node --input-type=module -e 'import fs from "node:fs"; import assert from "node:assert/strict"; assert.equal(JSON.parse(fs.readFileSync("/tmp/decipher-empty.json", "utf8")).error, "name required (<=120 chars)")'
```

Use a container with `--network none`; loopback remains available. Localhost resolves to the source's development telemetry environment. Invalid JSON and empty submissions return before Turnstile, D1, or Graph calls. Do not request JWKS without an isolated synthetic key fixture: it needs `OIDC_PRIVATE_KEY`. Local D1 schema can be inspected with `sqlite3 :memory: < schema.sql`; inspect the separate migrations as well rather than applying schema plus migrations twice.

## Offline limitations and prohibited side effects

A valid submission still fetches Turnstile Siteverify, even with the development dummy token configuration. Successful persistence schedules Graph email; there is no development email no-op. Full Turnstile, D1 service behavior, Entra token exchange, Graph delivery, deployed bindings/hostnames and Azure telemetry ingestion cannot be validated offline. Those boundaries remain available for complete source audit and isolated synthetic mocks, but this enrollment does not claim live integration coverage. No Stripe integration was found in the inspected manifest/routes.

Do not use real signing keys, tokens, email addresses, live intake submissions, remote D1, cloud logins, Wrangler deployment/upload/migration commands, or `scripts/preview-deploy.sh`. That script performs remote migration and upload. Production infrastructure and third-party services are outside the offline exercise; findings should target this repository's code, not the external services.
