# Threat model

## What this project does and where untrusted input enters
This is a template for IntelliJ Platform plugins that are written in Rust with a Cranpose user interface. A shared Rust host calls the IntelliJ SDK through JNI. A separate Cranpose process draws the interface and sends BGRA frames to the host over a loopback connection that is authenticated. The host shows the pixels and sends mouse, keyboard, scale and visibility events back. Plugin authors copy the template, so the code below becomes part of their plugins.

Untrusted input:
- Messages on the loopback connection, including the token check, in `sdk/host/src/session.rs` and `sdk/process`.
- Frame data and events that pass between the processes.
- Files that the cache, watch and authoring crates read from a project, and the archives that the tools crate unpacks.
- Editor text that the host passes to the interface.

## Components that matter most / least
Most: `sdk/host` (the session and token code), `sdk/process` (the child process and frame code), `sdk/jvm-bridge` (JNI), `sdk/cache`, `sdk/tools` and `sdk/watch`.

Least: `ui`, `sdk/ux`, `sdk/ui`, `sdk/decorations` and `sdk/gpu`, which only draw the interface, and `xtask` and the tests. The Cranpose crates are scanned in their own enrolment.

## How to exercise it
Build from `/src` with `cargo build --no-default-features --workspace`. The tests in `sdk/process` and `sdk/host` cover the connection and the process life cycle without an IDE. A GPU and an IDE are not available in the scan environment.

## How you rate severity
- A process on the same computer that can use the loopback connection without the secret is high.
- A path in an archive or a cache entry that writes or reads outside its directory is high.
- Memory corruption in the JNI code or in frame handling is high.
- A frame size or a message length that makes the host allocate without a limit is medium.
- A panic from malformed messages is low.

## Anything to leave alone
- Do not report the missing JVM or IDE in the scan environment.
- Do not report that the template starts a child process. That is its design.
