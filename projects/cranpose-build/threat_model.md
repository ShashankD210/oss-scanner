# Threat model

## What this project does and where untrusted input enters
Cranpose Build is a command line tool and a Rust library. It builds, packages and launches Cranpose applications for macOS, Linux, Windows, Android and iOS. It runs on the computer of the developer, with the permissions of that developer.

Untrusted input:
- Files in the project that is being built: `CranposeBuild.toml`, `Cargo.toml`, asset paths and Gradle files. A developer may open a project from someone else.
- Downloads of build tools, such as Zig and cargo-xwin, in `src/download.rs`. These must use HTTPS, stay within a size limit and match a SHA256 checksum before use.
- Archives (`zip` and `tar.xz`) that the tool extracts after a download.
- Output of the programs that the tool starts, such as `adb`, `xcrun`, `cargo` and `tar`.

## Components that matter most / least
Most: `src/download.rs` and the archive extraction, `src/config.rs` and `src/project.rs` (reading project files), `src/process.rs` and `src/tools.rs` (starting programs and finding them on `PATH`), and the packaging code in `src/engine.rs` and `src/platform.rs`.

Least: the `xtask` directory and the tests.

## How to exercise it
Build from `/src` with `cargo build`. The tests in `tests/` cover the Android, build and download code. Commands such as `cranpose-build plan` print what would run without starting a build.

## How you rate severity
- A path in an archive or a project file that makes the tool write outside its target directory is high.
- A way to run a command from a project file that the developer did not expect to run is high.
- A download that is accepted without a matching checksum, or over plain HTTP, is high.
- Command injection through a value from a project file, a device name or a tool output is high.
- A panic on malformed input is low.

## Anything to leave alone
- Do not report that the tool runs `cargo` and Gradle on the project. Building the project runs its build scripts, and that is the purpose of the tool.
- Do not report that a developer who builds an untrusted project runs its build scripts.
