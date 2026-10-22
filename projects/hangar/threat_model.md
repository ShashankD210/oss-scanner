# Hangar security review

Hangar is a QEMU VM manager with an HTTP API, embedded web interface, remote
client protocol, and optional desktop wrapper. Review Hangar's own code;
defects solely in QEMU, noVNC, SPICE, or other dependencies belong upstream.

## Inputs and assets

Treat HTTP and WebSocket clients, request bodies and uploaded images,
imported VM descriptions, disk paths, and guest display/protocol data as
adversarial. Prioritize API-key bypass, Host/Origin checks, browser-driven
requests, unauthorized console/control access, and file reads or writes
outside the caller's intended VM resources.

Review VM names, disk and network settings as they reach QEMU/QMP, subprocess
argument lists, archive imports/exports, snapshot operations, upload/download
streaming, and VNC/SPICE/serial relays. Investigate memory safety, integer
overflow, path traversal, symlink races, secret disclosure, and remotely
triggered resource exhaustion.

The default daemon is loopback-only when `KV_API_KEY` is unset or equal to
the built-in default. A configured non-default key also enables listening
on all interfaces. TLS termination belongs to the operator's reverse proxy;
the daemon does not provide multi-user accounts. Successful authorized VM
control is intended behavior, not a vulnerability by itself.

## Test constraints

The Dockerfile caches build dependencies, builds all shipped binaries, and
runs `zig build check`, JavaScript lint, types, and unit tests offline.
Browser suites are separate and require Chromium.

Run daemon-backed tests only inside an isolated container: the daemon owns
`/tmp/hangar-daemon.sock`, so changing its TCP port alone does not isolate it.
Use temporary `HOME` and `HANGAR_CONFIG_HOME` values and `KV_API_KEY=hangar`.
Use disposable images, software emulation, and local fake peers only. Never
load the operator's real VM inventory, alter host networks, or connect to
production hypervisors.

Report a minimal reproducer, required deployment settings, attacker
privileges, affected commit, and concrete impact privately to `SECURITY.md`'s
contact.
