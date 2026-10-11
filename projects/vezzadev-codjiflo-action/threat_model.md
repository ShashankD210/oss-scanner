# CodjiFlo GitHub Action threat model

## Source and dependency provenance

This enrollment targets https://github.com/vezzadev/codjiflo-action. Preparation inspected standalone main `84bf319db970e820224347f6a4784400159696c1`, rather than the stale local checkout. The root `package-lock.json` is committed. Owner-authorized PR #1 has merged a root MIT LICENSE with `Copyright (c) 2026 Pedro Paulo Vezza Campos`; the source main revision after that licensing change is `2b9d7f4`. The scanner enrollment does not change source behavior.

The action is a flattened package at `/src`, but its unmodified build expects `../packages/diff-engine` and `../scripts/generate-schema.cjs`. The Dockerfile fetches public upstream `vezzadev/codjiflo` at immutable commit `0ef8bebd1f336accb171e9c7b79af3bea46227e9` online, retaining the action checkout as the sole `/src` source. This is a compatible dependency pin, not a claimed historical SHA of the standalone mirror's original sync. The pinned engine manifest is v1.0.0 and matches the local-link dependency contract in the action lockfile. The standalone sync commit does not identify its original monorepo SHA.

The linked engine's manifest exports TypeScript source directly; it has no separate build script. The action lockfile records the engine link and manifest but does not contain its resolved `diff-match-patch`/type dependency records. The pinned monorepo lock supplies those dependencies. Its workspace dependencies are installed with `npm ci --workspace=@codjiflo/diff-engine --include-workspace-root=false --ignore-scripts`: the browser app lifecycle is irrelevant to this pure-JS/types workspace. The action itself uses ordinary `npm ci` with lifecycle scripts enabled, including better-sqlite3/node-gyp and esbuild setup. The linked package resolves its dependencies in the retained pinned upstream tree.

The upstream generator is copied unchanged to `/scripts/generate-schema.cjs`; `/action` is a symlink to `/src` solely to satisfy its hard-coded schema paths. `/packages/diff-engine` links to the pinned source. `npm run build` runs the original prebuild generator, then `ncc build src/index.ts -o dist --license licenses.txt`. No action behavior or package scripts have been patched.

## What the project does

The action consumes a repository GitHub token, PR event context or `PR_NUMBER`, fetches PR metadata and changed files through Octokit, and captures base/head file snapshots in a SQLite iteration database. It calculates line/span mappings for force-push-resilient review and edits the PR description to add a link under `https://codjiflo.net`.

`src/index.ts:50-61,86-109,141-160` writes `codjiflo-data/iterations.db` under `GITHUB_WORKSPACE` (or the current directory), reads a previous artifact's ZIP entry `iterations.db`, and emits artifact-name/work-directory outputs. Despite the action step's name and output wording, the implementation does not upload an artifact: upload and comment updates are delegated to a consuming workflow. No such workflow is supplied in this checkout. The integration boundary with a caller's artifact upload is relevant, but an upload implementation must not be invented.

The composite step (`action.yml:29-43`) invokes the committed `dist/bootstrap.js`. The repository ships `dist/index.js`, worker and license files, and genuine better-sqlite3 prebuilds for Node 20/22/24 (ABIs 115/127/137). The bootstrap selects the runner ABI, copies its prebuild into `dist/build/Release/better_sqlite3.node`, and loads the bundle. The Dockerfile preserves the complete shipped `dist/` at `/opt/codjiflo-action-committed-dist` before ncc regenerates `/src/dist/index.js`, so both the source-built bundle and original shipped artifacts remain available for offline audit. Node 22 selects ABI 127.

## Assets and trust boundaries

- Assets include the GitHub token and its repository/actions/PR permissions, runner filesystem integrity, PR-description integrity, and confidentiality/integrity of captured repository source. Private-repository snapshots can become sensitive artifact contents even though the action repository is public.
- PR authors influence file contents, filenames/rename metadata, SHAs and PR bodies. Treat API-returned content as untrusted data, not code to execute. Capture uses paginated `pulls.listFiles` and fetches base/head contents by ref (`src/capture/iteration-capture.ts:75-114`).
- PR comments are inputs to artifact selection. `src/comment/comment-manager.ts:69-113` selects a comment containing a marker and parses its artifact ID without authenticating the comment author. Review whether an unprivileged commenter can steer imported history.
- Repository artifact metadata, ZIP bytes and imported SQLite bytes cross into privileged runner processing. `src/artifact/artifact-download.ts:130-238` first accepts a referenced non-expired artifact, then falls back to an artifact name matching the PR pattern. Assess PR/run/workflow provenance, not merely naming or successful authenticated download.
- `src/index.ts:86-109` extracts one ZIP entry to a fixed database path rather than extracting arbitrary archive filenames. Prioritize decompression/resource bounds and malicious database handling; generic ZIP path traversal requires a demonstrated data flow.
- Review GitHub workflow command/log/output handling, shell interpolation in action metadata, fixed workspace output handling and symlinks, database schema/version handling, SQL bindings, snapshot deduplication, and diff/span-tracking complexity under attacker-chosen large inputs.
- PR-description replacement (`src/comment/comment-manager.ts:146-185`) runs with token write privileges and consumes an attacker-influenced existing body. Preserve unrelated content and prevent unintended cross-PR/repository changes.

## Components and priorities

Prioritize `src/capture/`, `src/artifact/`, `src/comment/`, `src/db/`, `src/spantracker/`, `src/index.ts` and `action.yml`. Relevant dependencies include better-sqlite3, adm-zip, Octokit and the pinned shared diff engine. The pinned helper tree is dependency source, not a second enrolled application: do not report unrelated browser application behavior as an action finding. Dependency findings should demonstrate reachable input paths in this action.

## Offline exercise guidance

The parent reports the scanner build, offline typecheck, all 60 unit tests and ncc rebuild passed. No author-run verification is claimed. All source/dependency downloads happen during the Docker build. The image uses Node 22, Python 3, make and g++ for node-gyp/better-sqlite3; source-native builds use installed Node headers and retain debug information/frame pointers. The action is installed and bundled under `/src`, with node_modules and pinned helper dependencies retained for offline tests. The original shipped bundle/native binaries are retained under `/opt/codjiflo-action-committed-dist`.

In a network-disabled container, with no live GitHub credentials, run the actual non-watching package commands:

```sh
cd /src
npm test -- --run
npm run typecheck
```

Focused offline tests:

```sh
cd /src
npm test -- --run src/artifact/artifact-download.test.ts
npm test -- --run src/comment/comment-manager.test.ts
npm test -- --run src/db/database.test.ts
```

Artifact/comment tests mock Octokit responses. Database tests use temporary local files and real better-sqlite3. Exercise ZIP/SQLite import and capture/span tracking with local fixtures and mocked GitHub API responses. Useful adversarial cases include forged comment markers, mismatched artifact provenance, expired/fallback metadata, oversized archives/files, malformed SQLite/schema versions, unusual paths/renames and resource-intensive diffs.

The bundle can be rebuilt offline with `cd /src && npm run build`: its schema generator, ncc and all dependencies are already installed. Do not reinstall dependencies after network isolation.

Do not set a real `GITHUB_TOKEN`, invoke the action against a real repository/event, upload/download live artifacts, edit live PRs, or deploy anything. A dummy token does not make a real Octokit transport offline: disable network access. A safe production-bootstrap smoke may use a synthetic event with no pull_request and no PR_NUMBER, so the actual entry point takes its documented skip branch before any API call; this verifies JavaScript bundle loading, not PR capture or native SQLite initialization. SQLite is lazy and must be exercised separately with a real Database constructor, local input and the selected ABI binary. Run bootstrap only against a complete owned temporary copy of the chosen dist tree, then remove the copy: normal bootstrap replaces its target native file, including the shipped Win32 ABI137 target with a Linux ABI127 prebuild on Node22. Never let the smoke overwrite the canonical original snapshot or the source-compiled native file in /src/dist. No source rewrites or transport hooks are required.

## Severity guidance

Demonstrated token exfiltration, runner arbitrary code execution, or unauthorized sensitive source disclosure across a trust boundary is high severity, potentially critical depending on reach and permissions. Unauthorized PR mutation or forged iteration history that materially undermines review integrity is a significant integrity issue, with severity tied to demonstrated attacker capability. Report attacker-triggered memory/CPU/disk exhaustion with practical input bounds and impact; exhaustion confined to one disposable action run is generally medium. Ordinary line-mapping correctness differences are not automatically security vulnerabilities without concrete review-integrity impact.

No production adoption or deployment scale is asserted. Parent-reported build/unit-test evidence is separate from a live GitHub workflow test; no live GitHub operation was performed.
