# Clanker security review

Clanker is a Zig agent harness with sandboxed WASM tools, provider adapters,
local HTTP services, and a gated source-improvement loop. Review its own
permission enforcement and host bindings. Defects solely in the WASM engine,
compiler, or other dependencies belong with their maintainers.

The repository's `docs/THREAT_MODEL.md` and `docs/manifest.md` describe the
current controls and known limitations. Investigate independently; a claim
in those documents is not proof that its implementation is correct.

## Inputs and assets

Guest modules, tool manifests, tool arguments/results, retrieved documents,
peer messages, provider responses, HTTP requests, and staged source patches
can be adversarial. Protect provider credentials, private conversation/state
files, host files and subprocess authority, and the code that decides whether
a source change may be promoted.

Prioritize guest access beyond filesystem, network, environment or execution
grants; nested calls inheriting unintended grants; privileged host imports;
path/symlink races; subprocess argument/config injection; credential leakage;
and memory safety or denial of service at host interfaces. Check that source
promotion cannot evade its protected paths and validation requirements.

HTTP listeners default to loopback. Host/Origin checks are not user
authentication. Distinguish the documented trust model and existing limitations
from a newly demonstrated bypass. Authorized hooks, debugging adapters,
external backends and explicitly granted execution are intended capabilities;
show a caller obtaining authority it was not granted.

## Test constraints

The Dockerfile prefetches dependencies and applies `patches/*.patch` before
building. It runs the native, guest, UI, deterministic-gate and mock-provider
end-to-end checks offline. Keep the patches active when reproducing a bug.

Use synthetic configuration, temporary state, mock providers and disposable
repositories. Never load real `.env` files, credentials, conversation stores,
or production peers. Live provider access and network dependency-audit
services are unavailable in the scanner environment.

Provide a minimal reproducer, manifest/configuration, attacker privileges,
affected commit and concrete impact. Reports go privately to `SECURITY.md`'s
contact.
