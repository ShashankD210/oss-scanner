# Silencer security audit

Silencer is an AGPL-3.0-only privacy gateway for AI clients. The supported public
runtime combines a Go HTTP gateway, a native C++17/PCRE2 masking engine and a React
management console. It masks sensitive values before forwarding requests and
restores them in JSON, SSE and NDJSON replies, including tool arguments. See
`docs/ARCHITECTURE.md` for the architecture and documented limits.

## Scope and trust boundaries

Audit this repository's public source, especially:

- `gateway/internal/gateway/`: authentication and sessions, control APIs, named
  upstream routing, header/credential handling, proxying, decompression, streaming
  protocol parsing, restoration, tool-argument enforcement and log access.
- `engine/cpp/`: adversarial text, Unicode, JSON scalars, regexes, placeholder
  parsing, buffer/session lifetime, incremental masking/restoration and the C ABI.
- `gateway/internal/storage/`: SQL access, durable original/placeholder mappings,
  scope isolation, migrations and retention. Review concurrent access and cleanup.
- The host-agent and capture code under `gateway/`: local privilege boundaries,
  file allowlists, path/symlink handling, IPC authentication, configuration backups,
  capture lifecycle, processing-target validation and certificate handling.
- `frontend/`: management-console XSS, unsafe rendering of request/log/upstream
  content, authentication handling and interaction with privileged APIs.
- Public installer and packaging scripts, for project-owned security flaws.

Treat client request bodies, provider responses, tool arguments, protocol framing,
headers, paths and stored log content as potentially adversarial. Consider both
unauthenticated network attackers and clients possessing provider API keys but no
management credentials. A configured upstream may return malicious responses.
Administrators intentionally control routes, custom rules and host-agent file
allowlists; distinguish that intended authority from crossing into other scopes
or escalating host privileges.

No live deployment, website, upstream AI service, or third-party infrastructure is
authorized for testing. Use local mock HTTP servers and synthetic data only.
Browser extension source/UI is not published and is outside this enrollment;
production disables extension endpoints. Report project-owned flaws rather than
standalone issues in third-party dependencies.

## Security goals and documented behavior

- Authentication must protect control, configuration and private log APIs.
  Gateway management tokens and session cookies must not reach upstreams.
- Masking/restoration errors must fail closed, never silently forward unmasked
  input. Durable mappings must not disclose originals across host, route or
  provider-credential scopes, including under concurrency and after restart.
- Log bodies and stored mappings contain sensitive originals. The UI mask is a
  display feature, not encryption; default detailed logging and three-day log
  retention are documented. Durable mappings intentionally survive log deletion.
  Test unauthorized access, retention violations and permission/path bypasses.
- Tool guard defaults to observe-only. In rewrite/block modes, complete tool
  arguments must be checked before executable tool calls are released. It is a
  rule-based control, not a general execution sandbox or automatic approval flow.
- Public health/metrics diagnostics are intentional; assess exposure of secrets
  rather than mere endpoint reachability. Admin-chosen network destinations are
  intentional; assess whether lower-trust users can abuse that authority.

## Build and offline tests

The checkout is at `/src`. Go 1.25, Node 22, C++ compilers, PCRE2 headers, Go module
cache and npm dependencies are installed during the network-enabled image build.
Run `silencer-test` after build with networking disabled; it runs native C++ tests,
ordinary and native-engine Go tests, and frontend build/lint/log/privacy/capture
tests. It uses `-count=1` so cached Go test results do not substitute for execution.
The image retains build-test output at `/opt/silencer-test-build.log`.

Debuggable native library/test: `/opt/silencer-native/`; Go binaries:
`/opt/silencer-bin/`; frontend: `/src/frontend/dist`. Rebuild after source changes
with the commands in this Dockerfile. For fuzzing, `engine/cpp/engine_fuzz.cpp` and
`engine/cpp/corpus/` are provided; clang is installed.

MySQL integration tests skip without `SILENCER_TEST_MYSQL_DSN`; SQLite tests run.
OS capture integration requires extra mitmproxy/runtime and platform privileges
and may skip in this image; this does not establish Windows/macOS/eBPF coverage.
Use unit tests/mocks for those paths and clearly state any untested prerequisite.
Never use real credentials, private CA keys or user traffic in reproducers.

## Reporting and severity

Prioritize reproducible management-authentication bypass, cross-scope disclosure
of sensitive originals/credentials, native memory corruption, privilege escalation,
and remote resource-exhaustion attacks. Rate severity by demonstrated impact and
prerequisites; authenticated admin-only configuration is not equivalent to an
unauthenticated remote exploit. Clearly separate confirmed exploit paths from
suspected behavior and document request sizes, access level and deployment assumptions.

For each distinct root cause, include the affected revision, source locations,
minimal local reproducer using synthetic data, expected versus actual behavior,
and a regression test. Proposed fixes should preserve fail-closed masking,
streaming protocol correctness and scope isolation. Send security details privately
to the enrolled contact; do not open public vulnerability issues.
