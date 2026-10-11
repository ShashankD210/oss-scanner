# Threat model

## What this project does and where untrusted input enters
Cranpose for IntelliJ IDEA adds Cranpose support to IntelliJ IDEA and RustRover. The Studio interface in `ui` is written in Rust and drawn by Cranpose. The native host in `ide-host` connects to the IntelliJ SDK through JNI. The `dev-runner` crate watches the source files of the open project, decides whether an edit needs a rebuild, and builds and runs the application of the developer with `cargo` or a compiler session that listens on `127.0.0.1` at a free port. The plugin shares its process, cache, watch and host crates with the `cranpose-intellij-plugin-template` repository, which it takes as a git dependency.

Untrusted input:
- Source files, `Cargo.toml` and `Cargo.lock` of the project that is open in the IDE. A developer may open a project from someone else.
- Edits of literals and colors that the plugin writes back into source files.
- Compiler and Cargo output, including JSON diagnostics, that the dev runner parses.
- The starter project that the New Project wizard creates, and the files it unpacks.
- Messages from the application or the compiler session on the loopback port.

## Components that matter most / least
Most: the dev runner (`dev-runner/src`), the code that finds and rewrites literals in source, the starter creation code, the JNI entry point in `ide-host/src/lib.rs`, and the messages on the loopback port.

Least: `samples/`, `docs/`, `plugin/` packaging files, `xtask` and the tests. Code that comes from the template repository is scanned in its own enrolment.

## How to exercise it
Build from `/src` with `cargo build --no-default-features --workspace`. The tests under `dev-runner/tests` cover the edit queue, the reload policy and the runner without an IDE. There is no GPU or IDE in the scan environment.

## How you rate severity
- A source file or manifest that makes the plugin run code beyond what building the project already runs is high.
- A path in a message or in the starter that makes the plugin write or read outside the project or the cache directory is high.
- A rewrite of a source file that changes code outside the literal that the developer edited is medium.
- Another local user or a web page that can send messages to the loopback port and change the running application or the files is high.
- Memory corruption in the JNI host is high.
- A panic from malformed source code or compiler output is low.

## Anything to leave alone
- Do not report that opening a project and building it runs the build scripts and procedural macros of that project. That is how Cargo works.
- Do not report the missing JVM or IDE in the scan environment.
