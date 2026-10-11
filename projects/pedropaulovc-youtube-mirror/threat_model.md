# Threat model

## Scope and trust boundaries

This small Cloudflare Workers/Workflows backend mirrors YouTube videos, community posts and comments to Bluesky. There is no frontend/static application in this checkout. The npm package is private; this enrollment makes no claim about downstream users or adoption.

Six Worker entrypoints are in scope: channel polling, item publication, deletion checks, profile synchronization, public OIDC discovery/JWKS, and a bearer-gated telemetry gateway. Review `worker/`, `scripts/` and the deployment/configuration code. Third-party npm libraries are dependencies, not original project implementations.

Treat YouTube API fields, Firecrawl HTML/JSON, community posts, comments, media URLs, PDS/DID responses, AT Protocol responses and incoming telemetry bodies as adversarial. Trace transformations through parsing, text splitting, facets/mentions, media fetching/uploading, KV persistence and post/thread construction. Image URLs and redirect targets cross an outbound-fetch boundary; size checks after buffering do not alone bound resource use. Telemetry parsing/transcoding and bearer checks are independently important public request boundaries.

Deployment environment, channel allowlist/configuration, KV write access, account/secret-binding selections, configured federation providers and destination origins are trusted administrative inputs. They must not accidentally cross production/PPE boundaries. The service holds privileged material in deployment: OIDC signing keys, Firecrawl tokens, per-account AT Protocol passwords, cached sessions and GCP/Azure tokens. Review secret disclosure, token audience/issuer/expiry, session refresh, account targeting and cache isolation. Public issuer endpoints should expose only public metadata. Mirroring Workers' fetch handlers return 404; their workflows are driven by trusted schedules/bindings, not an assumed public posting API.

High-priority findings include unauthorized posting/deletion or account crossover, credential disclosure, improper federation/telemetry authorization, and attacker-controlled fetches that meaningfully cross a security boundary. Content fidelity, duplicate handling and workflow/KV state are also relevant when they cause impersonation, misattribution or destructive actions. Distinguish expected reproduction of source text and links from unauthorized mentions/actions. Do not invent guarantees about upstream content authenticity or moderation.

## Build and offline local tests

The checkout lives at `/src`. The Dockerfile installs the lockfile's npm dependencies, including Wrangler, native workerd/esbuild, the Workers Vitest pool and tsx. Matching Chromium and its Linux libraries are fetched for source-level/local testing of browser helpers, not account creation. Husky hook setup and Wrangler usage telemetry are disabled; no credentials are supplied.

It compiles every checked-in `wrangler.mirror-*.jsonc` config with Wrangler `deploy --dry-run`, equivalent to the repository's resource-neutral build loop but retaining separate outputs in `dist/mirror-channel`, `dist/mirror-item`, `dist/mirror-delete`, `dist/mirror-profile`, `dist/mirror-oidc-issuer` and `dist/mirror-telemetry-gateway`. These baseline configs intentionally omit deployment-specific resources and schedules. Dry-run never deploys; it does not prove live bindings or federation setup. Node-side TypeScript tools are retained with their tsx runtime; they are not deployed Workers.

No tests run in the Dockerfile. Run the following inside the built image with networking disabled and writable local storage:

```sh
npm run typecheck
npm run test -- test/unit
npm run test -- test/integration
# Or execute both groups in one invocation:
npm test
```

`vitest.config.ts` uses `@cloudflare/vitest-pool-workers` with `wrangler.test.jsonc`, local KV/Workflow bindings and local secret-service stubs returning `test-api-token`. Unit tests substitute synthetic provider responses with fetch spies or module mocks; the AT Protocol agent is mocked. The integration suites introspect local workflow instances and mock workflow step results, testing orchestration/KV rather than real provider side effects. `test/helpers/factories.ts` supplies synthetic channel/content records. OIDC, GCP exchange, PDS resolution, content, API and configuration code have local tests. Dependencies are already present: do not download tools at runtime. These commands are source-derived and have not yet been executed for enrollment.

## Capability and safety limits

This Linux container does not reproduce Cloudflare's deployed scheduler, eventual consistency, real Secrets Store, cross-account IAM, GCP/Azure federation, provider quotas or live Bluesky moderation/account behavior. The local workerd runtime is useful evidence for application logic, not proof of those cloud properties. Windows/macOS behavior is not exercised. Chromium is available, but no real user profile or account is provisioned.

Do not run `deploy:*`, `verify:*`, `config:*`, `cf-typegen:*`, provisioning/deprovisioning/seed scripts, environment-secret synchronization, observability configuration or account confirmation against real services. Do not start an unmocked workflow against provider endpoints. Authenticated 1Password (`op`) and GitHub (`gh`) administrative tooling are intentionally not provisioned: their operations remain source-review-only, not fake offline replacements. Do not copy `.env`, credentials, provisioning state or deployed account data into the image. Testing must use synthetic credentials/data and container network isolation. Do not fetch live images, scrape channels, publish/delete posts, create accounts or mutate cloud resources.
