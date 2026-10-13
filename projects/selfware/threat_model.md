# Threat model

## What this project does and where untrusted input enters

Selfware is an autonomous agentic AI development environment and recursive self-improvement engine written in Rust. It inspects, analyzes, executes, and modifies source repositories using frontier models and specialized system tools.

Untrusted input enters in these places:

- **Untrusted source repositories and workspace files.** Selfware reads, indexes, and edits user codebases. Untrusted inputs include malicious repository filenames, hidden unicode, crafted control characters, prompt injection embedded in source comments or markdown guidance (`AGENTS.md`, `CLAUDE.md`), symlinks pointing outside the workspace, and deeply nested directories. The system must enforce canonical workspace boundaries, symlink-free descriptor-relative traversal (`O_NOFOLLOW`), and strict exclusion of private agent state (`.claude`, `.codex`, `.agents`, `.selfware`, `.git`).
- **Network and browser tools.** The agent can browse web pages, take screenshots, fetch documentation, and interact with web applications (`src/tools/net_policy.rs`, `src/tools/browser.rs`, `src/tools/page_controller.rs`, `scripts/playwright-bridge.js`). Untrusted web servers can attempt SSRF to loopback or link-local cloud metadata (`169.254.169.254`), private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), IPv4-mapped IPv6 literals, DNS rebinding, and WebRTC STUN/TURN egress leaks. Additionally, outbound tool calls must not leak credentials (`SELFWARE_API_KEY`, Bearer tokens, GitHub tokens) in URLs, headers, or request bodies.
- **Process execution and shell isolation.** The agent interacts with long-running shells and subprocesses (`src/tools/pty_shell.rs`, `src/devops/process_manager.rs`, `src/tools/process_guard.rs`). Untrusted commands must not forge completion exit codes, poison shared shell state (environment variables, aliases, traps) across subsequent commands, escape process group isolation (`pgid`), or leave unkillable zombie descendants. Child processes must also have sensitive parent environment variables scrubbed before execution.
- **Session persistence and checkpoints.** Selfware persists execution histories, checkpoints, and RAG vector store snapshots (`src/session/checkpoint.rs`, `src/session/chat_store.rs`, `src/analysis/vector_store.rs`). Malicious or corrupted checkpoint files must not cause arbitrary code execution, crash loops, path traversal during atomic replacement, or advisory file lock deadlocks.
- **Artifact-scoped coordination and read-only harness.** The artifact analysis runtime (`src/orchestration/primitives/`) enforces descriptor-relative artifact reads, frozen hash bindings, and metadata-only message brokering. Outgoing inter-worker messages must not carry arbitrary model prose, quotes, or unverified source excerpts across artifact boundaries.

## Components that matter most / least

- **Most important:**
  - `src/safety/` (`checker/validation.rs`, `path_validator.rs`, `source_context.rs`, `net_policy.rs`, `recursive_read.rs`)
  - `src/tools/` (`pty_shell.rs`, `browser.rs`, `page_controller.rs`, `http.rs`, `file.rs`, `process_guard.rs`, `workspace_root.rs`)
  - `src/orchestration/primitives/` (`safe_read.rs`, `activation.rs`, `broker.rs`, `runtime.rs`, `types.rs`)
  - `src/session/` (`checkpoint.rs`, `chat_store.rs`, `cache.rs`)
- **Also in scope:**
  - `src/agent/` (`context_files.rs`, `context_management.rs`, `tool_dispatch/`, `verification.rs`, `checkpointing.rs`)
  - `src/devops/process_manager.rs`
  - `src/cognitive/` (`rag.rs`, `memory_system.rs`, `memory_hierarchy/`)
- **Less important:**
  - `src/ui/` (TUI garden animations, console formatting)
  - `src/boot/` (CLI startup banners, initialization wizards)
- **Out of scope:**
  - `benches/`, `tests/` (test fixtures, benchmark harnesses, mocks)
  - Third-party dependencies unless Selfware uses them in an unsafe or vulnerable manner

## How to exercise it

- The project is pre-built with `cargo build --locked --all-targets`.
- Rebuild offline: `cargo build --offline --locked`
- Run the core unit test suite:
  `cargo test --offline --locked --lib`
- Run offline integration tests:
  `cargo test --offline --locked --test integration_offline`
- Check code formatting and Clippy:
  `cargo fmt -- --check`
  `cargo clippy --offline --all-targets -- -D warnings`

## How you rate severity

- **Critical:**
  - Remote code execution (RCE) on the host machine outside authorized workspaces or sandboxes.
  - Arbitrary file writes outside designated workspace boundaries resulting in host compromise.
  - Exfiltration of credentials (`SELFWARE_API_KEY`, SSH keys, cloud tokens) to unauthorized remote servers.
  - Complete bypass of SSRF protections allowing access to cloud metadata services (`169.254.169.254`) or internal network services.
- **High:**
  - File path traversal or symlink escapes that allow reading or overwriting arbitrary files outside the repository root.
  - Prompt injection that breaks out of the data block framing to execute unauthorized mutating tools.
  - Subprocess escape or command injection in `pty_shell` or `shell_exec`.
- **Medium:**
  - Denial of service (unbounded loops, memory exhaustion, unkillable processes, or permanent pipe stalls) triggered by untrusted source files or web responses.
  - Cache poisoning or checkpoint delta deserialization corruption.
- **Low:**
  - Information leaks where the user explicitly provided the file path or command flag.
  - Vulnerabilities requiring local root or pre-existing arbitrary control over the host user account.

## Anything to leave alone

- Do not report that `shell_exec` or `pty_shell` executes shell commands when the user explicitly approved or enabled autonomous execution mode.
- Do not report that an AI model can generate incorrect code; prompt outputs are treated as untrusted suggestions until validated by verification gates.
- Do not report that `SELFWARE_ALLOW_PRIVATE_NETWORK=1` permits private IP connections; it is an explicit environment opt-in for internal testing.
