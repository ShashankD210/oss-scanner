# Threat model

## What this project does and where untrusted input enters
FloorVote is self-hosted legislative tracking software for advocacy teams, deployed on Cloudflare Workers with D1 databases. Each team (tenant) is its own Worker and database; a shared central Worker ingests bills from legislature data providers and serves them to tenants over Cloudflare service bindings.

Untrusted input enters through:
- HTTP requests to a tenant's API (`api/`), from anonymous visitors (sign-in, magic-link redemption, public pages) and from signed-in members with different roles (standard, admin, owner, superadmin).
- Bill text, titles, people and vote data fetched from third-party legislature sources by the providers in `central/src/providers/` (LegiScan, DC LIMS, Virginia LIS, Maryland MGA). Bill HTML is sanitized before display and is never trusted.
- Single-use links sent by email (sign-in, email-change confirmation, and "This wasn't me" links).

## Components that matter most / least
- Most: authentication and session handling (magic links, email change, account deletion), role checks on admin and owner routes, tenant isolation, the deny-by-default tenant-facing entrypoint of the central service, HTML sanitization of bill text, and the Content Security Policy in `shared/securityHeaders.ts`.
- In scope: provider parsers in `central/src/providers/`, the web app in `web/`.
- Less important: `scripts/` (developer and operator tooling run by trusted people), `docs/`.
- Out of scope: `graphify-out/` (generated), and an operator's own Cloudflare configuration and secrets (`wrangler.toml` files are gitignored; see `docs/content/security/index.md`).

## How to exercise it
- `api/test/` and `central/test/` run on workerd through Vitest (`npm test` in each directory); `root/` and `web/` have their own `vitest run` suites.
- `docs/content/security/index.md` describes the protections the project intends to provide.

## How you rate severity
- Critical: reading or changing another tenant's data, authentication bypass, or reaching operator-only central routes from a tenant.
- High: privilege escalation between member roles within a tenant, account takeover of a single member, stored XSS through bill text or other provider data.
- Medium: issues that need an already-privileged admin or owner, rate-limit bypasses, CSP weaknesses without a working injection.
- Low: denial of service against a single self-hosted instance.

## Anything to leave alone
- Turnstile is off until an operator configures it, and the sign-in gate fails open when its secret is unset. This is by design for local development.
