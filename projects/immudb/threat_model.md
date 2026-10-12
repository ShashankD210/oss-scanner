# Threat model

## What this project does and where untrusted input enters
immudb is a tamper-evident database (key-value, SQL and document models) written in Go. Every write is committed to an append-only log backed by Merkle trees, so clients can verify inclusion and consistency proofs and detect any change to history. It is deployed as a network server, frequently as the audit or compliance store behind other systems, and is also embedded as a Go library (`embedded/`).

Untrusted input (treat as attacker-controlled):
- gRPC API on port 3322 (`pkg/server`, `pkg/api`), before and after authentication.
- PostgreSQL wire protocol on port 5432 (`pkg/pgsql`), enabled by default, including the startup/auth handshake and message parsing.
- HTTP REST gateway and web console on port 8080 (`pkg/server/webserver.go`, `pkg/server` gateway handlers).
- Metrics/health endpoint on port 9497.
- SQL text sent by clients, parsed and executed by `embedded/sql`; document queries (`embedded/document`).
- Replication streams: a replica consumes transactions from a primary, and a primary serves them to replicas (`pkg/replication`).
- Proofs, states and signatures returned by a server to a verifying client (`pkg/client`, `embedded/store` verification). The client must not trust the server: a malicious or compromised server is in scope from the client's point of view.
- Responses from S3-compatible remote storage (`embedded/remotestorage`, `embedded/appendable/remoteapp`).

Trusted input: CLI flags, config files, environment variables, and local filesystem access to the data directory. These come from the operator.

## Components that matter most / least
Most:
- Authentication, sessions, tokens and per-database permissions (`pkg/auth`, `pkg/server` session and user management). A user reaching a database or operation they were not granted is serious.
- Tamper evidence: `embedded/htree`, `embedded/ahtree`, transaction hashing and proof generation in `embedded/store`, and proof verification in `pkg/client` and `embedded/store`. Any way to make a verifying client accept a forged entry, a rewritten history, or an inconsistent state breaks the core promise of the product.
- SQL engine and parser (`embedded/sql`) and the pgsql wire layer (`pkg/pgsql`): parser crashes, injection that crosses permission boundaries, unbounded allocations driven by query input.
- Replication (`pkg/replication`): a malicious primary or replica corrupting the other side, or bypassing authentication.
- Storage engine (`embedded/appendable`, `embedded/tbtree`, `embedded/store`): corruption reachable from the network API.

Least / out of scope:
- `tools/`, `test/`, `build/`, `cmd/immutest`, example code.
- Generated code (`*.pb.go`, `*.pb.gw.go`, swagger files) unless the bug is in how immudb uses it.
- Web console frontend assets (built and shipped separately).
- FIPS build variants (`*/fips`) beyond what they share with the default build.

## How to exercise it
- Binaries are in `/usr/local/bin`: `immudb`, `immuadmin`, `immuclient`.
- Start a server offline: `immudb --dir /tmp/immudb-data` (pgsql server is on by default) (default admin credentials `immudb` / `immudb`).
- `pkg/integration` and `pkg/server` tests start in-process servers and are the easiest harness for end-to-end proofs of concept.
- Go unit tests per package: `go test ./embedded/sql/...`, `go test ./pkg/pgsql/...`, etc.

## How you rate severity
- **critical**: unauthenticated remote code execution; authentication bypass; a verifying client accepting a forged or inconsistent proof, or any undetectable modification of committed data (this defeats tamper evidence).
- **high**: authenticated remote code execution; privilege escalation or access to databases or operations a user was not granted; path traversal or file read/write outside the data directory reachable from the network; credential or token disclosure.
- **medium**: unauthenticated denial of service (crash, panic, unbounded memory or disk use); detectable data corruption reachable from the network.
- **low**: denial of service requiring authentication; information leaks limited to local logs.

Reports should include a reproducer (Go test, client program or wire capture) and the commit tested. Patches should be minimal and include a regression test.

## Anything to leave alone
- The default `immudb` / `immudb` admin password. It is documented and operators are told to change it.
- Plain-text transport in the default configuration. TLS is opt-in by design.
- Issues that require write access to the data directory or the server's configuration.
