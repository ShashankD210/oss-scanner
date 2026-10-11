# Threat model

## Purpose and untrusted input

This repository is a collection of plugins, skills, OpenCode adapters and native hooks for Claude Code, Codex and OpenCode. It is not the coding-agent harness itself. The host agent can have shell, filesystem, Git, browser and authenticated GitHub tools; plugin code and hooks run with that user's authority. A plugin installation is a deliberate trust decision, not a sandbox boundary.

Treat repository content, session transcripts, hook input JSON, tool arguments, paths and command strings, GitHub PR titles/bodies/comments/reviews/check output, and SSE event fields as untrusted. A GitHub commenter or contributor does not gain authority to run the agent's tools merely by placing instructions in a PR or lifecycle event. Review the transition from remote lifecycle/comment data to local monitor output and agent tool decisions, as well as hook handling of attacker-influenced paths, commands and transcript data.

## Repository-owned components and boundaries

- Root `package.json`/`package-lock.json`, `.opencode/plugins/agent-plugins.js`, and plugin-local OpenCode adapters implement plugin registration/configuration. Inspect tool/config construction, interpolation, module loading and execution boundaries. The separate `.opencode/package.json` supplies OpenCode runtime dependencies and has no committed npm lock.
- Six Rust workspace members implement `command-chain-separator`, `mediocrity-detector`, `memory-to-repo`, `playwright-cli-headed`, `unrelated-issue-detector` and `windows-bash-guard`. Hook JSON, command parsing, path validation, symlink traversal, filesystem writes, process spawning, memory consumption and guard-bypass behavior matter. Memory persistence and `record-memory-usage.ts` additionally handle project and session files. Hooks should not unexpectedly widen the caller's authority or write outside intended project locations. A hook that advises or blocks some commands is not an OS-level sandbox or proof that all other commands are safe.
- `plugins/watch-pr/skills/watch-pr/watch-pr-monitor.mjs` is the owned, read-only SSE client. It takes a monitor URL, parses streamed events, reconnects/resumes and formats them to stdout. It does not itself edit a checkout, call GitHub or invoke MCP tools. Parsing/framing limits, malformed or oversized messages, cursor handling, redirect behavior, terminal/control-character handling, and confusion between event metadata and untrusted comment text are source-level review targets.
- The monitor URL is an access-bearing capability even though it contains no GitHub credential. Documentation describes a PR/session-scoped, 12-hour capability; the local checkout cannot prove server enforcement. Protect full URLs in arguments, errors, stdout, logs and transcripts. The client allows HTTPS hosts and loopback HTTP rather than enforcing a service-host allowlist, and uses manual redirects. An arbitrary HTTPS hostname alone is not a demonstrated exploit: assess actual attacker control of the URL and consequences such as capability disclosure or unauthorized local network access. Use synthetic loopback URLs in tests.
- GitHub lifecycle data and comments are observations, not authorization or trusted operational instructions. Review the skill/adapter's interaction with privileged root-agent tools without conflating a malicious comment persuading an LLM with a deterministic bug in this repository.
- Plugin manifests, dependency manifests/locks, bundled hook binaries and build/install wrappers form a supply-chain boundary. Assess whether untrusted data can alter executable selection, loading or installation. Deliberately installing a malicious plugin or executing attacker-selected code is not itself a vulnerability. The container rebuilds all six Linux hooks from source and replaces their bundled Linux executables inside the image so tests do not silently exercise only prebuilt artifacts.

## Source vulnerabilities versus prompt-only behavior

Prioritize reproducible source defects: attacker-controlled strings reaching shell/process execution, arbitrary file writes or reads through hook paths, unintended capability exposure, parser/resource-exhaustion bugs, and authorization or input-handling failures in owned adapters/hooks/monitor code. Demonstrate the entry point, trust boundary, attacker prerequisites and effect with a local reproducer.

A comment saying "ignore previous instructions" followed by an agent choosing to obey, an undesirable model answer, or a failure of a natural-language skill instruction alone is prompt-content-only behavior, not an exploitable source security finding. If hostile text also exploits a deterministic source defect (for example unsafely constructed commands or event formatting that crosses an actual machine-enforced boundary), report that defect and show the exploit chain. Do not infer arbitrary tool execution merely from text appearing in the agent transcript.

## Offline preparation and exercise

The checkout is `/src`. The container installs root npm dependencies with `npm ci --ignore-scripts`, installs the separate OpenCode runtime dependencies, preserves npm caches and `node_modules`, fetches the locked Cargo workspace dependencies, builds all Rust targets and release hooks, and retains the Rust toolchain/registry caches and build artifacts. Release binaries have debug information and frame pointers. JavaScript modules and the Node-native TypeScript usage script execute directly; there is no npm compilation script. Windows cross-build wrappers are deliberately not invoked: they need additional cross-toolchains and modify the Git index. Source inspection of Windows-specific handling remains in scope; the image is Linux, not a Windows runtime.

Run from `/src` in the offline container:

```sh
npm run test:opencode
npm run test:watch-pr
npm run test:usage
node --test tests/developing-solidworks-mcp.test.mjs
cargo test --offline --locked --workspace
```

The Node tests use stub adapters, synthetic session/project/ZIP fixtures and local processes; watch-pr tests run a local ephemeral loopback HTTP server, not the hosted service. Loopback networking must remain available even with external networking disabled. Rust tests use temporary/local fixtures. Bash is installed for the command-chain separator's differential `bash -n` fuzz test (default 3,000 iterations; `FUZZ_ITERS` and `FUZZ_SEED` control it). The usage test invokes the source-built Linux memory hook placed at `plugins/memory-to-repo/hooks/bin/memory-to-repo`.

To smoke-test retained dependency caches and rebuild without network:

```sh
npm ci --offline --ignore-scripts
npm ci --prefix .opencode --offline --ignore-scripts
cargo build --offline --locked --workspace --all-targets
```

## Severity guidance

Rate demonstrated arbitrary code execution, unintended privileged tool invocation, sensitive file or capability exfiltration, and arbitrary writes according to attacker access, required user interaction and affected authority. Memory corruption in native code merits high severity when reachable from an untrusted input; support stronger ratings with a realistic exploit. Reproducible local resource exhaustion is generally medium, adjusted for reachability and impact. Guard-policy bypasses need a stated intended policy and demonstrated consequence, not an assumption that these hooks are comprehensive sandboxes.

## Out of scope and leave alone

The deployed `watch-pr.vza.net` MCP/OAuth server, GitHub webhook ingestion, durable snapshots, reconciliation, authentication, actual OAuth scopes, server capability issuance/expiry and data retention are external and not implemented in this checkout. Do not build or audit an invented backend, call the live MCP service, request real OAuth tokens, use live monitor capabilities, or alter real PRs. Likewise do not exercise real browser sessions, user session stores, credentials or remote services used by other skills. Use local synthetic fixtures and inspect source instead. The coding-agent harness and third-party dependencies are distinct products; report an owned integration bug when demonstrated, not assumed vulnerabilities in those external products. No adoption, deployment scale or backend-security claims are made here.
