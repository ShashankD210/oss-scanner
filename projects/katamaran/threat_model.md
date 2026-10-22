# Katamaran security audit

Katamaran orchestrates live migration of Kata Containers across Kubernetes nodes: disk mirroring, QEMU state transfer, network cutover, controllers, dashboard, and VM adoption. It is independently maintained with small public adoption; do not assume widespread production use. Review `SECURITY.md`, `docs/THREAT_MODEL.md`, `docs/TESTING.md`, and the actual deployment manifests against the scanned commit.

## Scope and priorities

Audit the project's own code and manifests. Prioritize:

- Dashboard request validation, origin checks, namespace/node/pod selection, target screening, cancellation, and transitions from HTTP input into Kubernetes Jobs (`internal/dashboard/`, `internal/orchestrator/`).
- Migration CRD admission/reconciliation, RBAC assumptions, namespace boundaries, cleanup/recovery, and the privileged Job templates (`cmd/katamaran-mgr/`, `internal/controller/`, `internal/orchestrator/templates/`, `deploy/`, `config/`).
- QMP response/frame parsing, socket lifecycle, concurrent use, timeouts, unexpected events, and migration-state transitions (`internal/qmp/`, `internal/migration/`).
- Source pod-log/cmdline replay, QEMU arguments, paths, environment, device handling, and subprocess execution. Treat captured metadata as adversarial at its parser; demonstrate which actor can supply it and which destination authority it reaches.
- Factory gRPC/socket ownership, sandbox metadata/watchers, experimental adopted shim listener/FD handling, filesystem paths, and cleanup (`internal/factory/`, `cmd/katamaran-factory/`, `cmd/containerd-shim-katamaran-adopted-v2/`).

State the attacker's actual Kubernetes rights, network position, Unix-socket access, namespace, and ability to alter a source workload. Distinguish cluster-admin capabilities from a lower-privilege actor reaching a privileged host Job. Existing threat-model findings such as no dashboard authentication, unencrypted migration traffic, powerful CRD/controller permissions, and trusted node sockets are known limitations: verify them, but report new boundary bypasses or materially new exploit chains rather than restating their existence. Do not assume a port-forward is cryptographic authentication or that namespaced CR creation implies permission to control a host.

## Offline reproduction

The checkout is `/src`. The image provides the Go version required by `go.mod`, cached modules, CGO toolchain, the pinned Node version, Biome, ShellCheck, yamllint, and all built binaries. `make check` runs during image creation. Network access is unnecessary for the local gate:

```sh
make check
go test ./internal/qmp/ -race -count=1 -timeout 120s
go test ./internal/dashboard/ -race -count=1 -timeout 120s
make fuzz
```

The image bounds Go build parallelism to two workers for the scanner's CPU/memory budget. Unit tests use fake Kubernetes clients, temporary Unix sockets, simulated QMP peers, and synthetic guest metadata; smoke tests require no cluster or root privileges. Go's ordinary short `/tmp` paths are intentional for Unix-socket length limits. Use temporary synthetic data only. Never mount host VM sockets, guest disks, kubeconfigs, service-account tokens, or container-engine sockets into the audit environment.

Real migrations, KVM/QEMU/Kata, kernel tunnel/qdisc setup, cluster E2E, container-image builds, and multi-architecture runtime tests are not part of the offline gate. Do not invoke `scripts/e2e.sh`, apply manifests, change a host's networking, create privileged workloads, or interact with any production cluster. Test subprocess paths through stubs and mocks; clearly label integrations that were not exercised. Dependency-only vulnerabilities belong upstream.

## Findings

Provide commit, files/symbols, exact attacker rights, reproducible input, expected versus actual behavior, and demonstrated impact on host authority, guest confidentiality/integrity, or cluster resources. Prefer deterministic regression tests and minimal fixes preserving migration behavior and recovery. Proposed patches and commit messages should contain no attribution or co-author trailers.
