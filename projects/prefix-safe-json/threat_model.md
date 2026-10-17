# prefix-safe-json scanner scope

## Project and boundary

prefix-safe-json is a TypeScript library for incremental parsing of streamed JSON
and normalization of LLM tool-call events. Its execution gate distinguishes
complete original arguments from partial, repaired, failed, ambiguous or
unconfirmed input. The application owns tool dispatch and must obtain a fresh
one-use authority with `takeDecision()` before acting on the returned value.

This is a library, not a hosted service. It performs no provider or tool network
requests and does not implement authentication, authorization, sandboxing,
prompt-injection defense or distributed idempotency. See `SECURITY.md`,
`SPEC.md`, `docs/EXECUTION_GATE.md` and `docs/VALIDATION.md` for intended behavior.

## Inputs and assets

Treat JSON strings/bytes and caller-supplied provider-event contents as hostile:
they may contain malformed UTF-8 or JSON, incomplete tokens, duplicate keys,
extreme nesting, adversarial chunk boundaries, contradictory terminal events,
conflicting/reused call identities or late argument changes. Test sequences as
well as individual payloads. Protect argument provenance, call identity,
completion evidence, schema verdicts, one-use dispatch authority, and bounded
CPU/memory consumption under the documented configured limits.

Application configuration, schema/validator functions and the application's tool
implementations are trusted inputs. Arbitrary code execution by a custom
validator or by a tool is not execution by this parser. If the provider/server
hides or falsely replaces terminal evidence, the library cannot recover that
hidden fact from otherwise identical observable input; do not report this
documented limit as a standalone parser bypass.

## Focus paths

Focus on the shipped runtime in `src/`: `parser.ts`, `utf8/`, `lexer/`,
`grammar/`, `semantic/`, `limits.ts`, `validation/`, `providers/`,
`coordinator/`, `gate/`, `guard/`, and the public exports in `src/index.ts`.
Examine prototype-sensitive keys, number/string boundaries, snapshots/event
ownership, state transitions, identity collisions, stream-finalization ordering,
resource accounting and repeated authority consumption.

Tests, `corpus/`, `conformance/` and `examples/` provide intended-use evidence and
regression fixtures. Prioritize vulnerabilities in this project's implementation,
not independent findings in third-party dependencies. `experiments/` contains
separate unpublished experiments and is outside the npm runtime scope. Any local
`artifacts/` research checkout is outside scope and is not part of this repository.

## Findings and fixes

Demonstrate a finding with the exact input/event/chunk sequence, relevant options
and registered schema, expected outcome and actual result. For an execution-gate
bypass, show that `takeDecision()` returns usable authority under supported caller
usage; an observational snapshot alone is not permission to execute. Include a
complete valid control so an always-rejecting adapter is not mistaken for a fix.
For denial of service, report input size, configured limits, measured time/memory
and the violated bound. Distinguish a newly introduced root cause from additional
payloads that exercise an already reported root cause.

Suggest a focused patch plus a deterministic regression test in the relevant
existing suite. Do not modify dependency versions solely to hide an own-code
failure. Do not classify syntactic repairs, caller misuse or disabled limits as
vulnerabilities without demonstrating a violation of a documented guarantee.
Severity should follow demonstrated reachability and application impact; an
unsafe executable verdict under normal documented use is more significant than
a diagnostic-only difference. Do not assume the caller grants arbitrary shell or
database access just because the library can be used with side-effecting tools.

Reports and unpublished reproductions should remain private to the maintainer
contact in `project.yaml`, consistent with `SECURITY.md`.

## Offline build and test commands

The Dockerfile leaves the complete source, installed development dependencies,
compiled package in `dist/`, schemas and fixtures in `/src`. No live model account,
API key or network service is needed for the deterministic tests.

From `/src`, use `pnpm build`, `pnpm typecheck`, `pnpm validate-corpus` and
`pnpm test`. Focused checks include `pnpm test:unit`, `pnpm test:providers`,
`pnpm test:invariants` and `pnpm test:corpus`. The command
`node --test test/scripts/adoption-watch.test.mjs` tests a separate read-only maintainer tool;
it does not need live GitHub access. Do not run publication, outreach, live-provider
trials or release-network verification scripts during an offline runtime audit.
