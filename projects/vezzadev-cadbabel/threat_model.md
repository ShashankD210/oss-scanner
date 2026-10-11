# Threat model

## Purpose and untrusted inputs

Cadbabel is a TypeScript Cloudflare Worker serving a static landing page and anonymous event/reservation APIs. D1 stores events and reservation records. A reservation may create a Stripe Customer and SetupIntent to collect a card on file; the inspected server has no charge/PaymentIntent operation. Stripe Elements handles card entry rather than raw card data passing through this Worker. This repository is not a CAD file parser/converter; its current attack surface is the reservation/measurement site.

Untrusted inputs include HTTP bodies/headers, event enums, reservation email/purpose/direction/date and disclosure fields, reservation and SetupIntent identifiers, raw webhook payloads/signature headers, and Stripe responses. Focus on `src/api/http.ts`, `router.ts`, `events.ts`, `reserve.ts`, `stripe.ts`, `webhook.ts`, `stats.ts`, `src/index.ts`, `public/app.js`, database migrations, and Wrangler environment configurations. SQL uses bound values; verify this throughout. Email syntax and client disclosure acknowledgements do not establish identity or verified consent.

## Security boundaries and severity

Important boundaries are unauthenticated clients to D1 writes; public reservation IDs/email values to reservation recovery and card-state changes; the Stripe secret to Customer/SetupIntent operations; signed webhook payloads to database updates; and the stats bearer token to aggregate access. High-severity issues include secret disclosure, injection exposing reservation data, unauthorized meaningful card-state changes, or unauthorized use of Stripe credentials. Evaluate response/body exhaustion, rate-limit bypass, stored browser injection and reservation privacy leaks by demonstrated impact. Public publishable keys, account/database identifiers, intentionally public assets and aggregate public-event inputs alone are not secrets.

Audit webhook HMAC over the exact raw body, timestamp acceptance, reservation association and replay/idempotency behavior. Confirm stored SetupIntent/customer relationships before card-state changes. Check streamed as well as declared body-size handling: a post-read length rejection does not prove bounded memory usage. Configured event/reservation rate limits are abuse controls, not authentication. Inspect the missing-binding behavior and the trust in `cf-connecting-ip`; offline fakes do not prove live platform limits.

## Build and offline exercise

The image retains `/src`, npm's locked development dependencies, native esbuild/workerd binaries, Wrangler and the Workers Vitest pool runtimes. Node 24 follows `package.json`'s `>=24` requirement and CI (the root lock metadata still says `>=22`). There is no package build script: the Dockerfile follows CI's `wrangler deploy --dry-run --config wrangler.ppe.jsonc`, adding an output directory at `/src/scanner-build`. Dry-run bundles without deploying or validating remote bindings. No `.dev.vars` or real secrets are supplied or required.

`public/` includes the HTML/JS/CSS/icons and bundled Archivo WOFF2 font; these assets remain available offline. Do not invoke `scripts/subset-font.sh`, which downloads fonts and regenerates the tracked asset.

Existing post-build offline commands (not executed during enrollment preparation):

```sh
cd /src
npm run typecheck
npm test
```

Run those in a container with `--network none`. `test/api.test.ts` disables external fetch connections and stubs Stripe using fetchMock. `test/platform.test.ts` exercises local `SELF.fetch`, local D1 migration fixtures, assets, routing, validation, stats authentication and fake limiter bindings; Stripe secrets are absent in its platform environment. `vitest.config.ts` provides a synthetic stats token. Tests use local state, not remote databases, and need no Cloudflare or Stripe credentials. They test limiter dispatch/keying, not the production configured numeric thresholds: the pool's older bundled Wrangler does not support those configured rate-limit bindings.

For a separate local static/authorization smoke, without any secret file:

```sh
cd /src
./node_modules/.bin/wrangler dev --config wrangler.ppe.jsonc --local --ip 127.0.0.1 --port 8787
```

In a second shell inside the same network-isolated container:

```sh
curl --fail --silent --show-error http://127.0.0.1:8787/ >/tmp/cadbabel-home.html
code=$(curl --silent --show-error -o /tmp/cadbabel-stats.json -w '%{http_code}' http://127.0.0.1:8787/api/stats)
test "$code" = 401
code=$(curl --silent --show-error -o /tmp/cadbabel-method.json -w '%{http_code}' http://127.0.0.1:8787/api/stripe/webhook)
test "$code" = 405
```

Use HTTP clients rather than loading Stripe.js in a browser. The existing tests are the more comprehensive source/runtime exercise, including ephemeral D1 migrations and synthetic Stripe calls; the minimal smoke does not migrate or submit reservations.

## Live limitations and prohibited effects

Offline tests cannot prove deployed Cloudflare bindings/domains/D1 behavior, actual rate-limit enforcement, Stripe Customer/SetupIntent creation or status, browser Stripe Elements handling, real webhook delivery, or validity of production/PPE secrets. Those paths remain fully available for source audit and isolated fixture execution; no live integration success is claimed. Do not use real customer data, Stripe keys/cards, Cloudflare tokens, live domain requests, remote migration/deploy scripts, or Stripe API calls, even when a config describes test mode. Do not perform billing, email, secret retrieval or infrastructure changes. Deploy workflows and third-party service code are contextual boundaries; this enrollment exercises this repository locally only.
