# Threat model

## What this project does and where untrusted input enters
- `@vetojs/core` is an authorization engine: one rule set answers `can()`, compiles to a query condition (`where()`), checks writes field by field and guards tool calls. It has zero dependencies.
- Rules are trusted input (written by the application's developers, optionally validated by `parseRules`). Rows, environment values and request data passed to the engine are untrusted.
- `@vetojs/react` (`<Can>`, `useAbility`) is a UI courtesy only; the server-side check is the control.

## Components that matter most / least
- Most: `packages/core/src` (decision logic in `check/`, query compilation in `compile/`, rule validation in `validate/`).
- Less: `packages/react`. Out of scope: `examples/`, `docs/`, benchmarks.

## How to exercise it
- `packages/core/tests` holds the suite; `conformance.test.ts` checks that `can()` and `where()` select the same rows on messy data.

## How you rate severity
- Critical: an `allow` that grants what the rules do not, or a `deny` that fails to fire, for any row or input shape.
- High: `can()` and `where()` disagree on the same row (a query returns rows `can()` refuses, or the reverse).
- Medium: crashes or unbounded work on malformed rows, rules or conditions.
- Malformed data must fail closed: unknown never grants, and a `deny` still fires.

## Anything to leave alone
- Mutating rows or rule objects after handing them to the engine is the application's responsibility and out of contract.
- A hidden UI element still reachable through the API is not a bug in `@vetojs/react`.
