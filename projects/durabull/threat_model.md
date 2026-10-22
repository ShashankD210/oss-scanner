# Durabull

Durabull is a dashboard, HTTP API, desktop application, and remote MCP server for
operating BullMQ queues on Redis. It can inspect job payloads, results, logs and
stack traces, retry/promote jobs, pause queues, manage schedulers, and perform
destructive queue operations through the web/API surface. A compromise can
expose production job data or disrupt downstream workers. The code on the
repository's main branch is open source under Apache License 2.0; see `LICENSE`.

## Operational impact

Durabull is an administrative surface for the queues that drive downstream
business workflows. Assess unauthorized access to job data and credentials, and
unauthorized queue/job mutations, against the documented access model. Depending
on the connected workers, a successful attack could halt processing, remove
pending work, replay non-idempotent business actions, or expose sensitive data
across queues and connected systems. Establish the attacker's actual privileges
and reachable impact; intended actions by an authorized operator are not an
authorization bypass.

## Scope and trust boundaries

Scan Durabull's own code in this repository. Prioritize:

- `apps/api/src`: request validation, authentication/organization membership,
  connection ownership, privileged queue/job/scheduler operations, Redis key
  handling, alert delivery, webhook/Linear clients, rate limits and telemetry.
- `packages/auth`: login/session handling, OAuth consent, redirects, token
  issuance and resource/audience binding.
- `apps/api/src/mcp` and `packages/mcp`: bearer validation, scopes, tenant and
  service-account policy checks, tool/resource dispatch, audit logging,
  sanitization, and the embedded MCP App's host bridge and rendering.
- `packages/dal`: tenant scoping, database queries, persistence modes, and
  encryption/decryption of stored Redis URLs and integration secrets.
- `apps/web/src`: rendering of untrusted job data, logs and errors, credential
  handling, and requests that mutate queues or jobs.
- `apps/desktop/src`: IPC, navigation/window boundaries, and local server setup.

Treat HTTP/MCP requests, OAuth parameters, resource URIs, identifiers, job names,
payloads, results, logs, stack traces and Redis-provided strings as untrusted.
Consider unauthenticated remote attackers, users of another organization,
restricted OAuth/service-account principals, and malicious job content rendered
to a more privileged operator. Users do not gain access to another organization's
connections just by knowing their identifiers. Check SSRF and credential leakage
in outbound connections/integrations against the documented intended behavior.

MCP reads and writes have separate scopes. Service accounts also require policy
bindings. Destructive queue operations, job data edits and arbitrary Redis key
access are intentionally absent from the MCP catalog; they must not become
reachable through other MCP operations. See `docs/adr/0001-mcp-security-architecture.md`
and `docs/mcp-operations-runbook.md` for the intended model.

Authless self-hosted mode intentionally grants owner access to every visitor and
must run on a trusted machine or behind external access controls. That documented
grant alone is not an auth bypass; unintended activation, crossover into
authenticated mode, or violations of other boundaries remain in scope.
Environment-defined Redis connections and PGlite are supported deployment modes.
Do not use authless-only testing as evidence that authenticated isolation holds.

Third-party dependencies, BullMQ workers owned by users, Redis/PostgreSQL server
implementations, cloud infrastructure and external SaaS providers are not scan
targets. A weakness in Durabull's own integration with them remains in scope.
Do not probe production services, real customer Redis instances, or third-party
systems. Use loopback fixtures and synthetic data. Never use production secrets.

## Offline environment and tests

The full checkout is `/src`; Bun 1.3.5, Node.js 22, development dependencies,
Chromium/Playwright, Redis and PostgreSQL are installed. The API bundle is
`apps/api/dist/index.js`, the dashboard is `apps/web/dist`, and the desktop main
bundle is `apps/desktop/dist/main.js`. No service starts automatically.

From `/src`, CI's principal security regression commands are:

```sh
bun run mcp:check
bun test packages/auth
bun test packages/dal/src/db/env-redis-connections.test.ts apps/api/src/lib/authless.test.ts
bun run --cwd apps/web test:unit
```

Other Bun tests live beside source files. Some use `mock.module`; when running
unrelated suites together, isolate test files in separate Bun processes if mocks
interfere. Integration tests and browser tests need disposable local services.
PostgreSQL can be started with `service postgresql start`; Redis with
`redis-server --daemonize yes --bind 127.0.0.1`. These services use loopback and
work with external networking disabled. Use the local setup/seed instructions in
`apps/docs/content/documentation/getting-started/local-development.mdx`, the
environment variable guide beside it, and `.github/workflows/ci.yml` for the
PostgreSQL database, test-only credentials, migrations and E2E configuration.
Playwright's browser binaries are already under `/opt/playwright`.

Authenticated E2E tests run from `apps/web` with `bun run test:e2e`; the separate
authless CI job uses `bun x playwright test e2e/authless.spec.ts e2e/alerts.spec.ts`
with its documented authless environment. Both need the fixture setup described
above. External email, OAuth providers, Linear and telemetry destinations require
local mocks; do not configure live provider credentials for a scan.

## Findings and severity

Include the affected commit and source locations, attacker prerequisites,
deployment/authentication mode, expected boundary, observed impact, and a minimal
reproducer using synthetic local data. Explain any mocked behavior and why it
reflects a reachable production path. Prefer a focused patch and regression test;
deduplicate symptoms that share the same root cause.

Assess severity from demonstrated impact and required privileges. Unauthenticated
code execution, broad credential compromise or broad tenant compromise can be
critical. Cross-tenant sensitive-data access, unauthorized privileged queue
operations and exploitable session/token flaws can be high. XSS and denial of
service depend on the affected principal, persistence, reach and recovery cost.
Do not claim code execution solely because arbitrary job data can be submitted:
show how a real trust boundary is crossed. Document reduced-impact findings too;
these examples are guidance, not automatic severity caps.

Send confidential reports to `hello@durabull.io`, the security contact published
at https://durabull.io/contact. Keep vulnerability details out of public issues
and enrollment PR discussions.
