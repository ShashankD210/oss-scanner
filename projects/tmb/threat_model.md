# Threat model — TMB plugin

## What this project does and where untrusted input enters
TMB is a Claude Code plugin: a set of PreToolUse/PostToolUse/SessionStart hooks (`scripts/hooks/`, bash), an MCP server (`mcp/trajectory-server/`, TypeScript on Node 22, SQLite + a kuzu graph) and prompt files (agents, skills, commands). It orchestrates three agent roles (bro plans, swe implements in a git worktree, pr-reviewer gates the push) and keeps the project's workflow state in `.claude/tmb/trajectory.db` inside the user's project.

Untrusted input reaches it from:
- Tool calls made by LLM agents: every hook parses the tool input JSON (Bash command strings, file paths, Agent spawn prompts) and every MCP tool takes arguments from an agent. A coerced or prompt-injected agent is the main adversary; the hooks are the guardrails and must not be bypassable by crafted input.
- Remote content: GitHub/GitLab issue and PR bodies, PR review comments and CI logs are fetched (`gh`/`glab` subprocesses) and written into the DB or shown to agents (`/monitor`, issue sync). Treat all of it as attacker-controlled text.
- Third-party "cheatcodes": skills, MCP servers and plugins found on marketplaces and installed on request (`cheatcode_search` → `cheatcode_vet` → `cheatcode_install`). Supply-chain input.
- The repository being managed: file names, README text, git branch names and commit messages flow into the world-model scan and into shell commands.

## Components that matter most / least
Most important, in order:
1. `scripts/hooks/*.sh` and `scripts/hooks/lib/*.sh` — the enforcement surface. A command shape that performs a write/push/merge/delete but is classified as a read, a quoted-text or heredoc trick that hides a prompt-surface write, a path that escapes the project directory or the worktree scope, or a gate that fails open when it should fail closed, is a real finding.
2. `mcp/trajectory-server/src/` — argument handling (path arguments, repo names, branch ids, SQL built from arguments, FTS queries), every `spawn`/`exec` of `git`, `gh`, `glab`, `node` and `claude` (argument injection via branch names, issue titles, remote URLs), file writes (must stay under the project directory; `.claude/tmb/` and the HF model cache are the only legitimate write locations outside the worktree), and the cheatcode install path (what gets written where, what gets executed).
3. Prompt files `agents/`, `skills/`, `commands/` — only in so far as a hook relies on prompt text for a security property it should enforce in code.

Lower priority / out of scope: `mcp/trajectory-server/dist/` is generated from `src/` (report against `src/`); `tests/`, `docs/`, `benchmarks` and `changelog.d/` are not runtime; `scripts/maintenance/` runs only by maintainers.

## How to exercise it
- Build: `bun install --frozen-lockfile && bun run build` (done in the Dockerfile).
- Full suite: `bash tests/run-all.sh` (L1 lint, L2 unit, L3 hook + server integration, L4 workflow simulation). Run it with an isolated HOME (`HOME=$(mktemp -d) bash tests/run-all.sh`).
- A single hook: pipe a PreToolUse JSON payload into it, e.g.
  `printf '%s' '{"tool_name":"Bash","tool_input":{"command":"git push --force"}}' | bash scripts/hooks/git-guards.sh` — exit code plus a `permissionDecision` JSON on stdout. `tests/l3-integration/hooks/*.test.sh` show the payload shapes for every hook.
- The MCP server: `node --experimental-sqlite mcp/trajectory-server/dist/index.js` speaks MCP over stdio; `tests/l3-integration/mcp/*.mjs` drive it directly.

## How you rate severity
- Critical: command injection or arbitrary file write/exec from agent-controlled or remote-controlled text (issue bodies, PR comments, branch names, cheatcode manifests) reaching a shell or a path outside the project directory.
- High: a bypass of a deny-class hook (git push/merge/force guards, prompt-surface write guard, worktree scope fence, containment guard) by a crafted but realistic command shape; writes outside `CLAUDE_PROJECT_DIR`; a gate that silently allows when its DB or tool dependency is missing where the contract says fail-closed.
- Medium: SQL or FTS injection limited to the local trajectory DB; information disclosure of local project state to the remote (e.g. local ids or paths leaking into pushed artifacts); TOCTOU in the atomic file writes.
- Low: denial of service of the hooks or the local server (the user can always restart Claude Code), resource exhaustion in scans.

## Anything to leave alone
- The hooks intentionally fail open for the SessionStart banner and other advisory (non-deny) hooks; only hooks registered with `onFailure: block` in `hooks/hooks.json` are security gates.
- `TMB_*` environment variables documented as bypasses (`TMB_ALLOW_SOURCE_EDIT`, `TMB_DISABLE_REMOTE_ID_GUARD`) are deliberate operator overrides set at process launch, not vulnerabilities.
- The plugin requires Claude Code ≥ 2.1.295; behaviour under older CLIs that ignore `onFailure` is known and documented.
