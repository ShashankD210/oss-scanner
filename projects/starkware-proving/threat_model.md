# Threat model: StarkWare proving monorepo (stwo, stwo-cairo, stwo-circuits, proving-utils)

## What this project does and where untrusted input enters
This repository is StarkWare's Circle STARK proving stack in one Cargo workspace (see `CLAUDE.md` for the crate map):
- **stwo** (`crates/stwo`, `constraint-framework`, `air-utils*`): the AIR-agnostic prover and verifier.
- **stwo-cairo** (`adapter`, `cairo-air`, `prover`, `common`, `cairo-serialize*`, and the Cairo verifier in
  `stwo_cairo_verifier/`): proving and verifying Cairo program executions, including recursive verification in Cairo.
- **stwo-circuits** (`circuits`, `circuit_*`, `stark_verifier`, `cairo_verifier`, `unpacker`): a circuit DSL and a
  circuit-based prover and verifier.
- **proving-utils** (`stwo_run_and_prove*`, `privacy_*`, `leaf_*`, `cairo-program-runner*`, `vm_runner`): end-to-end
  run-and-prove, recursive proof trees and privacy proving.
- **stwo-air-infra** (`air_*`, `airs`, `compiled_*_air`, `eval_air_fn_constraints`): the code generator that produces
  the AIR code in the components above.

The verifier is the security boundary. **Proofs are untrusted**: a verifier (Rust, Cairo or circuit) receives proofs,
public inputs and claimed outputs from an adversarial prover. Serialized proofs and public data
(`cairo-serialize`, `circuit_serialize`, `circuit_cairo_serialize`, `leaf_proof_format`) are untrusted bytes.
Inputs to the prover (Cairo programs, PIEs, private inputs) come from the party running the prover and matter mainly
for privacy proving, where private inputs must not leak through the proof.

## Components that matter most / least
Most important, in order:
1. **Soundness**: anything that lets a verifier accept a proof of a false statement. This covers field arithmetic,
   constraints and their evaluation, the generated AIR components, lookups and logup sums, FRI, the polynomial
   commitment scheme, out-of-domain sampling, the Fiat-Shamir channel (anything absorbed or not absorbed into the
   transcript), Merkle commitments, proof-of-work, and public-input and memory checks. The same code exists in Rust
   (`crates/stwo`, `cairo-air`), in Cairo (`stwo_cairo_verifier/`) and in circuits (`stark_verifier`, `cairo_verifier`);
   divergence between the three verifiers is itself a finding.
2. **Completeness**: valid proofs rejected, or the prover producing proofs that do not verify.
3. **Privacy**: private inputs leaking through privacy proofs (`privacy_prove`, `privacy_circuit_verify`).
4. **Robustness of the verifier on malformed proofs**: panics, unbounded allocation or hangs while deserializing or
   verifying.
The `.claude/rules/*-guide.md` files tag files `[SOUNDNESS-CRITICAL]` and `[SECURITY-CRITICAL]`; use them to prioritize.

Out of scope: `crates/examples`, `stark_verifier_examples`, `dev_utils`, `visualizer/`, `scripts/`, benchmarks, wasm
builds and test-only code, except where they reveal a bug in production code. Performance regressions are not
security findings.

## How to exercise it
- The build compiled every test binary with `cargo nextest run --workspace --no-run` (dev profile, debug assertions on).
  Run one crate's tests with `cargo nextest run -p <package>`, for example `-p stwo --features prover`,
  `-p stwo-cairo-prover`, `-p cairo-air`. Slow tests sit behind `--features slow-tests`; CI runs them with
  `--cargo-profile ci -P ci` (a release build, which compiles offline from the fetched crates).
- `test_data/` and the crates' own `test_data/` directories hold real proofs to mutate when testing a verifier.
- The Cairo verifier: `cd stwo_cairo_verifier && scarb test -p <package>`, with `--features qm31_opcode` or
  `--features poseidon252_verifier` as CI does. Packages: `stwo_verifier_core`, `stwo_verifier_utils`,
  `stwo_constraint_framework`, `stwo_circuit_air`, `stwo_circuit_verifier`, `bounded_int`.

## How you rate severity
- **Critical**: any soundness break, i.e. a verifier (Rust, Cairo or circuit) accepting a proof of a false statement,
  including through Fiat-Shamir omissions, missing constraints, unchecked public inputs or memory, or deserialization
  that lets a proof bind to different data than was checked. Leaking private inputs from a privacy proof.
- **High**: a verifier rejecting valid proofs produced by the honest prover (completeness), or verifiers disagreeing
  on the same proof; a malformed proof that crashes a verifier where the verifier runs as a service.
- **Medium**: malformed proofs causing excessive memory or time in a verifier; prover crashes on valid but unusual
  inputs.
- **Low**: prover crashes on malformed inputs from its own operator, and test or tooling issues.

## Anything to leave alone
- Do not report reduced security parameters used only in tests or examples.
- Generated files (marked `// This file was created by the AIR team.`) are in scope, but a fix belongs in
  stwo-air-infra; say so in the proposed patch rather than editing the generated file.
- `unsafe` blocks in SIMD backends are in scope only if a concrete memory-safety bug is demonstrated.
