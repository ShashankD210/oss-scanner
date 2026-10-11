# Threat model: watch-pr hosted MCP service

## Project and audit scope

This repository is the hosted Cloudflare Worker/MCP implementation, not the
separate agent-plugins watch-pr client. It brokers GitHub App user OAuth,
maintains account-scoped pull-request watches in a SQLite-backed Durable Object,
applies authenticated GitHub webhook payloads, reconciles snapshots through
GitHub REST/GraphQL, and exposes MCP Streamable HTTP and read-only SSE monitors.
The same repository also contains an OTLP telemetry gateway, an OIDC discovery
Worker, and deployment/telemetry configuration scripts.

Audit the entire retained checkout: `src/`, `test/`, `scripts/`, `.github/`,
`docs/`, and operator documentation. The image keeps original source and all
locked production/development dependencies. `build/worker.js`,
`build/telemetry-gateway.js`, and `build/oidc-issuer.js` are ESM/browser/ES2022
bundles with external source maps and embedded source content. The build follows
the esbuild invocation in `.github/workflows/deploy-pr.yml`, adding source maps
and the two other Worker entrypoints. There is no project-owned native Windows
binary to cross-compile; workerd, esbuild, and Rollup are platform-specific
upstream development/runtime dependencies installed for the image's platform.

The enrollment is preparation, not a claim that Anthropic has accepted the
project, that a scan has completed, or that the project has broad adoption.

## Assets, adversaries, and entrypoints

Protect GitHub access/refresh tokens, broker bearer tokens, webhook secrets,
private PR bodies/comments/reviews, per-account snapshots and event history,
monitor capability records, OAuth redirect/state/PKCE bindings, persistent
revocation records, availability, and the user's GitHub API/Worker/storage quota.

Treat these as untrusted input:

- Every public HTTP method, URL, path, query, header, Origin, content type, body,
  MCP session ID, JSON-RPC method/params, and SSE cursor.
- Dynamic client registration metadata and HTTPS/loopback redirect URIs; clients
  are public OAuth clients, not trusted operators.
- GitHub webhook JSON and event/delivery headers, even after authenticating the
  transport. A user with permission to author a PR/comment can influence payload
  fields; authentication does not make their contents trustworthy.
- GitHub REST/GraphQL response JSON, pagination URLs, validators, repository
  identifiers, PR numbers, comments/review text, timestamps, check/deployment URLs,
  and the sizes/shapes/order of all collections.
- Previously persisted state, including legacy and chunked/sidecar records;
  concurrent requests, replays, refreshes, object restarts, and partial failures.
- OTLP logs/traces, gzip streams, arbitrary attribute keys/nested values, and
  malformed deployment key/bearer rotation configuration.

Likely adversaries include unauthenticated callers, a malicious authenticated
GitHub user, a registered malicious OAuth client, a contributor controlling PR
content, a holder of one leaked monitor URL, and a caller trying to exhaust the
single global Durable Object. OAuth/GitHub/Cloudflare/Azure credentials belonging
to real users are never available in this scanner image.

## Important trust boundaries and invariants

### Public Worker routing and MCP authentication

`src/index.ts` exposes `/`, `/health`, icons, OAuth discovery, `/mcp`,
`/webhooks/github`, `/monitor/{capability}`, and four `/oauth/*` routes.
Only those public routes are dispatched into `WatchPrHub`; the scheduled handler
makes the internal `/internal/poll` request directly through the HUB binding.
A caller sending `/internal/poll` to the public Worker must not reach it.

`src/hub.ts` validates the broker bearer before serving MCP. Stateful transport
IDs must not authorize a different bearer; closed MCP IDs have persisted
session-scoped tombstones, and recovered transports must retain the account's
watch/subscription restrictions. `src/mcp.ts` validates repository/number inputs,
resource URIs, subscriptions, and membership in the active watch set.
The CORS policy deliberately permits arbitrary client origins; CORS is not an
authorization mechanism, and the service does not rely on an ambient login cookie.

### OAuth, PKCE, and token lifecycle

The broker registers one to ten unique HTTPS or loopback HTTP redirects. It
rejects fragments/userinfo and matches registered redirects; loopback ports may
vary, but hostname/path/query must match. Authorization requires `state`, a
registered client, `response_type=code`, and S256 PKCE. Separate random internal
state binds the GitHub callback to the client request. Broker codes bind client,
redirect, PKCE challenge, and GitHub identity and are consumed before exchange;
states/codes have a bounded lifetime. Audit races and replay, not just sequential
success cases. Token/session refresh, invalid GitHub authorization, expiration,
and revocation must not resurrect authority or leak another user's stored state.
The service uses GitHub App user tokens; it does not hold an App private key.

### Webhook HMAC and asynchronous fanout

`src/crypto.ts` computes HMAC-SHA256 over the received body and compares the full
`sha256=` signature; missing secrets/signatures fail closed. `src/hub.ts` verifies
this before admitting payload processing. Unauthenticated requests must not
change coverage, snapshots, event history, or delivery-deduplication state.
Do not treat delivery IDs as authentication. Deduplication includes in-flight,
per-watch, and durable completed-delivery state; partial fanout must resume safely
without duplicating completed watches or suppressing remaining ones.

Valid deliveries are acknowledged before asynchronous fanout completes. A 202
is not proof that all watches updated; scheduled reconciliation covers missed or
failed work. Payload reducers in `src/reducers.ts`, `src/events.ts`,
`src/github.ts`, and `src/schedule.ts` must respect account scoping, timestamps,
head revisions, conditional validators, webhook coverage, and terminal state.
Headers that select unsupported event kinds are still attacker-controlled.

### Monitor capabilities, replay, and revocation

Monitor URLs carry a random read-only capability, not a GitHub/broker credential.
They deliberately authorize access without an Authorization header. Possession
can expose that capability's PR event stream; treat it as scoped read authority,
not as a general account credential or a URL with no confidentiality impact.
The capability binds a broker session, numeric GitHub user ID, repository, PR,
creation time, and deadline. Reuse must not extend the original 12-hour maximum;
the deadline is also bounded by session expiration. Monitor admission rechecks
the persisted session and watched PR. Unwatch, invalid/expired sessions, and
capability expiry must revoke persisted authority and close existing feeds.

SSE cursors are bounded and prefer Last-Event-ID over a query cursor. Audit
cross-scope replay, terminal handling, cursor misses, backpressure, reconnect
storms, and races between opening a feed and revoking it. MCP resource
notifications are advisory; durable snapshots/events and monitor replay are the
correctness path. PR/comment bodies are data, not instructions or executable
code; findings involving prompt injection need a concrete project trust-boundary
violation, not merely malicious text that a downstream LLM could read.

### Durable Object persistence and resource use

One global HUB object stores OAuth clients, requests, codes, sessions, monitor
scopes, coverage, scheduling, snapshots, event history, and deduplication state.
Watch data is keyed by numeric GitHub user ID plus repository/PR; capabilities
are additionally session-scoped. Multiple sessions of the same GitHub account
may share watch snapshots; different numeric users must not.
Audit transaction/locking semantics, replay after restart, session invalidation,
legacy migration, chunk integrity and batch bounds, sidecar indexing and cleanup,
immutable snapshot/payload ownership, lost updates, and stale refresh races.
The event history is bounded to 100 entries, but this is not a universal byte,
request, registration, or quota limit. The documented 10,000-delivery storage
budget applies to its specific bounded regression workload, not arbitrary large
payloads. Production has an external monitor WAF rule, which is not a Worker
code-level defense or a distributed-DoS guarantee and is absent offline.

### Telemetry and deployment helpers

The separate `src/telemetry-gateway.ts` authenticates configured ingest bearers,
bounds compressed/decompressed payloads, sanitizes OTLP with a drop-by-default
allowlist, encodes protobuf, signs workload assertions, caches Entra tokens, and
forwards to operator-configured Azure destinations. Check for secret/private-PR
leakage through retained attributes, error/log text, route/capability paths,
compressed input, key rotation, JWT claims, and cross-environment caching.
`src/oidc-issuer.ts` publishes configured public JWKs/discovery and must not
publish private key material or accept mismatched environment/key identifiers.
The configuration/verification scripts and GitHub workflows remain in scope for
credential handling and untrusted PR versus trusted deployment-source separation.

## Offline build and existing tests

The Dockerfile installs Node 24 (the manifest requires Node >=22), OS tools, and
`npm ci` with development/optional dependencies and install scripts enabled
while networking is available. It retains workerd/Miniflare, esbuild, Vitest,
TypeScript, dependency source/notices, and the lockfile. It does not run deploy,
secret-upload, OAuth, federation-verification, or observability provisioning
commands. No private `.env`, `.dev.vars`, credentials, or production state should
be added to the clean source checkout/build context.

Run after building, inside the network-disabled image:

```sh
cd /src
npm test
npm run typecheck
```

For a focused security pass using existing tests, not replacement tests:

```sh
cd /src
./node_modules/.bin/vitest run test/oauth.test.ts test/monitor.test.ts test/mcp.test.ts test/github.test.ts test/reducers.test.ts test/schedule.test.ts test/events.test.ts
./node_modules/.bin/vitest run test/telemetry-gateway.test.ts test/oidc-issuer.test.ts test/otlp-sanitize.test.ts test/otlp-protobuf.test.ts test/*.test.mjs
```

`npm test` includes all 15 current suites, including the `.mjs` script tests.
Tests stub GitHub/Entra/Cloudflare network responses and use synthetic keys,
payloads, tokens, and in-memory Durable Object storage. They exercise real
project handlers/reducers/transports, including auth, PKCE, HMAC, monitor expiry,
revocation/replay, transport recovery, persisted/chunked/sidecar state, storage
write budgets, reducer races, API normalizers, pagination, and telemetry privacy.
MemoryStorage transactions are not the production SQLite runtime. Run the full
suite, not just the focused groups. Authoring did not run these checks; the image
and commands still require the parent's final build/offline verification.

## Synthetic actual Worker/MCP request smoke

This command runs the built Worker and real SQLite-backed HUB under the locked
Miniflare/workerd runtime. It uses actual Request/Response dispatch, not a health
placeholder. Only token-exchange/profile calls receive synthetic success
responses; all other outbound requests receive an explicit synthetic 503 and
never reach the Internet. Redirects are inspected without following them. The
synthetic watch intentionally has no successful GitHub PR snapshot: this tests
registration, stored watch membership, capability reuse/revocation, and MCP
transport persistence, not real GitHub permission/data fidelity. Existing tests
above cover rich snapshots, webhook fanout, cursor replay, expiry and races.

Miniflare 5 in the lockfile exports a converter for the previous options shape;
the command uses that supported converter and `resourcePersistencePath` rather
than the removed `durableObjectsPersist` option. State is disposable and unique
per invocation, then removed. No real credentials or deployed bindings are used.

```sh
cd /src
timeout 180s node --input-type=module <<'NODE'
import assert from 'node:assert/strict';
import { createHash, createHmac } from 'node:crypto';
import { mkdtemp, rm } from 'node:fs/promises';
import { Miniflare, Response as MFResponse, convertV4MiniflareOptions } from 'miniflare';

const base = 'http://localhost';
const redirectUri = 'https://client.example.test/callback';
const stateDirectory = await mkdtemp('/tmp/watch-pr-smoke-');
const outbound = [];
const options = convertV4MiniflareOptions({
  name: 'watch-pr-offline', rootPath: '/src',
  modules: true, scriptPath: '/src/build/worker.js',
  compatibilityDate: '2026-09-03', cf: false,
  telemetry: { enabled: false }, resourcePersistencePath: stateDirectory,
  durableObjects: { HUB: { className: 'WatchPrHub', useSQLite: true } },
  bindings: {
    PUBLIC_BASE_URL: base, GITHUB_CLIENT_ID: 'offline-client',
    GITHUB_CLIENT_SECRET: 'offline-not-a-real-secret',
    GITHUB_WEBHOOK_SECRET: 'offline-webhook-secret',
  },
  outboundService: async (request) => {
    const url = new URL(request.url);
    outbound.push(`${request.method} ${url.origin}${url.pathname}`);
    if (url.origin === 'https://github.com' && url.pathname === '/login/oauth/access_token') {
      assert.equal(request.method, 'POST');
      return MFResponse.json({ access_token: 'synthetic-github-token', token_type: 'bearer' });
    }
    if (url.origin === 'https://api.github.com' && url.pathname === '/user') {
      return MFResponse.json({ id: 42, login: 'offline-user', name: 'Offline User', avatar_url: null, html_url: 'https://github.com/offline-user' });
    }
    return MFResponse.json({ message: 'External API deliberately unavailable in offline smoke' }, { status: 503 });
  },
});
let mf = new Miniflare(options);
const request = (path, init = {}) => mf.dispatchFetch(new URL(path, base), { redirect: 'manual', ...init });
let rpcId = 0;
async function rpc(bearer, method, params = {}, sessionId) {
  const id = ++rpcId;
  const headers = { authorization: `Bearer ${bearer}`, 'content-type': 'application/json', accept: 'application/json, text/event-stream' };
  if (sessionId) headers['mcp-session-id'] = sessionId;
  const response = await request('/mcp', { method: 'POST', headers, body: JSON.stringify({ jsonrpc: '2.0', id, method, params }) });
  assert.equal(response.status, 200, method);
  const text = await response.text();
  const messages = response.headers.get('content-type')?.includes('application/json')
    ? [JSON.parse(text)]
    : text.split(/\r?\n/).filter((line) => line.startsWith('data: ')).map((line) => JSON.parse(line.slice(6)));
  const message = messages.find((value) => value.id === id);
  assert.ok(message, `${method} response missing`);
  assert.equal(message.error, undefined, JSON.stringify(message.error));
  return { result: message.result, sessionId: response.headers.get('mcp-session-id') };
}
async function initialize(bearer) {
  const initialized = await rpc(bearer, 'initialize', { protocolVersion: '2025-06-18', capabilities: {}, clientInfo: { name: 'offline-smoke', version: '1.0.0' } });
  assert.ok(initialized.sessionId);
  const notified = await request('/mcp', { method: 'POST', headers: { authorization: `Bearer ${bearer}`, 'mcp-session-id': initialized.sessionId, 'content-type': 'application/json', accept: 'application/json, text/event-stream' }, body: JSON.stringify({ jsonrpc: '2.0', method: 'notifications/initialized' }) });
  assert.equal(notified.status, 202);
  await notified.text();
  return initialized.sessionId;
}
async function tool(bearer, sessionId, name, args = {}) {
  const { result } = await rpc(bearer, 'tools/call', { name, arguments: args }, sessionId);
  assert.notEqual(result.isError, true, JSON.stringify(result));
  return JSON.parse(result.content.find((item) => item.type === 'text').text);
}
try {
  const health = await request('/health');
  assert.equal(health.status, 200);
  assert.equal((await health.json()).service, 'watch-pr');
  const discovery = await request('/.well-known/oauth-authorization-server');
  assert.equal((await discovery.json()).token_endpoint, `${base}/oauth/token`);
  assert.equal((await request('/internal/poll', { method: 'POST' })).status, 404);
  assert.equal((await request('/mcp', { method: 'POST', body: '{}' })).status, 401);
  assert.equal((await request('/monitor/missing')).status, 404);
  assert.equal((await request('/webhooks/github', { method: 'POST', body: '{}' })).status, 401);
  const ping = JSON.stringify({ zen: 'synthetic' });
  const signature = `sha256=${createHmac('sha256', 'offline-webhook-secret').update(ping).digest('hex')}`;
  const webhookHeaders = { 'content-type': 'application/json', 'x-github-event': 'ping', 'x-github-delivery': 'offline-ping', 'x-hub-signature-256': signature };
  assert.equal((await request('/webhooks/github', { method: 'POST', headers: webhookHeaders, body: ping })).status, 202);
  assert.equal((await request('/webhooks/github', { method: 'POST', headers: webhookHeaders, body: `${ping} ` })).status, 401);

  const registration = await request('/oauth/register', { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ redirect_uris: [redirectUri] }) });
  assert.equal(registration.status, 201);
  const { client_id: clientId } = await registration.json();
  const verifier = 'offline-pkce-verifier-0123456789-abcdefghijklmnopqrstuvwxyz';
  const challenge = createHash('sha256').update(verifier).digest('base64url');
  const authorizePath = `/oauth/authorize?${new URLSearchParams({ client_id: clientId, redirect_uri: redirectUri, response_type: 'code', state: 'offline-client-state', code_challenge: challenge, code_challenge_method: 'S256' })}`;
  const invalidRedirect = new URL(authorizePath, base);
  invalidRedirect.searchParams.set('redirect_uri', 'https://attacker.example.test/callback');
  assert.equal((await request(invalidRedirect)).status, 400);
  async function brokerCode() {
    const authorized = await request(authorizePath);
    assert.equal(authorized.status, 302);
    const githubRedirect = new URL(authorized.headers.get('location'));
    assert.equal(githubRedirect.origin, 'https://github.com');
    const callbackPath = `/oauth/callback?${new URLSearchParams({ state: githubRedirect.searchParams.get('state'), code: 'synthetic-authorization-code' })}`;
    const callback = await request(callbackPath);
    assert.equal(callback.status, 302);
    const clientRedirect = new URL(callback.headers.get('location'));
    assert.equal(`${clientRedirect.origin}${clientRedirect.pathname}`, redirectUri);
    assert.equal(clientRedirect.searchParams.get('state'), 'offline-client-state');
    assert.ok(clientRedirect.searchParams.get('code'));
    assert.equal((await request(callbackPath)).status, 400);
    return clientRedirect.searchParams.get('code');
  }
  const exchange = (code, codeVerifier) => request('/oauth/token', { method: 'POST', body: new URLSearchParams({ grant_type: 'authorization_code', code, client_id: clientId, redirect_uri: redirectUri, code_verifier: codeVerifier }) });
  const code = await brokerCode();
  const tokenResponse = await exchange(code, verifier);
  assert.equal(tokenResponse.status, 200);
  const { access_token: bearer } = await tokenResponse.json();
  assert.ok(bearer);
  assert.equal((await exchange(code, verifier)).status, 400);
  assert.equal((await exchange(await brokerCode(), `${verifier}-wrong`)).status, 400);

  let sessionId = await initialize(bearer);
  const { result: listed } = await rpc(bearer, 'tools/list', {}, sessionId);
  assert.deepEqual(listed.tools.map((entry) => entry.name).sort(), ['get_pr', 'list_pr_events', 'list_watched_prs', 'unwatch_pr', 'watch_pr']);
  assert.deepEqual(await tool(bearer, sessionId, 'list_watched_prs'), []);
  const args = { repository: 'offline-owner/offline-repo', number: 7 };
  const watched = await tool(bearer, sessionId, 'watch_pr', args);
  assert.ok(watched.monitor.monitorUrl);
  const feed = await request(watched.monitor.monitorUrl);
  assert.equal(feed.status, 200);
  assert.match(feed.headers.get('content-type'), /text\/event-stream/);
  await feed.body.cancel();

  await mf.dispose();
  mf = new Miniflare(options);
  sessionId = await initialize(bearer);
  const persisted = await tool(bearer, sessionId, 'list_watched_prs');
  assert.equal(persisted.length, 1);
  assert.equal(persisted[0].repository, args.repository);
  assert.equal(persisted[0].number, args.number);
  const reused = await tool(bearer, sessionId, 'watch_pr', args);
  assert.equal(reused.monitor.monitorUrl, watched.monitor.monitorUrl);
  assert.equal((await tool(bearer, sessionId, 'unwatch_pr', args)).removed, true);
  assert.equal((await request(watched.monitor.monitorUrl)).status, 404);
  const closed = await request('/mcp', { method: 'DELETE', headers: { authorization: `Bearer ${bearer}`, 'mcp-session-id': sessionId } });
  assert.equal(closed.status, 200);
  await closed.text();
  await mf.dispose();
  mf = new Miniflare(options);
  const rejected = await request('/mcp', { method: 'POST', headers: { authorization: `Bearer ${bearer}`, 'mcp-session-id': sessionId, 'content-type': 'application/json', accept: 'application/json, text/event-stream' }, body: JSON.stringify({ jsonrpc: '2.0', id: ++rpcId, method: 'tools/list', params: {} }) });
  assert.equal(rejected.status, 404);
  assert.ok(outbound.some((entry) => entry === 'POST https://github.com/login/oauth/access_token'));
  assert.ok(outbound.some((entry) => entry === 'GET https://api.github.com/user'));
  console.log('PASS: actual offline Worker/SQLite DO OAuth+PKCE/HMAC/MCP/capability/revocation/restart smoke; outbound requests were intercepted');
} finally {
  await mf.dispose();
  await rm(stateDirectory, { recursive: true, force: true });
}
NODE
```

## Severity guidance

- Critical: demonstrated systemic account/credential compromise or executable
  code compromise of the hosted service with broad cross-user consequences.
- High: authentication/PKCE/HMAC bypass that grants unauthorized authority;
  cross-account/private-PR data disclosure; GitHub/broker-token or signing-key
  exposure; persisted capability/session revocation bypass; impactful persistent
  corruption or malicious forwarding that crosses a real credential boundary.
- Medium: reproducible availability or quota exhaustion, including unauthenticated
  registration/webhook/monitor abuse, retained-state growth, decompression work,
  broken account-scoped availability, and bounded race/replay correctness failures
  without a demonstrated confidentiality/authentication escalation.
- Low/informational: limited presentation/documentation issues or leakage of
  genuinely public metadata with no concrete additional authority or private data.

Report attacker prerequisites, exact route/input, affected account/credential
scope, persistence across restart, practical size/rate, and whether an external
platform control is required. Do not claim global impact from a single synthetic
fixture or dismiss a code-level flaw because production currently has a WAF rule.

## Limits and things to leave alone

Do not contact `watch-pr.vza.net`, PPE/preview Workers, GitHub OAuth/API/webhooks,
Cloudflare APIs, Azure/Entra, or observability endpoints. Do not run deployment,
secret-upload, observability configuration, or live federation-verification
scripts; only their mocked tests are safe offline. No real App/OAuth private
credentials, user tokens, telemetry ingest bearers, or production Durable Object
state may be copied in. Use synthetic credentials and local responses exclusively.

Offline checks do not validate real GitHub OAuth/App permissions/installations,
production credentials or revocation latency, live webhook subscriptions, WAF/DNS,
cron delivery, Cloudflare eviction/network behavior, Azure federation/destinations,
or production migrations. The Miniflare smoke checks real local SQLite storage,
not full distributed/cloud operational equivalence. It intentionally returns 503
for PR/installations/GraphQL APIs, so it cannot establish real snapshot freshness
or a successful scheduled reconciliation. Existing mocked tests are the available
rich-data oracle; full operational deployment needs separate authorized work.

The lockfile pins workerd `1.20260831.1`, while the checked-in Worker configuration
uses compatibility date `2026-09-03`. The smoke keeps that configured date; any
runtime warning or fallback during final verification must be recorded rather
than presented as proof of exact production compatibility.

Keep third-party npm dependency license/notices intact; the root MIT grant is
for original watch-pr project code, not a relicensing of MCP SDK, Cloudflare,
Vitest, TypeScript, esbuild, workerd, or other dependencies. No third-party
vendored source/license constraint was identified in the tracked source audit.
