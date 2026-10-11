# Threat model

## Purpose and untrusted input

CodjiFlo is a Next.js/React GitHub pull-request review application with browser-side GitHub API access, OAuth token proxy routes, diff/artifact handling, and a separately packaged GitHub Action. GitHub repository content, diffs, comments, downloaded iteration artifacts, URLs and route parameters, OAuth callback fields, browser-persisted state, and provider responses cross trust boundaries. Do not assume repository content or externally returned JSON is safe merely because GitHub transports it.

## Authentication, PKCE, and refresh boundaries

Prioritize `src/features/auth/`, `src/app/api/auth/`, `src/app/auth/callback/page.tsx`, and `src/app/auth/landing/page.tsx`. The browser creates PKCE verifier/challenge and OAuth state, then the callback verifies state and exchanges the authorization code through a server API route. Inspect verifier entropy and lifetime, S256 handling, state binding, callback errors, replay, return-origin/path validation, cross-origin token handoff, and cleanup on every exit path. Public client ID and public app URL are not secrets.

The auth store persists user access and refresh tokens in browser localStorage under `auth-storage`. JavaScript-readable cookies carry PKCE/state/return routing and, for cross-origin handoff, a base64-encoded JSON token payload; base64 is not encryption. Source cookie defaults include short lifetimes, SameSite=Lax, Secure on HTTPS, and a parent-domain cookie for recognized codjiflo.net hosts. The return-origin policy permits localhost/127.0.0.1 and codjiflo.net/subdomains; consider whether an attacker controls an allowed destination before asserting a cross-origin exploit. These are not HttpOnly server sessions. Evaluate XSS, compromised same-site/subdomain access, token leakage, malformed token transfers, and token refresh/retry races against this actual design.

The server helper in `src/app/api/auth/utils.ts` obtains the OAuth client secret from runtime environment or a Cloudflare Secret Store binding. Token and refresh handlers send it only to GitHub's OAuth endpoint. Runtime `GITHUB_APP_CLIENT_ID` is distinct from build-time `NEXT_PUBLIC_GITHUB_CLIENT_ID`. Browser GitHub API requests use user bearer tokens directly. OAuth scopes include repo, read:user, and actions:read; compromise of a user's token can affect more than public repository reads. The development-token route invokes local `gh auth token` only in development; the production build is intended to return 404. Audit that gate without accessing a real CLI credential store.

Refresh logic, persisted refresh credentials, error handling, and retry-after-401 behavior matter particularly. Tests should use synthetic responses, not exchange real codes or refresh real tokens. Demonstrate a credential/session boundary crossing rather than reporting the mere existence of client-side credentials as a new vulnerability.

## Other components and severity

Diff rendering, markdown/comments, artifact/archive parsing, SQL.js state, and API request construction remain in scope. `packages/diff-engine` is a root npm workspace. `action/` has its own lockfile and bundle build; it uses a workflow GitHub token, artifact data, iteration storage, and PR description updates. That action token is separate from the web app's user token/client secret. Audit action inputs and artifact contents locally; do not invoke the action against a real repository.

Unauthorized account access, access/refresh-token or client-secret disclosure, and exploitable stored/script injection that captures credentials generally warrant high severity. Remote execution with meaningful privileges or broad cross-account compromise can be critical if supported by an exploit. Cross-origin redirects, denial of service, and routing errors should be rated by demonstrated attacker control and impact; avoid assuming every allowed subdomain is attacker-owned. Missing hardening without an exploit should be distinguished from a proven vulnerability.

## Build and offline exercises

The image retains the full checkout at `/src`, installs root/workspace dependencies with `npm ci` (including SQL.js postinstall assets), installs Chromium headless shell and OS dependencies online, installs/builds the independent Action with `npm --prefix action ci` and `npm --prefix action run build`, then compiles the application with `npm run build:next`. Node 24.15.0 follows the CI pin; workflows note a Playwright installation hang with 24.16.0. `npm run build` is the Cloudflare OpenNext wrapper, so it is deliberately not used for the standalone Node build. No OAuth secret is supplied. Compilation does not exercise provider authentication.

Run these commands inside the built image without Internet networking:

```sh
cd /src
npm run test -- --run src/features/auth/utils/pkce.test.ts src/features/auth/utils/cookies.test.ts src/features/auth/config.test.ts src/features/auth/hooks/useOAuthFlow.test.ts src/features/auth/stores/useAuthStore.test.ts src/app/api/auth/utils.test.ts
scanner-auth-e2e
```

The unit selection exercises PKCE, cookie/return validation, authorization URL construction, redirect initiation, token persistence/refresh, and mocked token-proxy requests. `scanner-auth-e2e` starts the already-built local Next server and runs `e2e/common/stateless-mode/authentication-token-refresh.spec.ts` via the existing `test:e2e` script. That spec intercepts refresh and relevant GitHub requests. Its loopback `E2E_BASE_URL` skips Playwright's configured server command, which would otherwise rebuild through OpenNext. Browser binaries are already installed. This is a local mock-backed refresh smoke, not a full real-provider OAuth flow. Other offline unit tests remain available through `npm run test -- --run`; the full mock-mode browser suite may be run against a manually started local server with the same loopback URL.

## Leave alone

Do not run `test:e2e:prod`, use a GitHub token, invoke `gh auth token`, contact production/preview deployments, or execute Cloudflare preview/deploy/secret operations. Prod-mode fixtures can access real GitHub repositories and credentials; exclude the whole mode, not merely files whose names suggest live tests. Keep scanner networking disabled after dependency/build setup, with loopback only for local mocks. Do not report production configuration, third-party host behavior, or live-provider outcomes as exercised by this image. No credentials or production data are needed for the supplied smoke commands.
