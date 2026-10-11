# Threat model

## What this project does and where untrusted input enters
Cranpose Stability is a Rust library, a command line linter and a GitHub Action. It reads Rust source files, Cargo manifests and `Cargo.lock`, and it reads dependency sources that Cargo has already unpacked under `CARGO_HOME`. It reports the stability of composable function parameters in text, JSON and SARIF. It does not run Cargo, fetch anything, expand procedural macros or run build scripts.

Untrusted input is the whole project tree that the tool analyses, which may come from a pull request of someone else in CI. That includes Rust syntax, `Cargo.toml`, `Cargo.lock`, the path dependencies and the unpacked dependency crates.

## Components that matter most / least
Most: `src/project.rs` and `src/deps.rs` (finding and reading files, following paths and symbolic links), `src/parse.rs` and `src/macros.rs` (parsing source), and `src/output.rs` (writing SARIF and JSON, and the output file).

Least: `src/types.rs` rules about which types count as stable. A wrong verdict is a quality bug.

## How to exercise it
Build from `/src` with `cargo build`. Run `cargo test`. The tests in `tests/` build small projects in a temporary directory and run the analysis on them. Run `target/debug/cranpose-stability --root <dir>` on a crafted tree to test the command.

## How you rate severity
- Reading or writing files outside the root through `..` paths, symbolic links or a crafted `path` dependency is high.
- Running a command or code from analysed input is critical.
- Output that injects content into SARIF or into GitHub workflow commands is medium.
- A panic, stack overflow or unbounded memory use on a crafted source file is medium, because CI may run on untrusted pull requests.
- A wrong stability result is low and out of scope.

## Anything to leave alone
- Do not report wrong or missing stability verdicts.
- Do not report problems in third-party crates unless this project calls them in an unsafe way.
