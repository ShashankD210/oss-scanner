# ReqogniLoom — threat model

Input for the OSS Scanner. Describes what the project is, where untrusted input
enters, and how the maintainer would like findings rated, so the scanner's
reports are calibrated to this codebase rather than to generic web-app defaults.

## What the project is

ReqogniLoom is an AI-native requirements- and test-management tool. It is a
self-hosted web application:

- **backend/** — Django 6.1 + Django REST Framework REST API (20 apps, service
  layer under `application/`, ~47 services). PostgreSQL 16 with pgvector
  (vector(384) embeddings, HNSW indexes), Redis + Celery for async tasks
  (Celery is *not* needed for the test suite — see below).
- **mcp_server/** — a native MCP server (JSON-RPC 2.0) exposing 227 tools under
  36 group prefixes, for AI-agent integration, mounted at `/mcp/` and
  `/api/v1/mcp/`.
- **frontend/** — React 19 + TypeScript SPA (Vite 8, Vitest 5).
- **llm_adapter/** — pluggable LLM providers (Anthropic, OpenAI, Ollama, Azure,
  and a deterministic `mock` used by all tests).
- Multi-tenancy: every row is tenant-scoped via PostgreSQL Row-Level Security
  (`app.*` GUCs, least-privilege app role `reqogniloom_app`); JWT auth with
  login throttling; field-level Fernet encryption for provider API keys at rest.

Trust boundaries of interest: unauthenticated internet → API → service layer →
ORM/PostgreSQL (tenant isolation is the security property), and any
user-supplied text that later reaches an LLM (prompt-injection surface), the
Markdown/report renderers, or the import/export parsers.

## Where untrusted input enters

1. **REST API** `/api/v1/*` — all artifact CRUD (requirements, architecture,
   ADRs, risks, issues, test cases, diagrams, ICDs), auth endpoints, search,
   bulk operations. JWT-authenticated, per-tenant.
2. **MCP endpoints** `/mcp/` and `/api/v1/mcp/` — JSON-RPC 2.0, API-key
   authenticated, throttled; same service layer and tenant isolation as REST.
3. **Importers** — CSV bulk import and ReqIF 1.2 (XML) import/export for
   DOORS/Polarion interoperability: file parsing of attacker-controlled files.
4. **LLM round-trips** — decomposition/validation output is persisted and
   rendered in the SPA; persisted prompts are a *deliberate* prompt-injection
   surface documented in the code (`rest_api/auth_enforcer.py`, REQ-043). An
   injection that changes LLM behaviour is in scope; LLM *output* being wrong is
   a correctness issue, not a vulnerability.
5. **Outbound integrations** — LLM base URLs (SSRF surface, guarded by
   `LLM_ALLOW_PRIVATE_BASE_URL`), optional Qdrant/Honcho memory backends.
6. **Admin** — Django admin under per-tenant superuser.

## Severity guidance (how we would like reports rated)

- **Critical** — cross-tenant data access: any RLS bypass, missing tenant
  predicate, or tenant-id confusion that leaks another tenant's artifacts. This
  is the crown jewel of this codebase; treat proven cross-tenant reads/writes
  as critical.
- **Critical** — remote code execution in the backend (the server runs as a
  least-privilege user, but RCE is still game over: full DB and every tenant's
  data).
- **High** — authentication/authorization bypass (JWT confusion, throttle
  bypass, privilege escalation between roles), SQL injection (there are
  hand-written raw SQL paths in metrics/analytics and search — post-auth SQLi
  is high, not medium), SSRF reaching cloud metadata or internal services via
  the LLM base-URL or any outbound-request path, server-side request forgery
  through the import/export pipelines.
- **High** — stored prompt injection that crosses a trust boundary (e.g. a
  requirement description that makes the LLM exfiltrate another tenant's data
  or disable a safety control).
- **Medium–High** — stored XSS in the SPA: severity scales with what an
  attacker-controlled artifact can reach (token theft is mitigated — the JWT
  lives in memory, not a cookie — so no direct session theft, but admin-action
  forgery via rendered content is high).
- **Medium** — CSRF (API + SPA share an origin), reflected content injection,
  information disclosure of non-artifact internals, dependency CVEs with a
  plausible path (report the patched version, please).
- **Low / Info** — hardening suggestions without a demonstrated path.

**Out of scope** — findings that only apply to deliberately unsafe optional
sidecars: the Honcho memory backend (QA/demo tool, no auth by design) and the
bluepencil review sidecar (no auth, no tenant isolation, QS-only, off unless
`BLUEPENCIL_ENABLED=1`); behavior that requires a non-`mock` LLM provider or a
Qdrant backend (both are opt-in, never the default); Python memory-safety
classics (buffer overflow, use-after-free) in pure-Python code — for C
extensions (psycopg2, torch/numpy via embedding arrays, reportlab) treat memory
corruption as high at most unless a reproducer demonstrates exploitability;
DoS that requires an authenticated tenant to import its own gigabytes without
amplification.

## How to build and test (already wired into the Dockerfile)

The image installs everything and bakes in dependencies, so the offline phase
needs no network:

- `run-backend-tests [pytest args]` — starts the local PostgreSQL 16 cluster,
  provisions the `reqogniloom` superuser role/database (the RLS tests need
  superuser to `ALTER ROLE reqogniloom_app`), then runs `pytest` from
  `/src/backend`. Settings: `reqogniloom.settings_test` (imported via
  `backend/pyproject.toml`) — deterministic `mock` LLM provider, locmem cache,
  eager Celery, generated-per-run Fernet key. Test-only secrets never touch
  production values.
- `run-frontend-tests [vitest args]` — `npm test` (vitest run) from
  `/src/frontend`.
- `run-all-tests` — both, backend first.

Notes for the auditor:

- The full checkout sits at `/src`; backend at `/src/backend`, frontend at
  `/src/frontend`. The whole suite (thousands of tests) runs in a few minutes.
  For a faster loop, run targeted subsets, e.g.
  `run-backend-tests rest_api/tests/test_security_hardening_269.py` or
  `run-frontend-tests src/test/theme-contrast.test.ts`.
- Backend tests marked `integration` expect a live compose stack and are not
  part of the default suite.
- The project's own CI (pytest + pip-audit + Trivy) already gates dependency
  CVEs; still worth reporting anything with a live path, but we track those
  separately and will cross-reference.

## What we would like in a report

- A minimal reproducer against the default configuration (mock LLM provider,
  pgvector backend, no optional sidecars enabled).
- The affected REQ/ADRs if the finding maps to a documented decision (the repo
  uses `REQ-*` IDs and MADR ADRs; `docs/` carries the matrix) — optional.
- For anything auth- or tenant-related: confirmation that the finding
  reproduces with `reqogniloom.settings_test` defaults, so we can tell it apart
  from deployment misconfiguration.
