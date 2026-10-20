# MoleSignal threat model

## Project and scope

MoleSignal is a self-hosted observability platform implemented as a Rust workspace
with a React/TypeScript web client. It ingests logs, metrics, traces, and other
telemetry, stores metadata in PostgreSQL and telemetry in object storage, and
provides organization-scoped queries and administrative APIs.

Audit the public default build: `bin/molesignal`, `bin/probe-agent`, `crates/`,
`web/`, and reference deployment configuration under `deploy/`. The project's
`SECURITY.md` is the authoritative disclosure and scope policy. Private cloud
services and external production infrastructure are outside this checkout.

## Assets and trust boundaries

- An unauthenticated client must not access authenticated APIs or other users'
  data. Check JWT handling, API keys, session lifecycle, and authorization.
- A member or ingestion client of one organization must not read, write, query,
  delete, or infer another organization's telemetry, metadata, or credentials.
  Trace organization scoping through HTTP/gRPC handlers, query planning, caches,
  object/file selection, PostgreSQL queries, and background work.
- Telemetry and query text are untrusted. Review protocol decoders, decompression,
  SQL/PromQL parsing and planning, expression execution, and resource limits.
- Stored telemetry, dashboard content, links, and replay content must not execute
  attacker-controlled script in the browser or expose privileged credentials.
- Connector, webhook, probe, and tool destinations can cross network boundaries.
  Review SSRF, redirects, DNS/IP validation, credential forwarding, and whether
  lower-privileged users can reach operator-only resources.
- User-defined functions and agent/tool inputs must respect their intended
  execution and authorization boundaries. Review sandbox escapes, path traversal,
  command execution, and access to secrets or other organizations' resources.
- Signing secrets, encrypted credentials, and cipher keys must not leak through
  responses, logs, errors, exports, or unauthorized configuration access.

## Deployment assumptions and exclusions

Follow `SECURITY.md`: operators set a real `MS_CIPHER_KEY`, rotate bootstrap JWT
signing secrets, and restrict internal `/api/v1/_*` and `/metrics` surfaces at
trusted ingress boundaries. Treat host administrators and deployment configuration
as trusted; do not assume ordinary organization members are host administrators.
Do not dismiss an application authorization bypass merely because ingress can
restrict an endpoint.

Use the existing security-policy exclusions: unreachable dependency issues,
DoS requiring unrealistic operator settings, locally patched non-default builds,
and already documented missing hardening features. Resource exhaustion reachable
under ordinary settings remains relevant.

## Reproduction and severity

Prioritize demonstrated authentication bypass, cross-organization access,
credential disclosure, and code execution across a trust boundary. Describe the
attacker's privileges, required configuration, affected assets, and realistic
impact; severity should follow the demonstrated exploit and its prerequisites.
For isolation reports, reproduce with two separate organizations and verify both
read and mutation paths where applicable. Include the commit, minimal payload,
expected versus actual behavior, and a focused regression test or proposed fix.
Send findings privately to the configured security contact.

## Build and tests in the scanner image

The checkout is at `/src`; default development binaries are under `target/debug`.
The Dockerfile retains Rust, Node, pnpm, installed web dependencies, Cargo caches,
compiled unit tests, and embedded web assets. Rust frame pointers are enabled by
`.cargo/config.toml`; the scanner build retains line-table debug information.

Run offline unit/bin tests with:

```sh
cd /src
cargo test --offline --locked --workspace --lib --bins
pnpm -C web test:run
```

Build-time test output is saved in `/tmp/molesignal-rust-tests.log` and
`/tmp/molesignal-web-tests.log`. Inspect failures rather than treating successful
image creation as proof that all tests passed.

Integration tests in `bin/molesignal/tests` have additional service requirements;
see `.github/workflows/it.yml` and each test's setup. The normal integration suite
uses `MS_RUN_IT=1` and Docker-backed services. The scanner image does not provision
a nested Docker daemon, PostgreSQL, or object storage, and does not prebuild the
full integration suite. A skipped service-dependent test is not isolation proof.
Browser end-to-end tests likewise need a running application and browser runtime;
those are not installed by this enrollment image. Prefer focused unit reproductions
or explicitly provision local fixtures when exercising these paths offline.
