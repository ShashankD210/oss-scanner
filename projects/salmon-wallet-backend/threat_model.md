This threat model tells security reviewers which Salmon API failures can put wallet users, transaction integrity, service secrets, or personal data at risk.

# Threat model

## What this project does

The Salmon API is the public backend used by Salmon Wallet. It provides Solana and Bitcoin account data, token prices and metadata, network information, decentralized-application metadata, optional analytics ingestion, and Powerup transaction construction.

Powerup endpoints prepare unsigned transactions for the wallet to inspect and sign on the user's device. The backend must never receive or possess a recovery phrase, private key, decrypted wallet credential, or signed Solana transaction. Bitcoin transactions are also signed and broadcast by the client rather than by this service.

The service runs as AWS Lambda functions behind API Gateway. Redis stores cached data, request counters, provider-control state, and sanctions information. Scheduled jobs refresh token, price, sanctions, and staking information. Production secrets are read from AWS Systems Manager Parameter Store and are not stored in this repository.

## Security goals

- The API must never request, receive, log, store, or expose wallet credentials or recovery phrases.
- Every unsigned transaction returned to a client must match the user's request and the documented Powerup behavior, including recipient, asset, amount, program, fee account, and instructions.
- Attacker-controlled request data or upstream responses must not inject arbitrary instructions, substitute accounts, redirect fees, or cross network boundaries.
- Network identifiers and environment selection must not allow a request to escape its intended blockchain or stage.
- Provider credentials, AWS configuration, Redis credentials, analytics secrets, and other server-side secrets must remain confidential.
- Public endpoints must validate and bound inputs before using memory, CPU, Redis, provider quotas, or outbound network access.
- Cached or shared state must not leak information between users, networks, accounts, stages, or request types.
- A failure in an upstream provider must not silently produce a valid-looking but security-significant response from stale, partial, or mismatched data.
- Optional analytics must enforce its allowlist and must not collect wallet credentials, balances, transaction contents, IP addresses for analytics, or other data outside the documented event catalog.
- Logs and error responses must not expose secrets, full provider URLs containing tokens, or unnecessary wallet-linked information.

## Security invariants to protect

These hold on `main` today. A change that breaks one is a finding even when no exploit is shown yet.

- Logs and error responses never contain a provider credential: every caught provider error passes through one redaction helper, no whole error object is logged, and an RPC address is logged as its origin only.
- Each Lambda function receives only the environment variables it reads; a test lists them per function and stage.
- Jupiter builds are validated instruction by instruction against the taker's own accounts: one router call, the taker's token accounts for the requested mints, the requested amount, slippage and fee, SOL wrap and unwrap only, a positive slippage share of 0, and Salmon's fee account at its observed position. A simulated balance check refuses a build that moves the taker's funds beyond the swap.
- `uiAmount` is converted to base units exactly and bounded in length.
- The Lambda role carries a permissions boundary; the GitHub deploy role can create or change only project roles that carry it, can neither remove nor edit it, and writes only to the project's stacks (`aws-deploy-policy.json`, `aws-lambda-permissions-boundary.json`, pinned by tests).
- Nightly live-provider tests receive provider keys only on `main`, through a protected environment.
- Analytics refuses requests from web-page origins, requires an install id, never reads the client IP, and stores and forwards events that follow an on-chain transaction with their UTC day only.
- The transaction-history fallback serves bounded pages within the provider budget and answers 503 instead of falling back while the provider is rate limited or down.

## Untrusted input and trust boundaries

Treat all public HTTP input as untrusted, including path parameters, network identifiers, wallet addresses, query strings, headers, origins, JSON bodies, analytics events, transaction parameters, token identifiers, pagination values, and requested include fields.

Also treat as untrusted:

- Responses from Solana RPC and DAS providers, Jupiter, 0x, CoinGecko, Esplora services, token lists, decentralized-application metadata sources, sanctions sources, and any other upstream service.
- Data read from Redis, because keys can collide, expire, become stale, or contain values written by another request or job.
- Scheduled-job inputs and downloaded datasets.
- Configuration supplied through environment variables, stage files, deployment metadata, and AWS Systems Manager Parameter Store.

The critical trust boundaries are:

- Public requests entering API Gateway, Express routes, middleware, and controllers.
- Transaction parameters crossing into Powerup adapters and unsigned transaction construction.
- Upstream provider data crossing into services, parsers, resources, and cached responses.
- Application code crossing into Redis and shared provider-control state.
- Deployment configuration crossing into Lambda permissions, environment variables, logging, and scheduled jobs.

## Components that matter most

Give the highest priority to:

- Powerup transaction construction, including swaps, NFT operations, fee handling, sanctions checks, address validation, instruction validation, and serialization.
- Route mounting and network resolution under `/v1/<chain>-<environment>` and multichain routes that dispatch from a network identifier.
- Middleware for origins, permissions, availability, rate limits, sanctions, and other request gates.
- Solana provider responses, transaction parsing, enrichment, resource mapping, and any fallback that changes response semantics.
- Redis key construction, tenant and network separation, cache integrity, expiry, distributed rate limiting, circuit breakers, and fail-open behavior.
- Outbound HTTP clients, URL construction, redirects, DNS behavior, timeouts, retries, response-size limits, and protection against server-side request forgery.
- Analytics schema validation and sinks, especially any route by which wallet-linked data or a client IP could reach analytics storage.
- Error handling, structured logs, provider-token redaction, and API responses.
- Serverless configuration, Lambda permissions, GitHub Actions deployment, dependency packaging, and secret retrieval.
- Scheduled jobs that ingest remote data or update shared state.

The HTTP API, scheduled jobs, analytics handler, deployment configuration, internal packages, and code under `src/`, `packages/`, `config/`, `scripts/`, and `docker/` are in scope when they can affect a distributed or deployed service.

## Out of scope

- Vulnerabilities in the wallet clients that do not arise from behavior in this repository.
- Vulnerabilities in third-party providers themselves, unless this service calls them unsafely, trusts their responses incorrectly, leaks credentials to them, or grants them unintended authority.
- Social engineering that does not depend on a product vulnerability.
- AWS account compromise or an attacker who already has unrestricted production administrator access, unless repository configuration unnecessarily expands the impact.
- Denial of service against a third-party provider with no material effect on Salmon users, data, costs, or service availability.
- Issues that require production secrets committed to the repository; committing a real secret is itself in scope.
- The accepted dependency findings documented in `SECURITY.md`, unless the report demonstrates a reachable exploit path that contradicts the stated constraints.

### Accepted limits

These are deliberate decisions with their reasons recorded by the team. Report one only with a path that breaks the reasoning, not the limit itself.

- The AWS WAF per-IP rate rule on production runs in count mode until checks confirm it sees client addresses; staging has no WAF.
- The API is also reachable at its API Gateway address besides the CloudFront distribution, until the wallets switch address.
- Staging shares the production Redis database and provider keys and is used only for partner tests.
- While installed wallets accept only Jupiter's `route_v2` form, the backend asks Jupiter again (at most 3 calls) when it answers with the shared-accounts form.
- Per-IP limits fail open when Redis is unreachable, because availability is preferred over enforcement; swap screening is the exception and fails closed.

## How to exercise the project

Install dependencies and package the service with local configuration:

```sh
npm ci
cp .env.example .env
npx serverless package --stage local
```

Useful checks and test entry points include:

```sh
npm run lint:check
npm run format:check
npm run test:unit
npm run test:integration:hermetic
npm run test:coverage
```

The hermetic integration test may require the repository's documented local Redis setup. Tests that call live providers or a local Solana validator should not be treated as build prerequisites.

Use only dummy credentials, generated test wallet addresses, local Redis data, and mocked provider responses in reproducers. Do not use production API keys, production Redis, funded accounts, real user addresses, recovery phrases, or private keys.

## Severity guidance

### Critical

- A remotely exploitable path that causes the backend to produce an attacker-controlled unsigned transaction that appears to represent a different safe operation and can credibly lead to asset loss when signed.
- Remote extraction of production secrets that permit material control of the service, providers, deployment, or shared data.
- Remote code execution in a production Lambda or deployment path with a realistic route to transaction manipulation, secret theft, or broad user impact.
- A deployment or supply-chain flaw that can inject attacker-controlled code into the production API at scale.

The backend does not sign transactions or hold user keys. Do not claim direct theft of funds unless the report demonstrates how the backend output defeats the client's review and signing boundary.

### High

- Account, token, network, instruction, recipient, fee, or program substitution in a security-sensitive unsigned transaction response.
- Server-side request forgery that reaches sensitive cloud metadata, internal services, authenticated endpoints, or provider URLs containing credentials.
- Authentication or authorization bypass on an administrative or privileged function.
- Cross-user, cross-network, or cross-stage cache confusion that can materially alter transaction construction or expose sensitive data.
- Injection or unsafe deserialization with a credible route to code execution, secret disclosure, or persistent integrity loss.
- Bypass of sanctions, availability, or transaction-building controls where the bypass creates material legal, financial, or security exposure.

### Medium

- Disclosure of wallet-linked, analytics, operational, or provider information that does not include wallet credentials or production secrets.
- Rate-limit or resource-consumption flaws with a practical path to meaningful outage, provider-cost amplification, or exhaustion of shared quotas.
- Cache poisoning or stale-data behavior that materially misleads users but does not construct or authorize a harmful transaction.
- Origin, validation, logging, or error-handling weaknesses that need substantial additional conditions to affect secrets, transactions, or availability.

### Low

- Limited information exposure, defense-in-depth gaps, or crashes with no credible path to transaction manipulation, production-secret disclosure, persistent compromise, significant privacy harm, or meaningful denial of service.

Do not rate a theoretical issue as critical or high without a concrete attacker-controlled input, reachable code path, and demonstrated impact. State whether an endpoint is public, which stage and network are affected, whether Redis or an upstream service is required, and whether the behavior is fail-open or fail-closed.

## Report and patch expectations

Each report should name the endpoint or job, provide a minimal request or test using dummy data, identify the failed trust boundary, distinguish backend impact from client-side assumptions, and list every required environmental condition.

Proposed patches should validate inputs at the appropriate boundary, preserve network separation, keep provider calls bounded, avoid broadening Lambda permissions, keep secrets out of logs and responses, and include a regression test where practical. Never fix a problem by sending wallet credentials, recovery phrases, private keys, or signed transactions to this service.

Useful proofs of concept: a Jest test with a mocked upstream body (for example a Jupiter or 0x build with an extra instruction) that the build service returns as valid, or a request against a local stack (`npm run serverless:start:local`) with dummy data. Never use production endpoints or real funds.

Deduplicate by root cause: one report per broken check, listing every route or job that reaches it.
