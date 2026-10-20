# Threat model: UnLostPaws web app

UnLostPaws is a public lost-and-found pet platform. Next.js (App Router, plain JavaScript), MongoDB
(Mongoose), Redis (Upstash) job queue, Qdrant vector DB, S3/R2 storage, better-auth sessions, Stripe
subscriptions, Prelude phone OTP, Cloudflare Turnstile. A separate Python worker
(the-dot-squad/unlostpaws-worker, enrolled separately) does image ML and calls back over HTTP.

## What this project does and where untrusted input enters
- Anyone on the internet: public pages, `/api/listings/*`, `/api/media/*`, `/api/og`, `/api/geocode/reverse`,
  `/api/csp-report`, the public Digital Collar tag page `/tag/[publicId]` (contact details hidden behind Turnstile).
- Signed-in users: listing and pet creation (free text, coordinates, phone numbers, photos), image upload
  (`/api/upload/presign`, `/api/upload/put`), server actions in `src/lib/actions/`, profile and handle changes.
- Staff and admins: moderation, settings, user bans, gifting Premium (`src/lib/actions/admin.js`).
- Machine callers authenticated by a shared secret: `/api/webhooks/vision` (worker callback),
  `/api/cron/*` (Vercel cron, Bearer `CRON_SECRET`), `/api/stripe/webhook` (Stripe signature).

## Components that matter most
- Authentication and authorization: `src/lib/auth/` and `src/lib/actions/`. Every server action and API route must
  check session, ban status, role (`admin` vs staff) and ownership (`require-owned.js`). Missing checks are the top finding class.
- Webhook and cron auth in `src/lib/request-metadata.js` (`safeEqual`, `rejectInvalidBearer`,
  `hasValidInternalSecret`) and `src/lib/api/cron.js`.
- Browser-only API guard `rejectCrossSiteRequest` and the middleware in `src/proxy.js`.
- Upload path: key validation, MIME and magic-byte checks (`src/lib/storage/images.js`), per-user quotas, path traversal in storage keys.
- ML callback handling in `src/lib/intelligence/callbacks/`: the payload decides listing approval and what is
  written to Qdrant and MongoDB.
- Stripe webhook to Premium state (`src/lib/stripe/`), Turnstile verification (`src/lib/turnstile.js`),
  rate limiting (`src/lib/rate-limit/`), public ID encoding (`src/lib/public-id/`), HTML sanitization (DOMPurify).

## Less important or out of scope
- Third-party services themselves (MongoDB, Redis, Qdrant, S3, Stripe, Prelude, Cloudflare, Telegram).
- Translations in `messages/`, static assets in `public/`, `scripts/` (operator-run maintenance scripts that need `.env.local`).
- Self-inflicted issues from operators running with the documented `dev-` default secrets. Production startup already rejects them.
- The Python worker is a separate project; only report its interface to this app (the callback contract).

## How to exercise it
- `npm ci` has already run. The app needs MongoDB, Redis, S3 and other services that are NOT available offline,
  and `next build` fails without a database. Prefer reading routes and server actions, and reproduce with small
  Node scripts that import pure helpers (for example `src/lib/request-metadata.js`, `src/lib/validation.js`,
  `src/lib/storage/images.js`, `src/lib/text.js`) and mock `@/config/env`.
- Required env for any import of `src/config/env.js`: `DATABASE_URL`, `BETTER_AUTH_SECRET` (>=32 chars),
  `WEBHOOK_SECRET`, `CRON_SECRET`, `PUBLIC_ID_SALT`.

## How I rate severity
- Critical: unauthenticated or low-privilege access to admin functions, account takeover, auth bypass,
  forged Stripe or vision webhooks that change data, RCE, or reading or writing arbitrary storage objects.
- High: IDOR on listings, pets, tags or user data; access to hidden contact details without passing Turnstile;
  mass assignment of `role`, `premiumStatus` or ban state; stored XSS in any page rendered to other users;
  SSRF reachable by an ordinary user; bypassing bans or quotas at scale.
- Medium: reflected XSS or open redirect needing user interaction, CSRF on state-changing routes (the
  `rejectCrossSiteRequest` guard is the intended defense), rate-limit bypass, information disclosure of
  non-sensitive data, NoSQL operator injection with limited impact.
- Low: missing hardening headers, verbose errors, timing differences on non-secret data.
- Denial of service is Medium at most and only if unauthenticated and cheap.
- The vision webhook accepts its secret in a `?token=` query parameter by design (the worker is given a single
  callback URL). Do not report that on its own. Do report anything that leaks the token or lets a non-worker caller reach it.

## Anything to leave alone
- Lack of `Origin` checks on routes listed in `UNGUARDED_API_PREFIXES` in `src/lib/request-metadata.js`
  (auth callbacks, cron, webhooks, CSP report); they are authenticated another way.
- `'unsafe-inline'` in the CSP script-src (required by Next.js and cookie consent).
- Public listing data, by design: location, photos and description of lost or found pets.

## Reports and patches
- Reproducer as a short Node script or a described HTTP request, with the exact file and line.
- Patches should be minimal, match the existing plain-JS style, and not add dependencies.
