# Threat model: Prividium Core

## What this project does and where untrusted input enters

Prividium Core is the permissioning layer that stands between every client and a ZKsync Prividium chain's node. In a
Prividium deployment the node itself is on a private network and this service is the only path to it, so a bypass of
this code is a bypass of the chain's entire access-control model: who may call which contract function, who may see
which transactions, balances and logs, and who may transact at all.

- `apps/permissions-api/` is a Fastify service with two halves:
  - **The RPC proxy** (`src/rpc/`, `src/routes/rpc-routes.ts`, `regular-rpc-routes.ts`): accepts JSON-RPC over HTTP,
    enforces a method allowlist (`src/rpc/methods/`, `src/rpc/permissions/`), validates and scopes the parameters of
    permitted methods — `eth_call`/`eth_estimateGas` target address and 4-byte selector against the caller's contract
    permissions, log filters against contracts the caller may see, raw transactions decoded and the sender recovered
    from the signature — and forwards what passes to the node (`target-rpc.ts`).
  - **The control plane** (`src/routes/`, `src/services/`, `src/middleware/`, `src/permissions/`): authentication
    (SIWE, OIDC sessions, M2M API keys, WebAuthn step-up), users, roles and system permissions, contracts and
    per-function contract permissions, organizations, applications, audit logs; plus `/admit` and `/judge`
    (`admit-routes.ts`, `judge-routes.ts`), which a sequencer configured for it calls to re-check transactions before
    inclusion.
- `packages/prividium-sdk/` is the TypeScript SDK clients embed: SIWE and session handling, the RPC transport, and the
  selective-disclosure verifier (`src/selective-disclosure/`).
- `packages/api-kit/`, `packages/access-control/`, `packages/api-metrics/` are the shared Fastify, database,
  permission and metrics helpers the API is built from.

How all of this is meant to behave — zones, organizations, roles, contract and RPC permissions, the SDK — is publicly
documented at [docs.prividium.zksync.io](https://docs.prividium.zksync.io).

Untrusted input:

- Every HTTP request. Pre-authentication: the SIWE challenge and login flow, OIDC callbacks, M2M token issuance,
  invite redemption. And the attacker holds valid credentials too — the privilege levels differ, the model does not:
  an ordinary user is untrusted against admin and organization boundaries, a wallet-token holder against everything
  beyond that wallet's RPC scope, and an M2M application — often a powerful actor that provisions users and transacts
  for its organization — against everything its assigned roles do not grant, other organizations included.
- JSON-RPC bodies in full: method names, params, batches, `eth_call` calldata, filter objects, raw signed
  transactions.
- The unauthenticated surface deserves particular attention. Before any credential is checked, the service parses
  bodies, validates schemas, runs rate-limit accounting, and serves the pre-auth routes: `auth-routes.ts` (SIWE
  challenge and login, OIDC callbacks, M2M token issuance), `public-check-routes.ts`,
  `organizations-public-routes.ts`, `public-applications-routes.ts`, `dot-well-known-routes.ts`, `health-routes.ts`,
  `info-routes.ts`, and the rejection path of `/rpc` itself. Every one of these code paths runs for anyone who can
  reach the proxy, and this service is the only path to the chain — so what an unauthenticated request can make the
  process spend (database work, crypto, memory, pool connections) is a first-class question, not an afterthought.
- OIDC claims: the identity provider is trusted to authenticate its own users — a claim says who is calling, never
  what they may do. Roles and permissions live in this service's own records, so a code path that turns a claim value
  (an email, a group, an issuer) into authority it did not look up here is a finding.
- `/admit` and `/judge` payloads: these are served on a separate mTLS-only listener (`build-policy-app.ts`), so the
  channel authenticates the calling sequencer — but the transactions inside remain user-authored. The caller is itself
  open source — the ZKsync OS sequencer's policy client, in
  [matter-labs/zksync-os-server](https://github.com/matter-labs/zksync-os-server) under
  `lib/tx_validators/src/policy_client/`.
- In the SDK: everything received from the server, and disclosure proof material presented for verification.

## Trust assumptions

- **Topology.** The node is reachable only through this service; a deployment that exposes the node directly is a
  deployment error, out of scope here. What is in scope is everything this code lets through.
- **Roles.** A zone is one deployment's whole permission domain; zone administrators are trusted for all of it.
  Organization admins are trusted only inside their own organization: reading or writing another organization's
  users, roles, contracts or credentials, or zone-level state, is a finding. Organization scoping is gated by
  `MULTI_ORG_ENABLED`; name the flag when a report depends on it. Users, M2M applications and wallet-token holders
  are trusted for exactly their grants — an M2M application's grants are often broad, which raises the impact of
  escaping them, not the trust.
- **Proxy-layer limits are documented.** Permissioning happens before the request reaches the chain; it does not see
  inside EVM execution. The six inherent constraints — multicall/arbitrary-call trampolines inside permitted
  contracts, gas-estimation oracles, wall-clock timing of EVM execution, constructor calls during permissioned
  deployment, selective-disclosure integrity limits, and application tokens carrying the signed-in user's RPC
  authority — are published in the Prividium docs
  ([Security Limitations of the Permissioning Layer](https://docs.prividium.zksync.io/developers/applications/security-limitations))
  and are
  operator-side constraints on what contracts to deploy and what to enable, not bugs in this code. Report them only
  where this code fails to enforce what it does claim to check: the top-level target and selector, the method
  allowlist, and the declared parameter scoping.
- **Dev switches do what they say.** Configuration documented as insecure-for-development (`INSECURE_`-prefixed
  variables, dev wallet allowlists, the faucet) is out of scope when deliberately enabled. A production configuration
  that silently degrades into one of these states is in scope.
- **Sequencer-side enforcement is optional.** `/admit`/`/judge` are defense in depth; their absence in a deployment is
  not a finding. The proxy is the load-bearing layer.
- The database and its migrations are trusted infrastructure; queries built from request input must still be
  injection-safe and actor-scoped.

## Components that matter most / least

Highest priority (each is a boundary whose failure is Critical or High):

- `src/rpc/` in full: method routing and the allowlist, per-method parameter validation and scoping
  (`params-schemas.ts`, `methods/`, `permissions/`), raw-transaction decoding and sender recovery, the deployment
  context (`contract-deployment-context.ts`), and error mapping (`errors.ts`, `filtered-read-error.ts`) — responses
  for "forbidden" and "nonexistent" are deliberately uniform, and a divergence is an oracle.
- `src/middleware/`: `sessions-auth.ts`, `api-key-auth.ts`, `require-org-admin.ts`, `require-org-membership.ts`,
  `require-step-up.ts`, `user-access-control.ts`, `rate-limit-middleware.ts`.
- Authentication routes and services: `auth-routes.ts`, SIWE challenge/verification, session issuance, expiry and
  revocation, `jwt-validator-service.ts`, `api-key-service.ts`, `passkey-service.ts`/`admin-mfa-service.ts`,
  `erc1271-verifier.ts`.
- Authorization model: `src/permissions/`, `roles-routes.ts`, `contract-permissions-routes.ts`,
  `contract-event-permissions-routes.ts`, `authorization-service.ts`, `admit-service.ts`, `judge-service.ts`,
  `decision-cache.ts` (a cached decision must depend on nothing outside the request), `account-classifier.ts`,
  `event-permission-verifier.ts`.
- Organization isolation: `organizations-routes.ts` and every repository query reachable from an
  organization-scoped route — a query missing its actor- or organization-bound WHERE clause is the recurring bug
  class here.
- M2M credentials: `m2m-app-*` routes and services (issuance, rotation, IP allowlists).
- The pre-authentication pipeline: body and batch size limits, schema validation cost, `rate-limit-middleware.ts` and
  the quotas on the auth routes, and any pre-auth path that reaches the database or does expensive cryptography — the
  cheapest unauthenticated request must not be able to buy expensive work or hold a scarce resource (a database pool
  connection, unbounded memory) past its response.
- The SDK: `siwe-auth.ts`, `verify-user-access-token.ts`, and the selective-disclosure verifier — accepting a proof
  for a claim it does not actually demonstrate is a finding even though the methods ship disabled by default. The SDK
  also runs inside third-party web applications, so its client-side behaviour is in scope: popup and postMessage
  origin handling in `popup-auth.ts`, token storage and transport (`storage.ts`, `memory-storage.ts`), and any path
  that lets a session or wallet token reach another origin, a URL or a referrer.

Lower priority: audit logging (evading the audit record for a privileged mutation matters; read-path shape less so),
`faucet-routes.ts` (dev tooling), `docs-routes.ts`, `health-routes.ts`, `info-routes.ts`, metrics.

Out of scope: `*.test.ts` and `test/` fixtures, `packages/test-deps/`, `scripts/`, `.github/`, `.env.example` values
(placeholders by design; running them in production is operator error, per the trust assumptions above).

## How to exercise it

- `start-test-db && pnpm test` — starts the PostgreSQL the tests expect on `localhost:5500` and runs every vitest
  project. The image already ran this once at build time, so everything is compiled and a run takes minutes, offline.
- One package: `pnpm --filter @repo/permissions-api test`, or any vitest filter. Unit and route tests sit next to the
  source as `*.test.ts`.
- Route tests build the real Fastify app against a per-worker clone of the migrated database, so the most useful proof
  of concept is a route-level test that issues real HTTP requests as a concrete actor; copy the fixtures from an
  existing `src/routes/*.test.ts`. Name the file and test in the report.
- The full REST surface is enumerated in this image: the build exports the OpenAPI spec to
  `packages/api-types/permissions-api/spec/openapi-spec.json`, so every route, schema and security annotation can be
  read from one file. A running deployment serves the same spec as a login-gated interactive reference at `/docs`.
- No chain node ships in this image: forwarding beyond the authorization decision, and `/admit`/`/judge` against a
  live sequencer, have no harness here — reason from the code for those. For the caller's half of the `/admit` and
  `/judge` exchange (wire format, and how a verdict is enforced before inclusion), read
  `lib/tx_validators/src/policy_client/` in matter-labs/zksync-os-server.
- The admin and user panels are separate applications in other repositories, so cross-site scripting in those UIs is
  out of scope here. In this repository the client-side surface is the SDK; the API itself serves JSON, plus the
  gated interactive API reference behind `docs-routes.ts`.

## How we rate severity

- **Critical:** anything that reaches the node, or returns chain data, without a grant that covers it — a method
  outside the allowlist going through, a permitted method's parameters escaping their declared scope (an `eth_call`
  to an unpermitted contract or selector, a log filter crossing its contract scope, a raw transaction whose effective
  sender is not the recovered signer); authentication bypass or credential forgery (a token accepted for a purpose it
  was not issued for, a SIWE verification flaw, M2M key confusion); reaching another organization's data, credentials
  or chain scope, whether directly (a query that ignores the organization boundary) or through escalation that ends in
  zone-wide power; an unauthenticated denial of service that halts the zone — a
  request stream no credential ever touched that crashes the process, wedges the database pool, or blocks the event
  loop stops every user and every application at once, because this service is the only path to the chain; SQL
  injection; remote code execution.
- **High:** privilege escalation whose gain stays inside the attacker's own organization (a user widening their own
  grants within their organization's scope) — the dividing line from Critical is the reach of what is gained, not the
  mechanism; role or permission mutations taking effect without the required authority; session fixation or replay
  across actors; a selective-disclosure proof accepted for a claim it does not prove; unauthenticated resource
  exhaustion that degrades the authentication or proxy path short of a full halt (connection-pool pressure, unbounded
  pre-auth work, amplification through batches); an SDK flaw that hands a
  session or wallet token to another origin (popup or postMessage origin confusion).
- **Medium:** oracles this code introduces that cross an isolation boundary (errors, status codes, response sizes or
  timing that distinguish another actor's "exists but forbidden" from "does not exist"); rate-limit bypass on
  authentication-adjacent routes; denial of service that requires authentication; audit evasion of bounded scope.
- **Low:** timing or size oracles already bounded by design caps (batch limits, pagination caps); disclosure of
  non-secret configuration; missing hardening headers; anything that requires a trusted role to misconfigure the
  system first.

## How we would like reports and patches

- One root cause per report. State the actor (unauthenticated; authenticated user; organization admin; M2M
  application; wallet-token holder; zone admin) and any non-default configuration the attack needs.
- Where the code can be exercised here, the reproducer is a vitest test that runs offline with the commands above.
- Keep patches minimal. Database changes go in new drizzle migrations — never edit an applied migration. REST
  response schemas feed a generated client (`packages/api-types`), so say so when a schema changes.

## Anything to leave alone

- The six documented permissioning-layer limitations above, when exercised through contract code rather than a gap in
  what this service claims to check.
- The wallet RPC token travelling in the URL path (`/rpc/wallet/:token`): the documented design for header-less
  wallet integrations, bounded by its own issuance and invalidation flow.
- Step-up/passkey protection being opt-in per account: documented design.
- `/health` exposing the running version: a monitoring contract.
- Request validation running before authorization: the schemas are public source.
- The faucet and anything gated behind a documented dev-only flag.
- Style, comments and performance micro-optimizations.
