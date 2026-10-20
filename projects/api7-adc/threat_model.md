# ADC security scanning guidance

## Scope and purpose

Scan the code maintained in `api7/adc`, including the TypeScript implementation
under `apps/cli/` and `libs/` and the Rust implementation under `rust/crates/`.
ADC converts, validates, compares, exports, and synchronizes gateway configuration
through Apache APISIX and API7 Enterprise Admin APIs. It also provides an HTTP,
HTTPS, or Unix-socket server used by automation to validate and synchronize it.
Its security importance comes from handling gateway management credentials and
changing routing, authentication, consumers, and TLS configuration in deployments.

Third-party gateways, Kubernetes controllers, and dependencies are not separate
scan targets. Demonstrate the root cause in ADC's code when reporting an issue.

## Inputs, authority, and trust boundaries

- YAML/JSON configuration and OpenAPI descriptions may come from repositories or
  other teams. Consider their contents adversarial when they are parsed, converted,
  compared, or validated. Distinguish processing a document from an operator's
  explicit decision to apply the gateway behavior described by that document.
- CLI flags, environment variables, backend endpoint selection, local certificate
  paths, and explicit output filenames are operator configuration. Describe how
  a less-trusted input can influence them before claiming a boundary crossing.
- Admin API keys, API7 tokens, TLS private keys, and backend credentials are
  secrets. Assess their handling in HTTP clients, redirects, error messages,
  logging, configuration exports, and client/connection caches. Configuration
  exports intentionally contain the gateway configuration requested by the user;
  distinguish expected export contents from unrelated secret disclosure.
- Backend HTTP responses are external input. Check response parsing and data
  reconciliation without assuming that a compromised backend is entitled to
  access the ADC process's unrelated local files or credentials.
- The server's `/sync` and `/validate` requests contain backend options and
  configuration. Examine schema validation, TLS/mTLS enforcement when enabled,
  and whether a request can reuse another request's credentials, TLS material,
  cached state, or backend connection. Establish the actual caller authentication
  and deployment configuration; a `cacheKey` alone is not proof of a tenant
  isolation promise.
- The server defaults to a loopback endpoint and supports Unix sockets and TLS
  client certificates. An operator-selected backend URL is an intended feature.
  For an SSRF claim, show how an actor crosses an existing access restriction or
  obtains authority beyond the request's permitted backend options. Distinguish
  an intentionally exposed service from a bypass of a configured protection.
- Sync can create, update, and delete gateway resources. Check label selectors,
  include/exclude resource filters, gateway group selection, and error recovery
  for changes outside the selected scope. Applying a deliberately permissive
  configuration with valid administrator authority is not an authorization bypass.

## Areas to prioritize

Prioritize unsafe file or command operations reached through document contents,
credential leakage to a different endpoint or request, TLS verification bypasses,
server request isolation, parser resource exhaustion with a bounded reproducer,
and incorrect reconciliation that crosses an explicitly selected resource scope.
Compare the TypeScript and Rust implementations where a difference can reveal a
security regression; a behavioral difference alone does not establish a flaw.

## Offline build and reproduction

The Dockerfile retains the full checkout at `/src`, installed pnpm dependencies,
the Node.js and Rust toolchains, Cargo's dependency cache, built TypeScript CLI
at `/src/dist/apps/cli/main.cjs`, and Rust debug binaries under
`/src/rust/target/debug/`. All setup downloads happen during the online build.

From the offline image, run:

```sh
timeout 900 oss-scanner-test
node /src/dist/apps/cli/main.cjs --help
/src/rust/target/debug/adc --help
```

`oss-scanner-test` runs the six TypeScript unit suites exercised by upstream CI
and `cargo test --workspace --locked --offline` for the Rust workspace. Rust
benchmark fixtures are generated during the build. Live APISIX/API7 E2E suites
are not provisioned; some require a commercial backend and license. Use local
HTTP/TLS fixtures, mock servers, and existing unit-test helpers for reproducible
backend interactions. Explicitly mark findings requiring live-backend validation.
Run Cargo commands with `--locked --offline` during the audit.

## Reports and severity

Provide the affected implementation and commit, required attacker access,
the precise trust boundary, a minimal local reproducer with synthetic credentials,
observed output, impact, and a focused regression test and patch when possible.
Group equivalent TypeScript/Rust instances by root cause and identify both paths.
Separate verified behavior from hypotheses and state any missing prerequisites.

Prioritize demonstrated unauthorized code execution, management-secret disclosure,
configured authentication bypasses, cross-request credential reuse, or changes
outside an authorized gateway scope. Rate availability issues according to their
reachability, input size, and recovery cost. Maintainers determine final severity;
missing hardening or a dependency version match alone is not a verified exploit.
