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

### Untrusted PR previews versus production

The CI preview job executes the PR's own code with a credential scoped to a dedicated
non-production Cloudflare account. PR code must be treated as untrusted: do not claim
that the preview API token or preview data is secret from the PR author. The derived
per-PR login bearer is intentionally public, and previews may contain only synthetic
or explicitly non-sensitive data. Production credentials, OAuth grants, transcript
storage and managed-CA operations must remain inaccessible to that account and code.

Audit `infra/cf/preview-trust.mjs` and its callers for approved-account enforcement,
rejection of production identifiers in generated configuration, PR-owned resource
names, contained regular-file inputs, the 20 MiB Worker bundle cap, and allowlisted
migration-directory entries. Generated preview configs have no production routes,
Durable Object bindings or exports; `hub/src/preview.ts` exports the app handler,
not the production CloudflareOAuthBroker. Child-process environment filtering limits
accidental credential propagation but is not a sandbox against malicious PR code.
Audit HTTP/body and parser caps separately, including the QA intake's streamed
256 KiB cap, JSONL line limits and multipart declared-versus-actual sizes; collector
defaults or infrastructure body limits do not prove every server-side input bounded.

Production migrations are applied by the protected-main production CI job using a
separate D1-only migration credential, current-main stale-run rejection, migration
manifests/baselines and deployment journals. Preview migrations intentionally come
from the PR and operate only on its disposable resources; their provenance must
never authorize production migration or resource selection. An unused
`hub/src/auth/github-oidc.ts` helper is not evidence of an active GitHub-OIDC migration
authorization path in the application router.

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

### Independent CLI and synthetic HTTP runtime smoke

The following additional smoke is independent of the test runners. Execute it inside
the scanner's root shell with networking disabled. Both commands invoke the installed
application console entries, not a Python import or a test wrapper; `--help` exits
before reading credentials, opening a browser or performing an HTTP operation.

```sh
uv run --directory /src/collector --offline --no-sync agent-collector --help
uv run --directory /src/client --offline --no-sync agent-sessions --help
```

Create this throwaway driver inside the container; it is not a fourth enrollment
file or a permanent project test. It starts the existing real local Worker runner,
then independently uploads a fresh synthetic session, waits for real queue-driven
FTS indexing, reads normalized and raw content, and checks the rendered HTML. It
uses neither test mocks nor a manually drained queue.

```sh
cat > /tmp/oss-sessions-smoke.mjs <<'JS'
import assert from 'node:assert/strict';
import { spawn } from 'node:child_process';
import { createHash, randomUUID } from 'node:crypto';
import { mkdir, mkdtemp, rm } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { setTimeout as sleep } from 'node:timers/promises';

// Throwaway driver: real repository local runner and real loopback HTTP, no test mocks.
const home = await mkdtemp(join(tmpdir(), 'oss-sessions-http-smoke-'));
await mkdir(join(home, '.config'), { recursive: true });
const env = {
  PATH: process.env.PATH,
  HOME: home,
  USERPROFILE: home,
  XDG_CONFIG_HOME: join(home, '.config'),
  LANG: 'C.UTF-8',
  CI: '1',
  WRANGLER_SEND_METRICS: 'false',
  NO_COLOR: '1',
};
const child = spawn(process.execPath, [
  '/src/hub/scripts/environment.mjs', 'e2e', '--port', '0', '--readiness-ms', '180000',
], { cwd: '/src/hub', env, stdio: ['ignore', 'pipe', 'pipe', 'ipc'] });
let logs = '';
let startupTimer;
let closeInfo;
const closed = new Promise((resolve) => child.once('close', (code, signal) => {
  closeInfo = { code, signal };
  resolve(closeInfo);
}));
const ready = new Promise((resolve, reject) => {
  startupTimer = setTimeout(() => reject(new Error('local runner startup exceeded 300 seconds')), 300000);
  child.once('error', reject);
  child.once('exit', (code, signal) => reject(new Error(`local runner exited before readiness: ${code ?? signal}`)));
  child.stderr.on('data', (chunk) => { logs = (logs + chunk).slice(-16384); });
  child.stdout.on('data', (chunk) => {
    logs = (logs + chunk).slice(-16384);
    const match = logs.match(/Local hub ready: (http:\/\/127\.0\.0\.1:\d+)\r?\n/);
    if (match) resolve(match[1]);
  });
});

async function request(base, path, init = {}) {
  const response = await fetch(new URL(path, base), { ...init, signal: AbortSignal.timeout(5000) });
  return { response, text: await response.text() };
}

try {
  const base = await ready;
  clearTimeout(startupTimer);
  const health = await request(base, '/healthz');
  assert.equal(health.response.status, 200, health.text);
  const diagnostics = JSON.parse(health.text);
  assert.equal(diagnostics.pendingMigrations, 0);

  const machine = 'oss-http-smoke';
  const sessionId = randomUUID();
  const marker = `offline synthetic obsidian lantern ${sessionId}`;
  const common = { isSidechain: false, cwd: '/workspace/oss-synthetic', sessionId, version: '2.1.99', gitBranch: 'main' };
  const content = [
    { ...common, type: 'user', userType: 'external', uuid: 'smoke-user', parentUuid: null,
      timestamp: '2026-07-02T10:00:00.000Z', message: { role: 'user', content: marker } },
    { ...common, type: 'assistant', uuid: 'smoke-assistant', parentUuid: 'smoke-user', requestId: 'oss-smoke-request',
      timestamp: '2026-07-02T10:00:05.000Z', message: { id: 'oss-smoke-message', role: 'assistant', model: 'claude-test-1',
        content: [{ type: 'text', text: `Synthetic acknowledgement: ${marker}` }], usage: { input_tokens: 11, output_tokens: 7 } } },
  ].map((line) => JSON.stringify(line)).join('\n') + '\n';
  const bytes = Buffer.from(content);
  const digest = createHash('sha256').update(bytes).digest('hex');
  const relpath = `-workspace-oss-synthetic/${sessionId}.jsonl`;
  const headers = { 'x-dev-machine': machine };
  const upload = await request(base, `/api/v1/files/${machine}/claude-projects/${encodeURIComponent(relpath)}`, {
    method: 'PUT', body: bytes,
    headers: { ...headers, 'x-content-hash': `sha256:${digest}`, 'x-file-mtime': '2026-07-02T10:00:05Z', 'content-length': String(bytes.length) },
  });
  assert.ok([200, 201].includes(upload.response.status), `upload ${upload.response.status}: ${upload.text}`);

  const deadline = Date.now() + 60000;
  let hits = [];
  do {
    const query = new URLSearchParams({ q: 'obsidian lantern', machine, limit: '20' });
    const search = await request(base, `/api/v1/search?${query}`, { headers });
    assert.equal(search.response.status, 200, search.text);
    hits = JSON.parse(search.text).hits;
    assert.ok(Array.isArray(hits), search.text);
    if (hits.some((hit) => hit.session_id === sessionId)) break;
    await sleep(250);
  } while (Date.now() < deadline);
  assert.ok(hits.some((hit) => hit.session_id === sessionId), 'new upload did not reach real queue-driven FTS indexing within 60 seconds');

  const normalized = await request(base, `/api/v1/sessions/${sessionId}`, { headers });
  assert.equal(normalized.response.status, 200, normalized.text);
  const session = JSON.parse(normalized.text).session;
  assert.equal(session.harness, 'claude-code');
  assert.equal(session.turns.length, 2);
  assert.ok(normalized.text.includes(marker), 'normalized read lost synthetic marker');
  const raw = await request(base, `/api/v1/sessions/${sessionId}/raw`, { headers });
  assert.equal(raw.response.status, 200, raw.text);
  assert.equal(raw.text, content, 'raw HTTP read differs from uploaded bytes');
  const html = await request(base, `/s/${sessionId}`);
  assert.equal(html.response.status, 200, html.text);
  assert.match(html.response.headers.get('content-type') ?? '', /^text\/html\b/);
  assert.ok(html.text.includes(marker), 'HTML viewer did not render the uploaded synthetic session');

  console.log(JSON.stringify({ smoke: 'PASS', base, environmentId: diagnostics.environmentId,
    pendingMigrations: diagnostics.pendingMigrations, uploadStatus: upload.response.status,
    uploadedBytes: bytes.length, sha256: digest, sessionId, normalizedTurns: session.turns.length,
    searchHit: true, rawBytesEqual: true, htmlRendered: true }));
} catch (error) {
  console.error(logs);
  throw error;
} finally {
  clearTimeout(startupTimer);
  if (!closeInfo && child.connected) child.send({ type: 'shutdown' });
  const forceStop = setTimeout(() => child.kill('SIGTERM'), 20000);
  const hardStop = setTimeout(() => child.kill('SIGKILL'), 30000);
  await closed;
  clearTimeout(forceStop);
  clearTimeout(hardStop);
  await rm(home, { recursive: true, force: true });
  assert.equal(closeInfo.code, 0, `local runner teardown failed: ${JSON.stringify(closeInfo)}`);
}
JS
node /tmp/oss-sessions-smoke.mjs
```

Require exit status zero in addition to the JSON success line: teardown failures
must also fail the smoke. During enrollment verification, this exact driver ran
in the built scanner image with `--network none`, 2 CPUs and 8 GiB memory. Both real
CLI help entries passed; the HTTP smoke returned 201 for a 914-byte synthetic upload,
zero pending migrations, a real FTS search hit, two normalized turns, byte-identical
raw reads and the rendered HTML marker. Each run uses a new UUID and therefore a new
content SHA-256; the driver computes and checks its own payload, not a fixed digest.
This is local application runtime proof, not production-authentication certification.

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
