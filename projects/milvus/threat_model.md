# Threat model: Milvus

## Project and trust boundaries

Milvus is a vector database with a Go service layer, a C++ query and indexing
engine, and Rust components used for text indexing and storage. It can run as a
standalone server or as distributed components.

Client-controlled data enters through the public gRPC and HTTP APIs, including
collection schemas, vector and scalar values, filter expressions, search
parameters, import requests and uploaded or referenced data files. A caller can
be anonymous or authenticated with limited privileges. Stored values remain
untrusted when they are read back, indexed, filtered or returned in responses.

Authentication and RBAC are security boundaries when authorization is enabled.
Database and collection names supplied by a caller do not grant access to those
objects. Compare gRPC and HTTP behavior, including batch and asynchronous paths.
Do not assume authentication is enabled in every deployment; state the exact
configuration required by a finding.

Operators control server configuration, deployment credentials and cluster
membership. Internal component RPCs, metadata stores and object storage normally
live on a trusted service network. Distinguish a public API exploit from a finding
that requires access to that network, arbitrary metadata writes or host access.
Do not assume that an API user controls an internal serialized query plan without
showing how that plan can be produced through a reachable interface.

PyUDF is opt-in and executes administrator-reviewed, trusted Python wheels. Its
worker process is not a sandbox for malicious UDF code, and its separate gRPC
endpoint belongs on loopback or a trusted network. See the trust-boundary section
of `docs/design-docs/design_docs/20260722-pyudf-function-chain.md`. A finding must
show an actual violation of these boundaries rather than code execution by an
already trusted wheel.

## Priority surfaces

- `internal/proxy` and `internal/distributed/proxy/httpserver`: authentication,
  privilege enforcement, request validation, database and collection isolation,
  credentials and error responses.
- `internal/parser/planparserv2` and `internal/core/src`: filter parsing, expression
  evaluation, vector dimensions and lengths, scalar and JSON handling, native
  buffer ownership, integer overflow and the Go/C++ boundary.
- Import and storage readers: malformed files, user-controlled paths or URLs,
  decompression, parser bounds and resource consumption. Show which actor can
  supply the offending object or reference.
- External function and model integrations: credentials, endpoint selection and
  server-side requests. Establish the required privilege and configured policy
  before describing a request to an operator-selected endpoint as SSRF.
- Native indexing and storage dependencies where Milvus exposes a vulnerable
  operation. Identify both the dependency and the reachable Milvus call path.

## Build and testing environment

The Dockerfile builds the CPU server and C++ unit tests from the checkout at
`/src`. The executable is `/src/bin/milvus`; native libraries and C++ test binaries
are under `/src/internal/core/output`. Build trees and Go, Conan and Cargo caches
are retained. GPU builds and external cloud services are not prepared.
Full setup logs are retained as `/tmp/milvus-dependencies.log`,
`/tmp/milvus-native-build.log` and `/tmp/milvus-server-build.log`.

From the offline shell:

```bash
cd /src
source scripts/setenv.sh
./bin/milvus dry-run
go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./internal/parser/planparserv2
(cd pkg && go test -tags dynamic,test -gcflags="all=-N -l" -count=1 ./util/crypto)
internal/core/output/unittest/all_tests --gtest_list_tests
# Select a relevant suite from the listing; some suites require external services.
internal/core/output/unittest/all_tests --gtest_filter='<suite>.*'
```

The focused Go suites and the C++ `StringExpr.Term` and `StringExpr.Compare` tests
are run in a network-disabled build step. The full suite is not run. Test sources in
`internal/proxy`, `internal/distributed/proxy/httpserver` and `tests/integration`
provide further fixtures; tests needing etcd, object storage, messaging or model
services require local fixtures. Those services are not started by this image.
Full distributed and SDK E2E suites are not the baseline validation.

For offline native edits, rebuild the existing CMake tree with
`cmake --build /src/cmake_build -j 8` and install with
`cmake --install /src/cmake_build`. Avoid top-level Make targets that unconditionally
fetch protobuf sources or query dependency registries. Additional tests may need
dependencies beyond the focused suites; report any missing offline fixture.

## Severity and evidence

Prioritize demonstrated impact and state the attacker's privileges and reachability:

- Critical: remotely reachable code execution or a broadly exploitable compromise
  of database confidentiality and integrity.
- High: authentication bypass, unauthorized access to another database or
  collection, credential disclosure, or attacker-reachable native memory corruption.
- Medium: a reproducible process crash, hang or disproportionate resource use from
  a bounded request, adjusted for authentication and deployment prerequisites.
- Low or informational: hardening suggestions without a demonstrated boundary
  violation. Expensive authorized workloads alone are not evidence of a vulnerability.

Include the exact Git commit, configuration, API sequence or crafted input,
expected security boundary and observed result. For memory errors, include a
native backtrace or sanitizer evidence. Prefer a minimal regression test and a
focused patch. Do not substitute a dependency-version warning or an unreachable
assertion for a reproducer. Scanner findings require maintainer review before
severity and disclosure decisions are final.
