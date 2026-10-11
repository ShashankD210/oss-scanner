# llm-quota threat model

## Scope and assets

This is a small maintained Bun CLI, not a hosted service. No broad adoption claim is made. It discovers local Claude and Codex subscription credentials, probes provider usage/profile endpoints concurrently, and can refresh and persist rotated OAuth tokens. Review `src/sources.ts`, `src/oauth.ts`, `src/claude.ts`, `src/codex.ts`, `src/render.ts`, and argument handling in `src/cli.ts`.

Assets include access and refresh tokens; account, organization and subscription identity; integrity and permissions of credential stores; preservation of valid grants under concurrent refresh; and confidentiality of terminal/JSON output and diagnostics. A lost rotated refresh token can invalidate a login. The CLI is not intended to display tokens, but that intent is a security property to audit, not a blanket guarantee.

## Inputs and trust boundaries

Credential sources are `~/.omp/agent/agent.db` (`auth_credentials`), `~/.claude/.credentials.json`, `~/.claude/cred-profiles/*.json`, and `~/.codex/auth.json`. Treat their JSON, SQLite rows, paths, permissions and possible symlinks as adversarial inputs. Relevant cases include malformed/oversized payloads, ambiguous identities, distinct grants sharing an email, disabled credentials, expiry overflow, and malicious strings reaching terminal output or parser error messages.

Provider responses and error bodies cross a network trust boundary. Audit token redaction in raw and encoded forms, response validation, account matching, bounded requests, and refresh/writeback failure paths. Provider identity must not silently merge distinct subscriptions. SQLite updates use a transaction with a re-read to handle a competing writer; JSON writeback uses a sibling temporary file and rename. These implementations still warrant scrutiny for races, file permissions, unrelated-field preservation and incomplete writes. Other local processes may modify stores concurrently.

The tool has the invoking user's file and network privileges. A same-user attacker already able to read all credentials is not a new sandbox-escape boundary; nevertheless malformed stores, concurrent writers, accidental disclosure, and unsafe file writes are relevant. It does not implement provider authentication, issue tokens, or authorize model requests. Compromise of the OS, Bun itself or the provider is outside this repository's implementation boundary.

## Offline audit environment

The Docker image installs dependencies using the committed `bun.lock` with `--frozen-lockfile` and compiles `/src/dist/llm-quota`, with Bun's SQLite support embedded. Bun 1.4.2, dependency sources, TypeScript, repository sources, and existing tests remain available. There is no project build script beyond its source-based `start` script; the scanner additionally compiles the CLI without executing it. No credentials, provider calls, deployments or token refresh are needed during build.

Run commands only in a disposable container with `--network none` and without mounting a real home directory. `--no-refresh` prevents refresh/writeback but does not prevent usage/profile network probes when accounts exist; it is not an offline switch. Never use real OAuth tokens in fixtures.

For every non-help CLI invocation, set `HOME="$(mktemp -d)"` and populate only synthetic credential fixtures, or explicitly pass `discoverAccounts({home: fixtureDirectory})` in a test harness. Never inspect or use the scanner agent's own `/root/.claude` credentials; avoiding host mounts alone does not isolate that in-container credential store.

Offline commands from `/src`:

```sh
bun test src/oauth.test.ts src/sources.test.ts
bun run src/cli.ts --help
/src/dist/llm-quota --help
bun -e 'import { Database } from "bun:sqlite"; const db = new Database(":memory:"); if (db.query("select 1 as value").get().value !== 1) throw new Error("SQLite smoke failed"); db.close();'
```

The existing OAuth tests inject fake fetch responses and use synthetic tokens. The source tests use in-memory synthetic account fixtures to test deduplication; they do not exercise real credential discovery or SQLite/JSON persistence. Further fixture-based audits may use `discoverAccounts({home: fixtureDirectory})` and the refresh functions' injectable `fetch`, with disposable files/databases only. Do not mistake these tests for complete concurrency, persistence, provider or Windows coverage. Type checking, if desired, is available offline through `bun run node_modules/typescript/bin/tsc --noEmit`.

## Local verification commands

With a separate checkout of the target repository at `/tmp/llm-quota`, run from the scanner registry root:

```sh
docker build -f projects/pedropaulovc-llm-quota/Dockerfile -t oss-scanner-llm-quota /tmp/llm-quota
docker run --rm --network none oss-scanner-llm-quota bash -lc 'bun test src/oauth.test.ts src/sources.test.ts && bun run src/cli.ts --help && /src/dist/llm-quota --help'
docker run --rm --network none oss-scanner-llm-quota bun -e 'import { Database } from "bun:sqlite"; const db = new Database(":memory:"); if (db.query("select 1 as value").get().value !== 1) throw new Error("SQLite smoke failed"); db.close();'
```

These commands are instructions for verification, not claims that a build or tests have already passed. The Linux image does not provide Windows runtime coverage or live provider validation.
