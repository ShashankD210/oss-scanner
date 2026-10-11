# Threat model

## Purpose, scope, and assets

This repository backs up coding-agent sessions and managed skills from machines to a
Cloudflare Workers hub. Python collectors incrementally upload local files; a Python
client reads approved session data. The hub retains original bytes in R2, parses and
indexes them in D1/FTS5, drains parse/rollup queues, and renders an HTML viewer, search,
usage/cost diagnostics, media, and session export ZIPs. It also receives anonymous OMP
QA reports. The separate telemetry gateway and OIDC issuer federate to Azure.

Audit all tracked source under `/src`, including `hub/src`, `hub/gateway`, hub scripts,
`collector`, `client`, `infra`, SQL migrations, provisioning/preview scripts, and tests.
Do not restrict analysis to a compiled Worker bundle or a hand-picked parser. Installed
third-party dependencies remain under their own licenses and notices; do not describe
the root MIT license as relicensing npm/Python dependencies or browser/runtime binaries.

Session content is sensitive even when uploaded by an authenticated machine: prompts,
responses, tool arguments/results, absolute paths, source snippets, images, exports,
usage metadata, and skill Markdown may contain credentials or private work. Raw R2
objects, displaced-object quarantine, D1 blocks/search indexes/reports, collector state,
read-grant caches, OAuth access/refresh tokens, session cookies, and signing secrets are
all protected assets. Public resource identifiers and public JWKs are not secrets.

## Untrusted inputs and trust boundaries

- Remote HTTP requests: paths, query parameters/cursors, cookies, headers, OAuth
  callbacks, WebAuthn responses, read-grant parameters/code exchanges, certificate
  enrollment/renewal requests, JSON upload metadata, multipart parts, and QA reports.
- File content: malformed/truncated/oversized JSONL, arbitrary UTF-8 and byte ranges,
  nested JSON, agent ancestry/tool-pairing metadata, Markdown, embedded media/base64,
  provider export ZIPs and `conversations.json`, filenames/relative paths, timestamps,
  and pricing payloads. An authenticated uploader can still supply malicious content.
- Collector inputs: discovered store paths, symlinks, file replacement/rotation during
  upload, config/state SQLite, staged exports, CDP/WebSocket replies and captured web
  conversations. The collector must not accidentally send excluded credential files
  or escape selected stores through path handling.
- Cloudflare's verified `request.cf.tlsClientAuth` is infrastructure-provided identity,
  not an ordinary user header. Production also resolves the certificate fingerprint
  against D1, including revocation/grace/current-slot rules. A fake client-supplied
  header must not substitute for this boundary.
- The local development `x-dev-machine` path is deliberately privileged and exists
  only under `ENVIRONMENT=development`. Its safety depends on the repository's
  loopback-only local runner. Missing/unknown environments and production must not
  inherit it. Preview bearer authentication is a distinct, disposable environment,
  not authority over production resources.

## Authentication and sensitive operations

The intended production policy is **machine certificates ingest; passkeys egress**.
Inspect `hub/src/auth/identity.ts`, the API router, upload/multipart handling, and
certificate routes for cross-machine access, grace-period privilege escalation,
revoked-cert acceptance, content reads accidentally allowed to a machine certificate,
and destructive replacement. Non-append uploads preserve predecessor bytes under
write-once `raw-prev/` keys; hash checks, retries, concurrency, and cleanup must not
silently destroy that protection or expose retained predecessors.

Passkey registration/authentication, session issuance, host/RP-ID binding, challenge
expiry/one-time use, user verification, credential counters, CSRF/origin checks, secure
cookie attributes, and bootstrap/setup authorization matter. Browser-controlled
parameters must not change what the owner approves. `hub/src/auth/read-grants.ts`
implements a fresh-passkey-approved, PKCE-protected loopback code flow with a bounded
TTL; codes/tokens are hashed and the bearer is read-only. Check exact callback address,
parameter binding, code replay/races, expiry/revocation, route allowlists, token leakage
through redirects/logs/referrers, and the Python client's grant-cache file permissions.
Possession of an ordinary machine certificate must not mint a read grant.

`hub/src/auth/cloudflare-oauth.ts` is a managed-CA OAuth broker backed by a Durable
Object. It stores pending state/PKCE verifiers and access/refresh grants in DO storage;
do not assume application-level encryption. Audit passkey authorization of the public
start/callback/disconnect/status routes, state expiry/consumption, PKCE, redirect and
scope binding, refresh-token rotation and serialization, fail-closed reauthorization,
and certificate-operation parameter scoping. Tokens must not appear in responses,
logs, provisioning artifacts, or readable session storage. The legitimate
`ssl-and-certificates.write` and `offline_access` scopes are security-sensitive.

The OIDC issuer publishes only public verification keys. Telemetry gateway signing
keys and ingest bearers are deployment secrets. Inspect assertion issuer/audience/
subject/time checks, authorization, retry/ack behavior, OTLP conversion/bounds and
unexpected exfiltration to external endpoints. Test this with synthetic inputs and
stubbed outbound fetches only, never with an Azure or Cloudflare credential.

## Parsing, storage, rendering, and exposure priorities

- Upload identity/path validation, multipart integrity/size limits, content hashes,
  deduplication/canonical choice, byte-range consistency, retry/race recovery, R2/D1
  divergence, stale queue messages, reindex jobs, and retention/quarantine semantics.
- JSONL readers and all harness parsers: integer/offset overflow, UTF-8 boundaries,
  resource exhaustion, malformed-line recovery, unbounded nesting/large base64, and
  preservation of the correct raw-to-indexed provenance after reparses.
- ZIP ingestion/export: decompression amplification, unexpected entries/path
  traversal, safe narrowed relative paths, streaming/backpressure, and authorized
  export of exactly the intended session rather than unrelated machine/store bytes.
- D1/FTS queries, search syntax/cursors, SQL migrations, managed-skill metadata,
  diagnostic/report ingestion, and rollup queues: injection, accidental deletion,
  unauthorized disclosure and persistent storage/CPU amplification.
- Viewer HTML, Markdown, tool content, reports, filenames, links and media: stored
  XSS/HTML attribute or script-context injection, dangerous URL schemes, MIME
  sniffing, content disposition/CSP failures, and authorization of every raw/blob/
  export/search route. A rendered transcript is not trusted executable content.
- Logging, telemetry, exception bodies, caching, redirects, preview diagnostics,
  browser traces and exports must not turn private session data or credentials into
  an unauthenticated endpoint or a public build artifact. Anonymous QA intake must
  retain its rate/size/storage bounds and must not confer read privileges.

## Build environment and offline exercise

The Dockerfile keeps the complete checkout at `/src`. It installs CI's Node 22.22.2,
locked npm dependencies (including workerd/Miniflare), pinned uv, a managed Python 3.12,
collector/client development dependencies, the collector `webcapture` extra, hatchling
build requirements, and Playwright's Chromium plus Linux shared libraries. It dry-run
bundles the real hub, preview, issuer and gateway Workers, builds both Python packages,
and pre-resolves/synchronizes both PEP 723 enrollment-script environments. Package
caches, virtual environments, bytecode, wheels/sdists and browser binaries stay in the
image. Builds use network access; tests are deliberately left for offline execution.

No real credentials, client certificates, TPM keys, local `.env`/`.dev.vars`, user home
mounts, live portals, or real transcript corpus are necessary or permitted. The image
must be built from the scanner's fresh remote tracked checkout, not a live workstation.
Do not enable cloud credentials to make a test pass. Clear `BASE_URL` so Playwright uses
its real local harness rather than a deployed preview, and leave private seed/corpus
options unset.

From the scanner's network-disabled shell, run the actual CI test conventions:

```sh
npm --prefix /src/hub run typecheck
npm --prefix /src/hub test
uv run --directory /src/collector --offline --no-sync pytest
uv run --directory /src/client --offline --no-sync pytest
uv run --offline --frozen --script /src/infra/cf/test_enroll_cert.py
```

The hub suite uses the Workers Vitest pool and real local D1/R2/KV/queue/DO behavior,
plus its existing Node test drivers; `npm test` includes migration/lifecycle/preview
script coverage rather than filtering those drivers away. Collector/client tests use
temporary files and existing loopback/fake HTTP servers, not the production hub. The
certificate regression script is explicitly hermetic: its software-generated test
material is synthetic and it mocks Cloudflare/process operations; it does not access
TPM hardware or import/enroll a user's real client certificate.

### Offline synthetic runtime smoke

Run the existing browser suite against a freshly started real local Worker:

```sh
env -u BASE_URL -u PREVIEW_BEARER -u PREVIEW_BEARER_SEED \
  npm --prefix /src/hub run test:e2e
```

`hub/e2e/fixtures/environment.ts` owns the server per worker: it creates an isolated
home, starts `scripts/environment.mjs e2e` on `127.0.0.1`, waits for readiness, and
tears it down. The environment runner reproducibly builds the Worker using already
installed dependencies, applies local migrations, seeds only repository-generated
synthetic sessions/skills and fixture model prices, and exercises real uploads,
queue indexing, search/viewer/media/export behavior in Chromium. No real authenticator
is required for this development-mode smoke. The suite's existing diagnostics are
written under `/src/hub/test-results` and its HTML report under
`/src/hub/playwright-report`. Loopback networking still works with Docker
`--network none`; no host ports or host filesystem mounts are required.

For interactive synthetic inspection instead, run:

```sh
npm --prefix /src/hub run dev:up -- --name oss-scanner --port 8787 --readiness-ms 180000
```

This is a foreground loopback-only server, not a deployment. A second shell in the
same container can request `http://127.0.0.1:8787/healthz` and inspect the synthetic
viewer. Stop the process before resetting its owned state; do not use the private
`session:seed-corpus --acknowledge-private-data` path.

## Limits and exclusions

Linux local emulation is not Cloudflare's production edge/WAF/mTLS enforcement,
remote Queues/R2/D1 durability, Azure federation, or physical passkey/TPM behavior.
The production trust boundary therefore also needs source review. WebAuthn tests
include injectable verification seams; they prove endpoint/state wiring, not a real
hardware authenticator's cryptography. CDP fixtures do not establish a real ChatGPT/
Claude login. No native project binary or Windows cross-compile target exists in the
current source; Windows PowerShell/Schannel/TPM/scheduler paths remain in the source
audit but cannot be validated by this Linux container. Node/uv/workerd/Chromium and
Python cryptography wheels are installed platform dependencies, not project binaries
that can establish Windows collector behavior through cross-compilation.

Do not run deployment/provisioning workflows, `wrangler deploy` without `--dry-run`,
OAuth authorization/consent, certificate enrollment/revocation/reset scripts, real
browser capture, scheduled production jobs, price downloads, private-corpus seeders,
or commands targeting the configured production hostnames. Existing provisioning
and preview-control tests mock their external APIs; review those scripts as source
without attempting their live operational side effects. No adoption or production
security certification is asserted by this enrollment.

## Severity guidance

Unauthenticated access to private transcripts/media/exports, OAuth/read/session token
exfiltration, production authentication bypass, cross-machine data access, stored XSS
against the owner, destructive loss of protected backups, or managed-CA privilege
escalation is high or critical depending on reach and impact. Stolen ingest-only
credentials enabling content reads is a policy breach, not expected access. Remote
code execution or broadly exploitable infrastructure/token compromise is critical.
Availability/storage amplification is usually medium, escalating when cheaply
repeatable against the production service or capable of causing unrecoverable backup
loss. Pure display/accounting mistakes without a security impact are lower priority.
Intended loopback-development privileges and explicitly public synthetic previews
are not themselves production auth bypasses; crossing that environment boundary is.
