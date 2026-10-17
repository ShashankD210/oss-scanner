# Threat model: yogit

## What this project does and where untrusted input enters
yogit (`yogit.mjs`, one file, no dependencies) is a mailbox for coding-agent sessions (Claude Code, Codex, Cursor, humans). Messages are stored as git refs in the shared ref store of a repository, and synced with `git fetch`/`git push` to a remote.

Untrusted input, in rough order of importance:
- **Message content and metadata received from other parties**: the JSON packets and ref names that arrive via `yogit sync` from a remote, or that another local agent or worktree wrote into the same ref store. Senders are other agents, possibly other people, and are not trusted.
- **Ref names and identities** taken from those refs and from command-line arguments, all of which reach `git` (through `execFileSync`, no shell).
- **The optional "ring" command** (`spawn`), configured by the local user in their own git config. Treat the config as trusted, but anything a *remote* or a *message* can influence about what is run, or its arguments, is in scope.
- **Agent configuration files** that `yogit init` reads and edits (Claude, Codex and Cursor settings under the user's home directory, or `CLAUDE_CONFIG_DIR` / `CODEX_HOME`).
- **Terminal output**: message text printed to a TTY.

## Components that matter most / least
- Most: message parsing and validation (JSON parsing of packets and of the ring spec); ref-name and path handling; anything that builds `git` arguments or spawn arguments from received data; `init`'s edits to other tools' config files.
- Less: pure output formatting, colour handling and help text.
- Out of scope: `git` itself and Node.js itself.

## How to exercise it
- `node --test test/` runs the test suite (`test/yogit.test.mjs`, `test/reference-transaction.test.mjs`). Tests create temporary git repositories, so no network is needed.
- `node yogit.mjs init`, `send <to> <text>`, `inbox`, `outbox`, `sync <remote>`, `reply <id> <text>` exercise the main paths against a local repo. A second local repo can act as the remote.

## How you rate severity
- Critical: arbitrary command execution or file write triggered by a received message or a fetched ref (no local user action beyond running normal yogit commands such as `inbox` or `sync`).
- High: argument injection into `git` or the ring command from received data; path traversal or writes outside yogit's own refs and files; a message able to alter or delete other users' refs or the repository's branches; corruption of other tools' config files by `init`.
- Medium: terminal escape-sequence injection from message text; denial of service from a crafted packet (crash, hang, unbounded memory).
- Low: information disclosure of message metadata to parties who already share the repository.

## Anything to leave alone
- Commands a local user deliberately configures themselves (their own ring command) running with their own privileges is the intended design, not a vulnerability.
