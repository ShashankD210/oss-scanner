# Threat model: tyd

tyd keeps persistent, remotely attachable terminal sessions on a machine. A daemon holds PTY/shell sessions under
`~/.tyd/live/`; a client pairs with that host, then creates or attaches sessions. One binary is both ends
(`cmd/tyd`); the daemon also ships a Control Panel (`cmd/controlpanel`) and a standalone rendezvous relay
(`cmd/relay`).

The blast radius of a bug here is a remote shell: a peer that gets past the handshake owns a process running as the
daemon's user, with that user's files, keys and sockets. The authentication handshake, the capability checks and
the frame parser are the highest-value code in the repository.

## What this project does and where untrusted input enters

- The frame protocol: length-prefixed JSON (uint32 big-endian length, at most 1 MiB) over a Unix socket, TLS TCP,
  QUIC, or a relay splice. Both directions are attacker-controlled.
- The Ed25519 challenge-response handshake: `challenge {version, nonce}` then `auth {version, public_key, sig}`. The
  nonce is 32 bytes. The version field, the signature and the public key are all attacker-supplied.
- The channel binding: on every transport that carries TLS, the login signature covers the TLS exporter, so a
  signature cannot be carried to another connection.
- Peer metadata from the Control Panel: daemon ids, public keys, signed endpoints. A client dials what the Control
  Panel publishes, so a hostile Control Panel is untrusted input to the client.
- Invites: single-use tokens minted by the Control Panel, valid for 10 minutes.
- The Control Panel's HTTP API: pairing, invites, endpoints, revocation.
- File paths and file content for `file_read` and `file_write`, bounded by `--file-root`.
- Names a person is asked to paste: session aliases and peer nicknames. These become part of a shell command.
- Terminal bytes in both directions, plus window-size and signal requests.

## Components that matter most

- `internal/auth`, `internal/protocol`, `internal/transport`, `internal/peernet`: the handshake, the frame parser,
  the TLS and QUIC setup, the channel binding, and the capability checks. A flaw here is a remote shell.
- `internal/server`, `internal/session`, `internal/live`, `internal/procs`: the daemon side, the PTY, the approval
  gate, and the live agent that owns the session.
- `internal/controlpanel`, `internal/relay`: the pairing service and the rendezvous relay. The relay must never be
  able to read or alter a session.
- `internal/mcp`: the MCP server behind `tyd mcp`. It is the path a model reaches the daemon through.
- `internal/strutil` (`ShellSafe`), `internal/fileroot`, `internal/safefile`, `internal/audit`: the rules that keep
  untrusted names out of a shell, file operations inside the file root, and audit records in a hash chain.

## Components that matter least

- `internal/termclean`, `internal/ttyutil`, `internal/paths`: cosmetic helpers.
- `cmd/controlpanel` and `cmd/relay` are separate processes with a smaller surface. They are in scope, but a finding
  in the Control Panel's HTML or CSS is not a security finding.

## How to exercise it

- `go test ./...` is the main harness. It covers the handshake, the frame parser, the capability checks, the audit
  chain, the file root and the session lifecycle.
- `make acceptance` (`scripts/acceptance_send_read.sh`) drives a real daemon over the wire: it creates a session,
  sends bytes and reads the output back. It is the shortest path to a working reproducer.
- The binaries are built at `bin/tyd`, `bin/controlpanel` and `bin/relay`. `bin/tyd up --listen 127.0.0.1:PORT`
  starts a daemon; `bin/tyd register`, `bin/tyd invite` and `bin/tyd accept` drive a pair of peers.
- `go test -race ./...` works in this image, and the daemon is a goroutine-heavy program, so please try it.

## How you rate severity

- **Critical**: remote code execution as the daemon's user without a completed handshake or without the capabilities
  that were granted; a bypass of the approval gate (`pre` or `post`); forging, replaying or re-originating a login
  signature; a relay or Control Panel that can read or alter session bytes.
- **High**: a peer exceeding the capabilities it was granted (attaching without `attach`, reading without `attach`,
  writing without `write`, acting on a session it does not own); a path escape from `--file-root`; reaching another
  peer's identity key, trust file or audit log from the network side.
- **Medium**: a denial of service a peer can trigger (a crashed daemon, a wedged live agent, unbounded memory or
  file growth); an audit record a peer can forge or remove; a leak of metadata a peer should not see.
- **Low**: anything that needs local code already running as the daemon's user.

## Anything to leave alone

- **The local boundary is not a boundary.** A session runs as the daemon's user, in the daemon's home. Code already
  running there can read `~/.tyd/id_ed25519`, rewrite `~/.tyd/audit.log` and `peers.json`, use `~/.tyd/tyd.sock`,
  approve its own pending request, and stop the daemon. `docs/security.md` says this plainly. Do not report it as a
  vulnerability. `TYD_SESSION`, `TYD_PEER` and the `SO_PEERCRED` check are speed bumps, and the documents say so.
- **Approval modes and audit logs are not a boundary against local code.** Same reason.
- **`--file-root` is not a sandbox.** It bounds `file_read` and `file_write` to a directory and refuses `/`. A peer
  with `session_send` has a shell, and the shell reaches wherever the daemon's user reaches. Do not report "the file
  root does not confine a shell".
- The `unix` transport has no TLS session, so its login signature carries no channel binding. That is deliberate and
  documented: a process on the same account already holds the identity key, so a forged login buys it nothing.
- The audit log is hash-chained, and deleting the whole file and starting a fresh chain is not detected. That is
  documented, with the reason. Report it only if a peer can do it over the network.
- The PTY echoes whatever is written to it unless the program in front turned echo off. `session_send` with
  `secret: true` refuses to write while echo is on, and that refusal is the design. A report that says "a secret can
  be echoed into the scrollback" repeats the documented behaviour.
- Third-party code is not tyd's code. A finding in `golang.org/x/...`, `quic-go`, `coder/websocket` or `creack/pty`
  belongs to those projects.
- **`make acceptance` can stop at its UTF-8 boundary check inside a container**, with `returned exactly 100 bytes, so
  the cut landed on a boundary anyway`. That is the script's own precondition, not a finding: it wants the page to
  begin inside a multibyte character, and in a container the shell answers the multibyte payload with a completion
  prompt, so the page begins on ASCII and the cut lands on a byte boundary instead. The reply is still valid UTF-8 of
  exactly the requested length, which is the correct behaviour. The same script passes on a developer machine and on
  the project's own CI.

## Report format

A minimal reproducer and a small candidate patch; one issue per report, deduplicated by root cause. Where a finding
needs a live peer, the acceptance script is the shortest way to reach one: it starts a daemon in a temporary HOME and
talks to it over the wire.
