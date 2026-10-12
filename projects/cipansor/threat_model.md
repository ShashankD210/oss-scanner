<!-- Threat model for the Cipansor IMS enrollment. The scanner reads this before it starts. -->

# Threat model

## What this project does and where untrusted input enters

Cipansor (Information Management System) runs a pesantren and its four schools
(TK Qur'an, SD IT, SMP IT, SMA Qur'an + tahfidz). Two runtimes share one
repository:

- `apps/api` — Express 5 REST API. It is the trust boundary. Untrusted input
  enters through every HTTP request: JSON bodies, query strings, path
  parameters, multipart file uploads (attendance selfies, identity documents,
  letter PDFs), signed e-office letters, and inbound webhooks/callbacks from
  third parties (mail, push, WhatsApp/SMS). Authentication is JWT + optional
  TOTP 2FA; authorization is `RoleCode` + fine-grained permissions.
- `apps/web` — Next.js 16 App Router client. It is the untrusted caller from
  the server's point of view, and its own session/CSRF handling matters too.

Treat every HTTP endpoint, every uploaded file, and every webhook payload as
attacker-controlled. The database is PostgreSQL; Redis is used as a cache and
for scheduled-job coordination.

## Components that matter most / least

- **Most important:** authentication and 2FA (`modules/auth`), the RBAC/
  permission middleware (`middleware/auth.ts`), the public letter-verification
  and signed-PDF upload path (`/public/verify-card`, `/verifikasi`), the
  identity-document and attendance-photo handlers (they serve files that are
  deliberately outside `public/` and must stay behind a role check), the
  chatbot, and anything that builds an outbound URL or sends a message to a
  third party (open-redirect / SSRF / injection surface).
- **In scope but lower impact:** scheduled jobs (`src/jobs/`), dashboards and
  aggregate queries, report-card generation.
- **Out of scope:** `apps/web/public/icons/**` and other static assets;
  `node_modules`; generated `dist/`; test fixtures under any `e2e/` or
  `tests/` directory; seed/demo data (`prisma/seeds/`, `seed-presentasi.ts`).

## How to exercise it

- Unit tests: `pnpm --filter api test` (vitest, Prisma mocked) and
  `pnpm --filter web test`.
- The API builds with `pnpm --filter api build` (tsc). The web builds with
  `pnpm --filter web build`.
- A full end-to-end run needs PostgreSQL and Redis. Point `DATABASE_URL` at any
  reachable Postgres (the unit tests mock Prisma and never touch the DB), run
  `pnpm --filter api db:push`, then start the API with `pnpm --filter api dev`
  and issue requests against `http://localhost:3001`. `GET /health` answers
  without a database.
- The Dockerfile in this directory states exactly how the scanner should build
  the checkout; it stops at a green build rather than starting the servers.

## How you rate severity

- Authentication bypass, privilege escalation across units/roles, or any
  unauthenticated read of another person's data (identity documents, attendance
  photos, financial records, grades): **critical**.
- SQL injection, stored/reflected XSS, SSRF, path traversal, or arbitrary file
  read/write reachable from an HTTP endpoint: **high** at minimum; **critical**
  if it needs no authentication or yields code execution.
- An authorization gap that needs an already-valid session (a role reaching data
  its role should not, e.g. a teacher reading another unit's finance): **high**.
- Denial of service, unbounded resource use, or missing rate limiting without a
  demonstrated amplification: **medium**.
- Information disclosure that only reveals data the caller may already see
  (verbose errors, version banners): **low**.

## Anything to leave alone

- Do not report missing security headers or TLS configuration in
  `next.config.ts` / `nginx.conf` unless the configuration actively defeats a
  control; those are set deliberately in the deployment layer.
- The legacy `UserRole` enum and the `src/services/*` / `lib/api-client.ts`
  re-exports are being removed; a finding that exists only because of them is
  low priority. Report the equivalent issue in the `RoleCode` path instead.
- Do not treat the demo/presentation seed data as real people; it is synthetic.
