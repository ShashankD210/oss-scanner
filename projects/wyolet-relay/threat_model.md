# Threat model

## What this project does and where untrusted input enters
Wyolet Relay is a self-hosted LLM API gateway: clients send OpenAI- or Anthropic-shaped requests, relay authenticates them with its own API keys, applies policies and rate limits, picks an upstream provider key, forwards the request, and records usage. It stores upstream provider credentials, sealed at rest.

Untrusted input:
- Inference API (`app/httpapi/inference`, `app/transport/ws`): request bodies, headers (including the `X-WR-*` proxy-mode headers) and API keys or relay-minted tokens from any internet client. Bodies can be large (megabytes) and streamed responses are long-lived.
- Upstream provider responses (`sdk/adapters/*`, `app/pipeline`, `app/proxy`): JSON and SSE from third-party APIs, parsed and translated between wire shapes. Treat as untrusted.
- Control API (`app/httpapi/control`): authenticated admin and user CRUD, manifests applied via `app/apply`, OIDC login callbacks. Authenticated, but authorization boundaries between users, teams and projects matter.
- OTLP receiver (`app/otlpreceiver`, `pkg/otlp`): trace payloads from clients.
- Catalog releases fetched at boot (`app/seed`, `app/catalogserver`), verified against a published sha256.

## Components that matter most / least
Most: authentication and authorization on both APIs (`app/httpapi/inference` middleware, `app/authz`, `auth/`), API key and token verification, the secret module (`secret/`: sealed format, data-key ring, KEK wrapping) and `app/hostkey`, tenant isolation (a key or user reaching another tenant's policies, keys, usage or payload logs), rate-limit and budget enforcement bypasses, SSRF through upstream host or URL configuration, request smuggling or header injection toward upstreams, and leaking upstream credentials into responses, logs or usage records.

Less: the SDK client (`sdk/client`), CLI tools under `cmd/` other than `cmd/relay`, and test or bench helpers. `deploy/`, `chart/` and `docs/` are out of scope.

## How to exercise it
`go test -short ./...` (and per module: `./sdk/...`, `./auth/...`, `./secret/...`, `./jobq/...`) runs without external services. Handler-level tests in `app/httpapi/inference` and `app/httpapi/control` build the real router with in-memory stores (`kv.Mem`) and an httptest upstream; they are the quickest harness for request-path findings. The binary is `cmd/relay`; a full run needs Postgres, so prefer handler tests.

## How you rate severity
- Critical: unauthenticated access to the inference or control APIs, disclosure of stored upstream credentials or master/data keys, cross-tenant access to keys, credentials, usage or payload logs, remote code execution.
- High: authenticated privilege escalation (a user acting as admin or on another team's resources), bypass of policy model restrictions, SSRF to internal networks, forging or replaying relay-minted tokens.
- Medium: rate-limit or budget bypass, unauthenticated resource exhaustion that costs the operator (memory, storage, upstream spend).
- Low: DoS that needs valid credentials, leaks of non-secret metadata.

## Anything to leave alone
- Gaps in OpenAI or Anthropic wire-format coverage are not vulnerabilities.
- Admin-token holders are fully trusted; findings that need the admin token are out of scope unless they reach secrets an admin should not see (the raw master key, unsealed provider keys through the API).
