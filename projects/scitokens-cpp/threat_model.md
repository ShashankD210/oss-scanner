# Threat model

## What this project does and where untrusted input enters
- scitokens-cpp is a C/C++ library (libSciTokens) for creating, parsing and verifying SciTokens, a JWT-based
  authorization token format used across distributed scientific computing (for example HTCondor, XRootD and
  CVMFS link against it). It is typically loaded into long-running, often privileged, storage and compute services.
- The primary untrusted input is the serialized token passed to `scitoken_deserialize`, `scitoken_deserialize_v2`,
  `scitoken_deserialize_start` and friends. Treat every byte of the token (header, payload claims, signature) as
  attacker-controlled.
- The issuer (`iss`) in a token is attacker-chosen. Unless it is rejected by the caller's allowed-issuer list, the
  library fetches `<iss>/.well-known/openid-configuration` and the referenced JWKS over HTTPS with libcurl. Those
  HTTP responses (status, headers, JSON bodies, key material) are untrusted.
- Scopes and claims are interpreted by the Enforcer API (`enforcer_create`, `enforcer_test`,
  `enforcer_generate_acls`) to make authorization decisions on paths and operations. Path handling here is
  security critical.
- The on-disk key cache (SQLite, under `$XDG_CACHE_HOME`/configured cache home) and environment/config variables
  (`SCITOKEN_CONFIG_*`) are set by the local administrator and are trusted.

## Components that matter most / least
- Most important: `src/scitokens_internal.cpp` and `src/scitokens.cpp` (token parsing, signature verification,
  issuer/audience/expiry validation, JWKS/OIDC discovery and parsing, scope and path authorization in the Enforcer),
  and `src/scitokens_cache.cpp` (key cache, which must not be poisonable by untrusted issuers).
- In scope but lower priority: `src/scitokens_monitoring.cpp`.
- Command-line tools (`src/verify.cpp`, `src/create.cpp`, `src/list_access.cpp`, `src/generate_jwks.cpp`) and the
  test programs (`src/test.cpp`, `src/test_access.cpp`, `test/`) are developer utilities; bugs there are low
  severity unless they reflect a library bug.
- `vendor/` holds third-party code (jwt-cpp, googletest) and is out of scope, except where scitokens-cpp uses it
  unsafely.

## How to exercise it
- `/src/build` is a RelWithDebInfo build; `/src/build-asan` is an AddressSanitizer build. Run `ctest` in either.
- `scitokens-verify` reads a token on stdin and verifies it (`--cred`, `--issuer`, `--keyid` allow offline
  verification with a local public key). `scitokens-generate-jwks` and `scitokens-create` produce keys and tokens.
- `test/integration-test-setup.sh` starts a local HTTPS JWKS/OIDC server (`test/jwks_server.py`) that can be used
  to exercise the key-discovery path without Internet access.
- A few unit tests (`KeycacheTest.RefreshTest`, `KeycacheTest.RefreshExpiredTest`,
  `KeycacheTest.LoadJwksTriggersRefreshWhenStale`) fetch keys from https://demo.scitokens.org and are expected to
  fail without Internet access; they are not a sign of a bug.

## How you rate severity
- Critical: accepting a token that should be rejected (signature bypass, algorithm confusion, issuer/audience/expiry
  bypass, key cache poisoning across issuers), or an authorization decision in the Enforcer that grants access to a
  path or operation the token does not authorize. Memory corruption reachable from an unauthenticated token with a
  plausible path to code execution.
- High: other memory-safety bugs (overflow, use-after-free, double free) reachable from a token or from an issuer's
  HTTP responses.
- Medium: denial of service (crash, unbounded memory or CPU, hang) triggered by an untrusted token or issuer
  response; information leaks in error messages.
- Low: issues that require control of trusted configuration, the local cache files or the environment, and bugs
  confined to the command-line tools.

## Anything to leave alone
- Do not report that the library makes outbound HTTPS requests to the issuer named in a token; this is the designed
  behaviour of OIDC discovery. Report it only if it can be abused beyond that (for example, non-HTTPS schemes,
  bypass of the allowed-issuer list, or SSRF into unexpected URLs).
- Bugs in vendored third-party code under `vendor/` should be reported upstream.
