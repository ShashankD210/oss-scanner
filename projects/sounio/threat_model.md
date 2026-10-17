# Threat model

## What this project does and where untrusted input enters
- Sounio is a programming language and self-hosted compiler (`self-hosted/compiler`, entry point `self-hosted/compiler/main.sio`) with a standard library (`stdlib/`) for scientific computing: quantities with units, values with uncertainty, and declared effects.
- Treat every `.sio` source file given to `bin/souc` (`check`, `compile`, `build`, `run`) as untrusted input. The lexer, parser, type/effect checker, IR lowering and native x86-64 emission must not crash, hang, corrupt memory, or emit code that does something the source does not say.
- Also untrusted: package manifests (`sounio.toml`, `souc pkg ...`), ontology bundles and caches (`.dontology`, `.ontocache`), and data files read by `stdlib/` I/O and format modules.

## Components that matter most / least
- Most: the compiler front end and the native code generator (memory safety of the compiler itself, and correctness of emitted code), the checker for the epistemic and effect types (`Knowledge<T>`, units, effects), and package handling.
- The compiler is itself written in Sounio and shipped as a committed static ELF (`bin/madaros-linux-x86_64`, started through `bin/souc`). That binary can lag `self-hosted/` source; report issues against the source where they are visible there.
- Least: `docs/`, `papers/`, `paper/`, `website/`, `site-next/`, `archive/`, `scratchpad/`, `slurm-jobs/`, and the dataset/model training material. `vendor/` is third-party and out of scope.

## How to exercise it
- `bin/souc check|compile|run FILE.sio` with `SOUNIO_STDLIB_PATH=/src/stdlib`. `examples/` holds small programs, `tests/` holds the regression corpus (`tests/compile-fail`, `tests/conformance`, `tests/fuzz` where present), and `scripts/run_sio_test_suite.sh` runs the suite.
- Compile and run generated programs inside the VM; they are as untrusted as the source that produced them.

## How you rate severity
- Memory corruption in the compiler or in compiled output (buffer overflow, use-after-free, out-of-bounds read or write) reachable from attacker-controlled source: high at least; critical if it gives controlled code execution.
- Miscompilation that silently changes a program's numeric result, or that lets a program bypass a declared effect, unit or uncertainty check: high, because Sounio is used for clinical and scientific calculations.
- A crash or unbounded resource use of the compiler on malformed input: medium. Path traversal or unintended file access during package or import resolution: high.
- Reports that need a user to run a malicious `.sio` program that is already allowed to use declared effects (IO, filesystem, network) are not vulnerabilities.

## Anything to leave alone
- Known limitations listed in `docs/compiler/KNOWN_LIMITATIONS.md`, and the GPU/PTX paths documented there as incomplete.
- Findings that only apply to the legacy bootstrap seed `bin/souc-lean-single-x86_64`, unless the same defect is present in `self-hosted/` source.
