# Phobos security review

Phobos is a Zig implementation of the Moonraker printer-control API. The
repository's original code is the target; defects solely in SQLite, Zig, or
other dependencies belong with their maintainers.

## Inputs and assets

Treat HTTP request targets, headers, bodies, multipart uploads, WebSocket
frames, JSON-RPC messages, MQTT messages, and files from network clients as
adversarial. Review the authentication and trusted-client rules in
`src/components/authorization.zig`, including proxy forwarding, JWTs, API keys,
one-shot tokens, and access shared across transports.

Priorities are authentication bypass, unintended printer commands, access
outside configured file roots, symlink and rename races, credential exposure,
memory corruption, and remotely triggered resource exhaustion. Uploaded and
copied files, archive creation/extraction, metadata parsing, database values,
and Klippy messages should not escape their documented limits.

The update system may replace operator-configured installations. Review
release/archive validation, repository and destination restrictions, and
rollback behavior. Do not assume every host command runs: reboot, shutdown,
and package-management behavior has deliberate no-op or injected-runner paths.
See `docs/UPDATES.md`, `docs/FILE_MANAGEMENT.md`, and `docs/MACHINE.md`.

## Test constraints

Run `scripts/test.sh Debug` and `scripts/test.sh ReleaseSafe` for the standalone
native suites. Python tooling is available through `uv run --locked`; the
Dockerfile also checks the API surface and generated table offline.

Use temporary data directories and local fake services only. Never connect to
a real printer, issue G-code to hardware, trigger physical motion or heat,
alter host services, or use real credentials. A trusted operator deliberately
configuring a command or a writable installation is not itself a vulnerability;
show an untrusted caller crossing an enforced restriction.

Provide a minimal reproducer, required configuration, affected commit,
attacker privileges, and concrete impact. Reports go privately to the security
contact in `SECURITY.md`.
