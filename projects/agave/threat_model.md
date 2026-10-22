# Agave security audit

Agave is an independently maintained Zig LLM inference engine. Public adoption is small; do not assume a large installed base. The repository's `SECURITY.md`, `docs/THREAT_MODEL.md`, and `docs/API.md` describe current contracts and known limitations. Recheck their claims against the scanned commit.

## Scope and priorities

Audit Agave's own implementation and shipped tooling, prioritizing:

- Adversarial GGUF, SafeTensors/shards, tokenizer metadata, LoRA, calibration files, and image/video inputs: lengths, integer arithmetic, tensor shapes, offsets, allocations, traversal, and partial-failure cleanup (`src/format/`, `src/tokenizer/`, `src/lora.zig`, `src/image.zig`).
- HTTP parsing, bearer authentication, Host/CORS/CSRF checks, JSON/SSE, resource bounds, conversation persistence, KV import/export, and idempotency (`src/server/`).
- Hub download metadata, redirects, credential forwarding, cache paths, partial downloads, updater URL/checksum validation, and atomic executable replacement (`src/pull.zig`, `src/update.zig`).
- Untrusted peer frames, dimensions, rank/discovery metadata, shared-memory ownership and cleanup, and arithmetic across distributed transports (`src/parallel/`).
- Browser/WASM input handling and origin boundaries (`web/`, `src/wasm_entry.zig`) and the conversation backup/restore script using temporary fixtures only.

State the exact attacker prerequisite: an unauthenticated HTTP caller, an API-key holder, a model publisher, a peer on the trusted network, a same-uid process, or a local filesystem writer. Demonstrate a concrete violated boundary rather than assuming all inputs are equally privileged.

Known limitations include unauthenticated distributed ports, one shared trust domain for API-key holders, shared KV/replay/conversation state, and checksums from the same release publisher rather than independent update signatures. Reconfirm whether each limitation is still present; do not report its existence alone as a newly discovered vulnerability. Prompt injection that merely makes a model request a declared tool is already documented: show an Agave control bypass or an additional authority crossing. GPU driver vulnerabilities and dependency-only findings belong upstream.

## Offline reproduction

The checkout is `/src`. The image installs the repository-pinned Zig and Bun versions, Node, locked web dependencies, Java, ShellCheck, Ruff, and uv. It builds all default native binaries and runs the full local gate during image creation. No real model weights, API tokens, external accounts, GPUs, or network services are required for unit/parser/protocol tests.

```sh
zig build -j2
zig build ci -j2
zig build test -j2 -Dtest-filter=GGUF
bash scripts/test-file.sh src/format/gguf.zig
```

Use `zig build test -Dtest-filter=<substring>` for a small reproduction; an unmatched filter is an error. Tests and the diagnostic binary use ReleaseSafe so assertions remain enabled. Keep temporary experiments and synthetic data in `/src/.scratch/`. Ordinary CPU tests work offline; GPU hardware tests may skip or require an unavailable driver. Do not contact production inference hosts, download model weights, or read real prompt stores. Do not run backup or restore against operator data. The image keeps the source and compiler caches so new focused tests can be compiled without network access.

The optional sanitizer and sustained fuzz jobs, hardware kernels, full model inference, cross-platform runtime matrix, and WASM forward-pass correctness are separate from the local gate; label anything not exercised. Existing research reference packages such as PyTorch are not required or installed for this audit.

## Findings

Include the affected commit, files/symbols, attacker prerequisites, exact input/reproducer, observed versus intended behavior, and demonstrated impact. Prefer a small regression test and a minimal patch that preserves model/API behavior. Distinguish memory corruption, auth bypass, credential/path escape, and persistent data loss from bounded resource costs or documented operational limitations. Proposed patches and commit messages should contain no attribution or co-author trailers.
