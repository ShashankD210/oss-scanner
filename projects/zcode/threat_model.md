# ZCode threat model and audit guide

## Project and scope

ZCode is an Apache-2.0 AI coding workspace with Electron desktop, browser,
and terminal interfaces. Its agents can read and write files, execute commands,
use plugins and MCP servers, and work in local or remote workspaces.

Audit the first-party code in https://github.com/zai-org/ZCode. The checkout is
available at `/src`. Relevant components include:

- `apps/zcode-cli/packages/core`, `adapters`, `bootstrap`, and `cli`: tool
  execution, permission decisions, workspace hooks, plugins, MCP, sessions,
  and the Agent protocol.
- `packages/server` and `packages/zcode-server-cli`: HTTP/WebSocket interfaces,
  authentication, file serving, remote connections, and deployment.
- `packages/desktop`: Electron Main/Host/preload boundaries, embedded pages,
  IPC, navigation, downloads, and remote-workspace handling.
- `packages/services`, `shared`, `rpc`, and `client`: application services,
  protocol validation, persistence, credentials, and workspace identity.
- `packages/ui`, `web`, and `web-share`: rendering of externally supplied
  content, previews, links, and UI-to-service trust boundaries.
- First-party plugins and build/update/install code included in this repository.

The public checkout is the target. Separately hosted services, private
implementations, other repositories, and vulnerabilities confined to third-party
dependencies are outside this enrollment. First-party misuse of a dependency
remains in scope. The public `packages/zcode-cua` package is an unavailable
placeholder; do not assume it contains the proprietary Computer Use runtime.

## Assets, adversaries, and trust boundaries

Assets include source trees and other accessible files, model/API credentials,
SSH credentials, browser sessions, conversation data, and execution authority
on the local or remote host.

Treat repository files, imported/shared conversations, attachments, Markdown,
HTML, URLs, archive entries, remote responses, model-generated tool arguments,
and external tool output as potentially adversarial. Identify the concrete
attacker-controlled entry point and the permissions needed to reach it.

Prioritize:

1. Unauthorized command execution or file access, including path traversal,
   symlink races, archive extraction, and command-argument injection.
2. Bypasses of configured tool approvals, explicit denials, workspace-hook
   trust, or protocol validation.
3. HTTP/WebSocket authentication or origin-check bypasses and unintended
   access from an untrusted website to local services.
4. Electron renderer, embedded-content, plugin, and preload/IPC boundary
   violations that expose privileged host capabilities.
5. Credential disclosure, unsafe redirects, unsafe downloads or updates,
   and first-party handling of untrusted plugin metadata.
6. Cross-workspace, cross-session, or cross-host routing and data-isolation
   errors. A filesystem path alone is not a remote workspace's identity.

## Intended behavior and deployment assumptions

Read `NOTICE.md`, `README.en.md`, and
`apps/zcode-cli/docs/usage/05-permissions.md` before classifying findings.

The Agent is not an operating-system sandbox. Processes normally execute with
the operating-system user's authority. Workspace directories and worktrees are
not universal filesystem security boundaries. Different permission modes have
different approval rules; a non-interactive CLI prompt defaults to `yolo` unless
`--mode` is supplied. An explicitly permitted command executing successfully is
expected behavior. This does not exclude bypasses of actual approval, denial,
hook-trust, or isolation checks.

Distinguish prompts that persuade a model from deterministic implementation
flaws, while still investigating whether untrusted content can cross a concrete
enforced boundary. Installed executable plugins and configured hooks may
intentionally run code; show any additional authority gained beyond that grant.

Authentication defaults differ by entry point. The unified CLI Web mode binds
to loopback without a token by default and generates a token for non-loopback
binding. The general HTTP server has its own authentication configuration.
Describe the exact entry point, flags, and deployment for each finding. Explicitly
disabling authentication is not by itself a bypass; unintended exposure or a
bypass in an authenticated deployment remains in scope.

## Build and offline reproduction

The Dockerfile installs the versions pinned in `mise.toml`, Linux native-build
dependencies and Electron runtime libraries. It installs the complete workspace,
prepares the desktop Agent/runtime, and builds the workspace clients and backend.
All source and build outputs remain in `/src`. There are no model API keys,
production credentials, or user data in the image.

After the image has been built, use local fixtures, temporary data directories,
mock model/tool servers, and loopback connections. Do not contact production
services or require an external LLM to reproduce a deterministic code defect.
GUI investigations can use `xvfb-run` and `dbus-run-session`. macOS/Windows-only
native behavior cannot be dynamically validated by this Linux image; source
review of those paths is still useful.

From `/src`, useful focused tests are:

```sh
pnpm exec vitest run packages/server/test/staticRoutePathContainment.test.ts packages/server/test/sshAuth.test.ts
pnpm --filter @zcode/core exec vitest run tests/permission-service.test.ts tests/workspace-hook-runtime-admission.test.ts
pnpm --filter @zcode/adapters exec vitest run tests/workspace-hook-trust-store.test.ts
```

Other tests live in `packages/*/test`, alongside module contracts under
`packages/*/src`, and in `apps/zcode-cli/packages/*/tests`. Use each package's
current `package.json` and test configuration. Some integration/E2E tests require
platform-specific tools or real external services; these are not offline smoke
tests. Do not infer that an unexecuted suite passes.

## Reports and severity

Report the vulnerable revision, relevant files/lines, attack prerequisites,
entry point, violated boundary, and a minimal deterministic reproducer using
synthetic data. Include expected versus observed behavior, practical impact,
and a focused proposed patch and regression test when possible.

Rate severity according to demonstrated impact and realistic reachability.
Unauthenticated execution or credential theft through a normal supported
deployment is especially important. Account for required local access,
user approval, installed executable plugins, non-default settings, and the
scope of affected data. Do not classify a hypothetical exploit as demonstrated.

Group manifestations sharing the same root cause. Send reports and build
questions to `vuln_report@zhipuai.cn`, the contact published by the
[ZCode Security Response Center](https://zcode.z.ai/en/zsrc/award).
