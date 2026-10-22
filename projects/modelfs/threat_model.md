# ModelFS security audit

ModelFS provides a Linux FUSE filesystem for LLM weights, backed by an authoritative origin directory, a local piece cache, and authenticated HTTP transfers between cluster peers. It is used on the maintainer's inference cluster. Do not assume broad third-party adoption or a public multi-tenant service.

The repository's `docs/threat-model.md`, `SECURITY.md`, and `docs/architecture.md` describe the implementation and its known limitations. Recheck those descriptions against current source before treating them as security guarantees.

## Scope and priorities

Audit ModelFS's own source, tests, and shipped recovery tooling. Prioritize:

- Peer HTTP request parsing, pre-authentication behavior, bearer authentication, range arithmetic, resource bounds, and path decoding in `src/peer.zig`.
- Origin and cache path handling, symlink races, filesystem permissions, piece hashes, sidecars, and manifests in `src/store.zig`, `src/piece.zig`, and `src/fuse_fs.zig`.
- Malicious or corrupted lease JSON, peer addresses, and discovery metadata in `src/proto.zig` and `src/discover.zig`.
- The live-update and handover paths, inherited file descriptors, sealed memfd state, and secret handling in `src/handover.zig`, `src/update.zig`, and `src/main.zig`.
- Download metadata, redirects, destination containment, partial files, and credential forwarding in `src/hf.zig` and `src/update.zig`.
- The backup, restore, and alarm wrappers under `scripts/`, including `scripts/nas/`. Exercise these only with temporary files and stub commands, as the existing restore-drill tests do.

Treat network requests, peer responses, advertised addresses, lease files, manifests, downloaded metadata, and paths as adversarial at the corresponding parser. Clearly state whether an exploit requires no credentials, the shared cluster PSK, origin-write access, or local filesystem access. Compare the resulting access with what that attacker already possesses.

The cluster uses a shared PSK over plaintext HTTP. Lack of transport encryption, per-peer identities, or isolation between legitimate PSK holders is already documented; report a new exploit or a violated guarantee rather than restating those design choices. The origin is writable by its authorized users, so modifying files those users already control is not itself a privilege escalation. Inspect whether an input crosses into another path, credential, process, or protected object.

The updater's checksum and binary are obtained from the same release. A compromised maintainer account or malicious release publisher is a known upstream trust assumption; bugs that bypass URL validation, leak credentials, escape the destination, or violate handover guarantees remain in scope.

## Reproduction environment

The checkout is `/src`. The image installs the manifest-pinned Zig compiler, libfuse3 headers, shell tooling, and Python 3.12 with the exact development dependencies in `uv.lock`. It builds the Debug binary and runs the blocking gate during image creation.

The following commands work without Internet access. The scanner's tools layer runs as root; use the prepared unprivileged account so permission-denial tests exercise ordinary user access:

```sh
runuser -u modelfs-audit -- ./scripts/check.sh
runuser -u modelfs-audit -- zig build test -Dtest-filter=relOk
runuser -u modelfs-audit -- ./scripts/run_e2e_tests.sh
```

`zig-out/bin/modelfs` is the Debug executable. Unit tests live beside the implementation and are imported by `src/root.zig`; `-Dtest-filter=` matches test names. Build temporary experiments and fixtures under `/src/.scratch/`, following `AGENTS.md`. Use generated model-like bytes, temporary origins, and synthetic credentials; no real model downloads or production secrets are necessary.

FUSE mount integration requires `/dev/fuse` and suitable mount privileges. If those are unavailable, test parser, storage, protocol, and handover logic through unit tests and local fixtures, and label the untested integration step. GPU hardware, live NFS, a real NAS, production backup pools, Hugging Face access, and GitHub access are not required for this audit. Never run recovery commands against real storage or contact deployment hosts.

## Findings

Provide the affected commit, source files and symbols, exact attacker prerequisites, a deterministic reproducer, expected and actual behavior, and the resulting security impact. Identify whether a report duplicates a documented limitation. Prefer a small regression test and a minimal patch that preserves protocol and filesystem behavior. Proposed patches and commit messages should contain no attribution or co-author trailers.

Prioritize unauthenticated code execution, access outside the configured origin/cache, credential disclosure, or persistent integrity violations with demonstrated impact. Distinguish those from bounded resource exhaustion, expected access by authorized cluster members, and ordinary operational failures. Report dependency-only vulnerabilities to their upstream maintainers rather than presenting them as ModelFS findings.
