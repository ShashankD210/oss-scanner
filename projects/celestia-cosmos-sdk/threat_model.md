# Celestia Cosmos SDK threat model

This is Celestia's independent Cosmos SDK repository, derived from Cosmos SDK v0.50 and maintained on `release/v0.52.x-celestia`. Celestia's validators use it through celestia-app's pinned Go module replacements. It implements transaction authorization, account and token accounting, application state storage, and the ABCI framework through which consensus executes blocks. Protect deterministic state transitions and app hashes, authorization, conservation of funds, valid state proofs, and validator availability.

The repository became standalone in October 2026 while retaining its Git history and release tags. Historical PRs are in `celestiaorg/cosmos-sdk-archive`; transferred issues and the number maps are in `docs/repository-migration`. The scan target is the active Celestia branch, not `cosmos/cosmos-sdk` upstream or the archived repository.

## Project guidance and scope

Read the scanned checkout's `README.md`, `CLAUDE.md`, `CHANGELOG.md`, module READMEs, and `docs/audit`. Prefer source and checked-in module versions over prose that may describe an older release. The [Celestia organization security policy](https://github.com/celestiaorg/.github/blob/main/SECURITY.md) governs reporting.

The whole repository is in scope for source review, including every nested Go module. Prioritize Celestia-specific changes and their interaction with inherited Cosmos SDK code. Use the README and Git history to identify those differences. This includes phased transaction execution, concurrent CheckTx and queries, lazy staking and slashing migrations, delayed distribution reward claims, historical account-number queries, signer extraction, and application-version handling.

celestia-app, celestia-core, IAVL, and other replaced dependencies are separate projects. Report a dependency issue when this SDK reaches it, identify its owner and pinned version, and show the path from the SDK. celestia-app supplies its own proposal handlers and module wiring: do not assume every simapp feature is enabled in Celestia. Compare a finding with the SDK revision, app version, configuration, and modules actually used by the affected application. Features on the scan branch may precede a network upgrade.

## Actors and entry points

Treat transaction bytes, account public keys and signatures, message fields, transaction metadata, query parameters, snapshot chunks, and peer-supplied data as untrusted. Any account may submit transactions, including malformed ones. A proposer may choose and order block contents. A single validator may be malicious; consensus safety assumes more than two thirds of voting power is honest. Distinguish an ordinary sender from a proposer, governance authority, local operator, or attacker controlling substantial voting power.

Trace transaction decoding through `codec/`, `types/tx`, and `x/auth/tx`, then the ante decorators in `x/auth/ante` and message dispatch in `baseapp`. Check sign modes, signer extraction, account numbers and sequences, multisignature limits, fee deductions, gas charging, and authorization in `x/authz` and `x/feegrant`. A pluggable signature verifier is trusted application configuration, but its concurrency contract and equivalence to ordinary verification matter. Simulation, recheck, and cached verification results must not authorize a transaction that should fail during block execution.

Review `baseapp/abci.go`, `baseapp/runtx_phases.go`, optimistic execution under `baseapp/oe`, cache branching, rollback, and commit handling. Check that failures cannot leak partial message writes, that ante and message effects follow their intended commit rules, and that concurrent queries or CheckTx cannot mutate consensus state or race with WorkingHash and Commit. Proposals, vote extensions, and begin/end blockers must produce deterministic results across honest nodes. App and consensus parameters must retain the correct historical values through upgrades.

Review `x/bank`, `x/auth/vesting`, `x/distribution`, `x/staking`, `x/slashing`, `x/gov`, `x/group`, and the nested modules under `x/`. Protect balances, supply, staking shares, reward entitlements, permission boundaries, and replay protection. Lazy migrations must agree with both old and new key layouts, preserve balances and validator history, and be deterministic regardless of the first access path. Arithmetic in `math/`, pagination, malformed addresses, and unusual denominations are useful boundary inputs. A governance-authorized action is not an authorization bypass by itself.

Review `store/` for cache isolation, committed version tracking, pruning, IAVL access, snapshot restore, proof construction, and resource limits. An accepted snapshot or proof must agree with the trusted commitment. Query paths through `server/grpc`, `server/api`, and BaseApp's query router must remain read-only and enforce their configured bounds. The Celestia BlockAPI integration and remote ABCI transport are relevant, but a locally exposed privileged service is not an unauthenticated public interface unless the affected deployment makes it one.

Default test keyrings and development genesis files are documented operator choices. Demonstrate a concrete unauthorized action before treating them as security findings. Generated code, tooling, examples, and test helpers remain reviewable, but prioritize a path that affects a deployed application or a maintainer's build process.

## Build and bounded validation

The checkout is `/src`, and `/src/build/simd` is the reference application binary. The Dockerfile downloads dependencies for every checked-in Go module, builds simd, and compiles test packages with ledger mock support, including the E2E tag and legacy `app_v1` simapp wiring. It runs bounded crypto, SDK type, and math tests during setup. All module downloads happen before the network is removed; `GOPROXY=off` and `GOFLAGS=-p=2 -mod=readonly` remain set. Do not edit `go.mod` or `go.sum` to make a reproducer build.

The scanner has two CPUs and 8 GB of memory. Run targeted tests with a timeout and at most two package/test workers. Nested modules require changing into their directory; root `go test ./...` does not cover them. Examples:

```sh
go test -p 2 -parallel 2 -short -timeout 10m -tags 'ledger test_ledger_mock' ./baseapp/... ./x/auth/ante/...
(cd /src/store && go test -p 2 -parallel 2 -short -timeout 10m ./cachekv/... ./rootmulti/...)
(cd /src/tests && go test -p 2 -parallel 2 -timeout 15m -run TestName ./integration/...)
(cd /src/simapp && go test -p 2 -parallel 2 -timeout 10m -tags 'app_v1 ledger test_ledger_mock' -run TestName ./...)
```

Use `testutil` and `simapp` helpers for in-memory stores and local networks. E2E packages under `tests/e2e` use the `e2e` build tag and can start local processes or networks, so select individual tests. Simulation suites are opt-in and may run for many hours with their default Makefile targets. Bound fuzzing with `-fuzztime` and an external timeout. The race detector can rebuild dependencies offline, but consumes additional time, memory, and cache space; apply it to the package under investigation.

Reproduce only in temporary directories, in-memory stores, or local test networks. Never connect to Mainnet Beta, Mocha, Arabica, or another public network, and never use real keys or funds.

## Evidence and severity

For each report, record the exact SDK revision, dependency versions, build tags, application configuration, attacker capability, and whether the affected path is deployed. Separate a source hypothesis from an executed reproducer. For consensus failures, demonstrate divergent state or app hashes between honest nodes, or a failure that escapes transaction-level recovery and halts execution. For denial of service, measure attacker cost against gas, configured limits, CPU, memory, and the effect on consensus progress. An unreproduced data race or a recovered transaction panic does not alone establish a chain halt.

Rate severity by demonstrated impact:

- Critical: unauthorized fund creation or theft, transaction authorization bypass, or a reproducible consensus split or chain halt reachable by an ordinary sender or less than one third of voting power.
- High: sustained remotely reachable denial of service against validators in supported configuration, invalid committed state proofs, or permanent loss of funds through normal authorized use.
- Medium: bounded availability degradation, limited accounting/proof failures, or attacks needing unusual configuration or substantial validator privileges.
- Low: defense-in-depth issues and failures requiring local access or stronger trust assumptions without a concrete network impact.

Keep confidence separate from severity. Check existing tests and Git history for the same cause. Fixes must preserve historical replay and consensus compatibility; identify the required application upgrade gate rather than silently changing execution at old heights. Keep patches minimal and supply a regression test that fails before the fix.

Report privately to `security@celestia.org` or through the active repository's GitHub security advisory flow, following the organization security policy. Include the smallest bounded reproducer, expected and actual behavior, and evidence limits. Do not publish suspected vulnerabilities or reproducer details in public issues or PRs; disclosure decisions belong to the Celestia security team.
