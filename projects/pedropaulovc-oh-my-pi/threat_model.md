# Threat model: pedropaulovc/oh-my-pi maintained fork

## Ownership, distribution, and scan boundary

This enrollment is for Pedro Paulo's maintained MIT-licensed fork at
`https://github.com/pedropaulovc/oh-my-pi`, with reports to
`pedro@vezza.com.br`. It does not enroll `can1357/oh-my-pi`, represent its
maintainers, claim their authorization, or claim upstream adoption of the fork's
changes. The fork has a small dogfood distribution used by its owner; no wider
adoption or critical-infrastructure deployment is asserted.

The selected checkout is the fork's `dogfood` branch, not its default `main`.
`dogfood` integrates upstream plus maintained fork changes and supplies the
independent Linux/Windows `omp-dogfood` release product. On 2026-10-11, the public
release `v18.8.9-dogfood.1` and its tag identified source
`afa7e90580d3777e6c996239b8b48a569b611e00`; assets included
`omp-dogfood-linux-x64`, `omp-dogfood-windows-x64.exe`, and `SHA256SUMS`.
GitHub's comparison to fork `main` at
`4c8f3b6f87e3dcb3294bffd2f40dae64f39be6ef` showed 61 commits ahead and one
behind, with 53 of those commits authored by `pedropaulovc` and eight by
`github-actions[bot]`. The selected branch is moving; these identifiers are
provenance evidence, not a claim that every subsequent scan uses this release.

The default `main` hosts trusted scheduled fork-maintenance and release-workflow
code. Its latest automation revision can differ from the checked-out dogfood
copy. Review the automation and release code present in this checkout, but do not
claim coverage of an absent newer main revision. The full source tree, inherited
runtime, native code, tests, SDKs, and locally built product remain available at
`/src`; this is not an automation-only scaffold.

Ownership evidence and source references:

- [Fork metadata](https://github.com/pedropaulovc/oh-my-pi) identifies
  `pedropaulovc` as owner, `main` as default, and MIT as the license.
- [Pedro's fork-maintenance commit](https://github.com/pedropaulovc/oh-my-pi/commit/8d2c26f04b2978d5749ff9b0dde035f95fc1ca42)
  explicitly maintains fork branches and independent dogfood builds.
- [Published distribution](https://github.com/pedropaulovc/oh-my-pi/releases/tag/v18.8.9-dogfood.1)
  is separate from official upstream/npm releases.
- `scripts/fork-maintenance.ts`, `.github/workflows/fork-maintenance.yml`, and
  `.github/workflows/dogfood-release.yml` describe rebasing, trusted-source
  validation, repository-scoped App permissions, build/publication separation,
  monotonic release revisions, and dispatch. These are supply-chain trust
  boundaries, not permission to invoke them during the audit.

## What it does and where untrusted input enters

Oh My Pi is a terminal AI coding-agent harness. It reads and edits workspaces,
executes shell commands, parses and persists sessions, renders terminal/web
output, communicates with model providers and local/remote tools, and loads
native Rust bindings. A normal user intentionally grants it file and command
access; it is not a sandbox for running arbitrary hostile agent programs.

Untrusted or lower-trust inputs include repository files and filenames, source
and document contents, edit payloads and read snapshots, terminal escape
sequences, subprocess output, model/API responses and streamed tool calls,
session JSONL and stored state, configuration and discovered instructions,
third-party extension/MCP outputs, provider/auth-broker data, collaboration/RPC
messages, and downloaded release metadata/binary bytes. Demonstrate an actual
violated trust boundary rather than treating every tool invocation as a bug.

## Components that matter most / least

Prioritize fork changes and their interactions with inherited functionality:

- Fork release discovery, exact repository/tag/asset validation, checksum/digest
  verification, native embedding/loading, and atomic updater installation.
  `omp-dogfood` must not replace an official `omp` or silently switch to official
  npm/stable/canary update channels. See
  `packages/coding-agent/src/cli/update-cli.ts`,
  `packages/coding-agent/scripts/compile-binary.ts`,
  `scripts/ci-release-build-binaries.ts`, and
  `packages/natives/scripts/embed-native.ts`.
- Fork-maintenance branch planning, stack boundaries, lease-safe pushes,
  source-SHA checks, and separation of read-only builds from publication
  privileges. Malicious refs, dispatch payloads, and release metadata must not
  redirect privileged operations outside the intended fork/source.
- Auth-broker lifecycle, token rejection/cache revalidation, provider response
  handling, and provenance of environment/credential discovery. Use synthetic
  credentials and local transport fixtures only.
- Read/edit provenance and filesystem path handling; asynchronous shell/job
  readiness, progress/wake routing, daemon lifecycle, session handling, and tool
  output attribution. Input must not silently gain a different tool authority,
  modify the wrong file, or leak secrets across sessions.
- Rust/native memory safety and denial of service at N-API, parsers, edit,
  grep/glob, image/document, shell/PTY, and archive interfaces. Relevant trees
  include `crates/pi-natives`, `crates/pi-edit`, `crates/pi-shell`,
  `crates/pi-vfs`, `crates/pi-ast`, `packages/natives`, `packages/ai`,
  `packages/agent`, and `packages/coding-agent`.

Inherited components are not categorically excluded merely because they came
from upstream. Establish reachability in this fork and identify whether a
proposed patch is fork-specific or would also concern upstream. Vendored and
third-party code should be reported when reached through this product, while
clearly attributing the affected dependency. Generated assets/documentation are
lower priority unless they alter runtime security or the release chain.

## How to exercise it safely and offline

The companion Dockerfile installs Bun 1.4.2, the checkout's pinned Rust toolchain,
Bazelisk and the checkout's Bazel toolchain/dependencies, all Bun workspaces,
Cargo dependencies/test tooling, and both Linux x64 shipping native variants.
It compiles the Linux dogfood executable through the actual fork release-build
script. Build and runtime caches stay in the image, not disposable BuildKit cache
mounts. No package publishing, release dispatch, updater installation, live
model request, or real credentials are required.
The image also installs Chromium, the Python SDK/robomp test dependencies, and
the optional fastembed/Transformers/sherpa/Kokoro CPU runtime graphs. Model weights
are input data, not required for the credential-free smoke. Keep the image's
baked dependency caches available when isolating session state; replacing HOME
with an entirely empty directory also hides HOME-relative side-runtime caches.
Every declared lazy WASM grammar is installed using the production size/hash
checks. `PI_NATIVES_DIR=/opt/omp-natives` keeps those grammars and native extraction
independent of HOME. The image's newly compiled dogfood binary is a scan build of
the selected checkout, not a downloaded or published release asset.

From `/src`, using the image's disposable, credential-free HOME:

```sh
packages/coding-agent/binaries/omp-dogfood-linux-x64 --version
packages/coding-agent/binaries/omp-dogfood-linux-x64 --help
packages/coding-agent/binaries/omp-dogfood-linux-x64 --smoke-test
bun run ci:test:smoke
bun test scripts/fork-maintenance.test.ts scripts/ci-release-build-binaries.test.ts
bun test packages/coding-agent/test/update-cli-dogfood.test.ts
CI=1 CARGO_NET_OFFLINE=true bun run test:rs
```

The fork-maintenance test creates disposable local Git repositories with a local
bare origin; importing the script does not run its production entry point. The
updater tests use injected fetch/release fixtures and temporary target files.
The release-build tests exercise dry-run planning and a local compiled probe,
not publication. Keep fixture mutations confined to the container's temporary
paths. `CI=1` is required for `test:rs` so a clean checkout does not skip Rust
work under its change-detection gate; that script invokes nextest and doctests.

For additional native/parser/tool tests, retain the complete checked-out tests
and use local inputs and existing synthetic transports. Real provider services,
remote browser pages, downloads of model weights, audio/video devices, desktop
capture, cloud credentials, and external MCP servers are not offline smoke
prerequisites. Test those interfaces through local fixtures or provide actual
local data; missing credentials/services are not findings. The image is a Linux
x64 build, not a claim of Windows/macOS execution coverage.

## Severity

- Critical/high: executable substitution or arbitrary code execution across a
  lower-trust input boundary, release/build-source substitution under privileged
  automation, reachable native memory corruption with meaningful impact,
  credential exfiltration, or unintended writes outside an authorized target.
- High/medium: bypass of edit/path/provenance restrictions, cross-session data or
  authority confusion, or updater replacement of the wrong product. Rate by
  attacker control, prerequisites, and reachable consequence, not by API name.
- Medium: reproducible resource exhaustion or crashes from plausible untrusted
  inputs that meaningfully disrupt the harness; distinguish bounded large-input
  work and expected cancellation from an unbounded denial of service.
- Low/informational: cosmetic output, ordinary malformed-input errors,
  non-security test flakiness, or behavior requiring pre-existing equivalent
  arbitrary-code access without an additional boundary violation.

An authorized user explicitly asking a shell tool to run a command is expected
behavior. Prompt injection alone is not automatically arbitrary-code execution;
show which enforced authority/provenance boundary is crossed. State affected
fork/source revision, entry point, prerequisites, impact, reproducer, and whether
upstream shares the code.

## Anything to leave alone

Do not dispatch workflows, publish releases/packages, push/rebase real branches,
create real issues/PRs, run `scripts/fork-maintenance.ts`'s production main, run
release/publish scripts, perform live updater replacement, or modify installed
OMP binaries. Do not mount host homes, credential stores, SSH/GPG agents, browser
profiles, Docker sockets, or real workspaces. Never use real model/OAuth/App
credentials or make live provider/telemetry/auth-broker requests. Do not activate
microphones, cameras, capture sessions, remote-desktop/control integrations, or
physical hardware. Use only disposable local repositories, subprocesses,
synthetic tokens, files, pages, and protocol fixtures. These limits restrict live
effects, not inspection of the corresponding source and validation logic.
