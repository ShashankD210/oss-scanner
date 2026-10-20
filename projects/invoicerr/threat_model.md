# Threat model: Invoicerr

## What this project does and where untrusted input enters

Invoicerr is an invoicing application for freelancers and small companies: quotes, invoices, credit notes,
expenses, received invoices, payments, and transmission to national e-invoicing networks (France, Germany, Italy,
Poland, Portugal). It is self-hosted by its users and also run as a hosted, multi-tenant service where unrelated
companies share one instance. The scan is of `dev`.

A NestJS backend (`backend/`) serves a REST API under `/api`, backed by PostgreSQL (Prisma) and Redis (BullMQ). A
React frontend (`frontend/`) is a static single-page app that only talks to that API. Authentication is better-auth
(`/api/auth/*`): email and password, sessions in a cookie. The company a request acts on is the session's active
company (`backend/src/guards/auth.guard.ts`), or the company an API key belongs to.

Untrusted input enters through:

- The authenticated REST API, every controller under `backend/src/modules/`. This is the main surface.
- Routes marked `@Public()`, reachable with no session:
  - `public/documents` and `public/signatures`: a client views a quote or invoice and signs a quote, through a link
    and a one-time code sent by email.
  - `portal`: the client portal, where a company's client sees their own documents.
  - `invitations`: accepting an invitation to join a company.
  - `public/payments`, the payment provider webhooks, and `billing/webhooks` for the hosted service's own billing.
  - Inbound callbacks from e-invoicing networks (`public/sdi`, and
    `backend/src/modules/documents/queue/document-authority-webhook.ts`).
  - `legal`, `health`, `company-lookup`, `address-autocomplete`, and `sso` (SSO lookup).
- API keys (`backend/src/modules/api-keys/`, scopes in `scopes.ts`) and the MCP server built on them
  (`backend/src/modules/mcp/`), which act on one company without a browser session.
- Files: logos and attachments, received supplier invoices parsed from PDF and XML (Factur-X, ZUGFeRD, UBL, CII,
  FatturaPA), and backup archives imported through `backend/src/modules/backup/`.
- Document content (item descriptions, notes, client and company details, branding) rendered into PDF by a headless
  Chromium (`playwright-core`).
- Outbound requests to URLs a company configures: webhooks (`backend/src/modules/webhooks/`) and e-invoicing channel
  endpoints.

## Actors and trust boundaries

From least to most privileged:

1. **Anonymous visitor**: no session. Sign-up is open by default (an operator can close it,
   `backend/src/lib/registration-policy.ts`); a new user then creates their own company.
2. **A company's client**: not a user. Holds only what the company sent them: a document link, a one-time code, a
   portal access. Must reach only the documents of that company addressed to that client.
3. **Invited user**: holds an invitation link for one company, before accepting it.
4. **MEMBER** of a company (`CompanyRole` in `backend/prisma/schema.prisma`).
5. **ADMIN** of a company: manages members, settings, channels, API keys.
6. **OWNER** of a company: everything ADMIN can, plus billing, ownership transfer and deleting the company.
7. **API key**: acts for one company, limited to its scopes.
8. **Instance operator**: an address listed in `INSTANCE_OPERATOR_EMAILS`, guarded by
   `backend/src/guards/instance-operator.guard.ts`. Fully trusted, like anyone with shell or database access.

On a default install anyone can sign up, so an attacker usually has a company of their own where they are OWNER.
The boundaries that matter:

- **Company**: no user, API key or client of one company may read or change another company's data. This is the
  main boundary, and the one a self-registered attacker attacks first.
- **Role inside a company**: a MEMBER must not do what only ADMIN or OWNER may (`@Roles(...)` on controllers).
- **API key scope**: a key must not reach a resource or action outside its scopes, nor manage keys or members.
- **Client**: a link, code or portal access gives one client one company's documents for that client, nothing
  more. Signing a quote proves that client's identity.
- **Integrity of issued documents**: once an invoice is issued, its number, content, and the hashed archive under
  `backend/src/modules/documents/archive/` must not change, and numbering must stay continuous. A forged or altered
  invoice is a legal problem for the company, not only a data one.
- **Server-side requests**: a company configuring a webhook or channel URL must not reach the instance's internal
  network (database, Redis, cloud metadata, other services).

## Components that matter most

- **Company isolation**: every query must be scoped to the request's company. Look for lookups by id alone
  (`findUnique({ where: { id } })` without `companyId`), routes that take a `companyId` or a nested id from the
  request, and secondary paths of an operation that is checked on its main path: list versus detail, export,
  PDF rendering, email sending, search, the MCP tools, background jobs that take an id from a queue payload.
- **Authorization**: `@Roles` and `@RequiresScope` checks, and any route that should have one and does not.
- **Public routes**: one-time code generation and checking, link tokens, portal access, invitation acceptance,
  payment and network webhooks (signature verification, replay).
- **Document integrity**: changing or deleting an issued document, a gap or duplicate in numbering, altering an
  archive without the hash check noticing.
- **Parsers**: XML of received invoices (external entities, entity expansion), PDF attachments, backup import
  (paths inside the archive, records pointing at another company's ids).
- **PDF rendering**: HTML or script injection from document fields into the headless browser, which could then read
  local files or reach internal URLs.
- **SSRF** through webhooks and channel endpoints, including redirects and DNS rebinding past the outbound URL check.
- **Credentials at rest**: channel credentials are encrypted with `CREDENTIALS_ENCRYPTION_KEY`; leaking them to
  another company or to the frontend is in scope.

Out of scope: `documentation/`, `e2e/` (test and load scripts), `scripts/`, `deploy/`, and the v1.x line.

## What counts as a finding

A report needs all four:

- **Entry point**: the exact route or function, reachable on a default install.
- **Actor**: the least privileged one from the list above that works. Try an anonymous visitor first, then a
  self-registered OWNER of their own company acting on another company.
- **Input**: the request value and how it reaches the sink.
- **Impact**: what the actor reads, changes or executes that they should not.

Report one finding per root cause, with every entry point it reaches.

## Known non-findings

- Anything an OWNER or ADMIN does inside their own company, within what that role is meant to do.
- Anything only the instance operator can reach, or that needs shell, database or environment access.
- Signing up and getting a company of one's own: that is the product.
- Rate limiting, brute force of passwords, user enumeration, denial of service, missing security headers.
- Behaviour that only exists with `NODE_ENV` other than `production`, or a test-only setting from `.env.test`. Say
  so on the finding if a result depends on one.
- Self-XSS, and XSS that needs the victim to paste content into their own account.
- Vulnerabilities in dependencies with no path from Invoicerr's own code.

## How we rate severity

- **Critical**: an anonymous visitor or any self-registered user gets remote code execution, arbitrary SQL, another
  company's data in bulk, or takeover of another user's account.
- **High**: a self-registered user, a client or an API key crosses the company boundary for specific records (read
  or write), forges or alters an issued document or its archive, signs a quote as someone else, or reaches internal
  services through SSRF and reads the response. Leaking another company's channel credentials is High.
- **Medium**: crossing a role or scope boundary inside one company (MEMBER doing ADMIN work, a read-only key
  writing), stored XSS that runs in another user's session, a bypass that needs an unusual configuration.
- **Low**: blind SSRF, metadata leaks such as ids or counts of another company, missing defence in depth.

## How to exercise it

The image has the checkout at `/src`, the backend built in `/src/backend/dist`, PostgreSQL 15, Redis, and Chromium
for PDF rendering.

- `invoicerr-start` starts PostgreSQL, Redis and the backend on `http://127.0.0.1:4000`, then creates two users, each
  the OWNER of a separate company with a few clients and articles. It is safe to run again. The users, their
  passwords and session cookies are in `/opt/invoicerr/users.json`; the backend log is `/var/log/invoicerr-backend.log`.
- Call the API with the cookie: `curl -H "cookie: <cookie>" -H "origin: http://127.0.0.1:4000"
  http://127.0.0.1:4000/api/clients`. Send the `origin` header, as `/src/e2e/load/seed.js` does.
- Sign in again with `POST /api/auth/sign-in/email` and `{"email": ..., "password": ...}`. better-auth allows three
  sign-ins or sign-ups per 10 seconds.
- More users: `POST /api/auth/sign-up/email`, as `/src/e2e/load/seed.js` does. To test roles, invite a second user
  into a company through `/api/invitations` as its OWNER.
- Database: `psql postgresql://invoicerr:invoicerr@127.0.0.1:5432/invoicerr_db`, to set up data or observe a result.
- Unit tests: run `invoicerr-start` first (they need the database), then `cd /src/backend && npm test` (Vitest).

To show a cross-company issue, act as the user of the first company on a record id of the second. Run a positive
control as the same user in the same run: a 403 or 404 can come from a check unrelated to the one under test. Prove
the impact without maximising it, and do not destroy data.

## Reports and patches

Include the HTTP requests that reproduce the issue against the local backend, the user they ran as, and a minimal
patch against `dev`. Patches follow the repository's own style (`backend/` is TypeScript, checked by ESLint and
Prettier) and fix every entry point that shares the root cause.