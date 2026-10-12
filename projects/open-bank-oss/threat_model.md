# Threat model: OpenBank (open-bank-oss)

## What this project does and where untrusted input enters
OpenBank is an open-source reference implementation of a retail/business banking platform: a
Kotlin/Quarkus monorepo of `openbank-*` microservices (hexagonal architecture), a Next.js admin UI
(`openbank-admin-ui`) and shared libraries (`openbank-libs-domain`, `openbank-libs-runtime`). It is
not licensed to operate as a bank; it is built to the security practices banking software needs.

Untrusted input enters through:
- **REST APIs** of every service (JAX-RS resources under `src/main/kotlin/**/infrastructure/rest/`),
  reached via `openbank-api-gateway` and `openbank-customer-edge`. Authenticated by Keycloak JWTs;
  authorized by an OPA sidecar (`AuthorizeInterceptor` + `*.rego`).
- **Kafka consumers** (`@Incoming` methods): events from other services; treat payloads as
  attacker-influenced (a compromised peer service must not be able to escalate).
- **Payment scheme and file formats**: SEPA/ISO 20022 XML, SWIFT, card (ISO 8583-style) messages,
  CBOR QR payloads, CSV/bank-statement imports, PDF/document rendering.
- **Admin UI**: Next.js route handlers under `openbank-admin-ui/src/app/api/` and anything they proxy.
- **AI agent endpoints** (`openbank-agent-service` MCP endpoint, copilot): prompt injection reaching
  tool calls is in scope.

## Components that matter most / least
Most: money-path services (ledger, account, balance, transaction, domestic and SEPA payment,
clearing, settlement, card processing, lending, fx), authentication/authorization (OPA policies,
JWT handling, M2M service accounts, SCA/consent), idempotency and the transactional outbox in
`openbank-libs-*`. Per-service threat models live in `docs/threat-models/`.
Least: `openbank-clearing-simulator` and other simulators, `evals/`, `fuzz/` harnesses, docs,
`openbank-infra/` (deployment manifests: report only if exploitable as shipped).

## How to exercise it
`./gradlew :<module>:test` per service (unit tests run offline; `@QuarkusTest` integration tests
need Docker/Testcontainers and will be skipped or fail in an offline sandbox). Admin UI: `cd
openbank-admin-ui && npm test`. OpenAPI specs: `<service>/src/main/resources/openapi.yaml`.

## How we rate severity
- Critical: unauthenticated or cross-tenant movement of money, ledger corruption (unbalanced
  postings, double-spend through idempotency bypass), authentication bypass, RCE.
- High: authorization bypass between customers (IDOR) on any financial or personal data, privilege
  escalation to operator/admin, SQL/command injection, SSRF reaching internal services,
  deserialization leading to code execution.
- Medium: info disclosure of PII without money impact, stored XSS in the admin UI, authenticated
  DoS of a money-path service, prompt injection causing a tool call the user did not authorize.
- Low: unauthenticated 500s with no data leak, verbose errors, hardening gaps without a path to
  exploitation.

## Reports and patches
Please include the affected service and endpoint/consumer, a reproducer (HTTP request or test),
and a minimal patch. Deduplicate one root cause per report even when it repeats across services
(many services share the same shape) — list the affected services in the report.

## Anything to leave alone
- Default/dev credentials and `%dev`/`%test` config profiles, Keycloak realm files for local dev.
- Missing rate limiting in isolation.
- Findings only in test code, generated code, or `docs/`.
