# Threat model — roshpinacare-sys/steem (SAOS sovereign fleet engine)

## What this project does and where untrusted input enters
- A trading/custody engine over the Steem blockchain for a multi-account fleet: it reads public chain state over RPC, constructs and signs market orders, and exposes local HTTP services (engine :3040, keeper :3044) plus a Next.js console.
- Untrusted input: public blockchain RPC responses (blocks, open orders, account history), market data, HTTP query parameters on the local services, and state/receipt files influenced by chain data.

## Components that matter most / least
- Most: key handling (signer paths — signing keys are runtime-adopted, memory-only, never persisted), transaction construction and signing, order execution and accounting (real funds), custody verification.
- Least: JSONL receipt writers, loggers, dashboard rendering of already-verified state.

## How to exercise it
- mini-services/saos-engine/selftest.mjs runs the engine self-test; `bun install` + `bun run build` at the repo root build the console; the engine's /health and /army-grid endpoints exercise the read paths.

## How you rate severity
- Critical: any path that leaks key material, signs an unintended transaction, or lets a remote peer trigger signing/broadcast.
- High: order accounting or balance miscalculation, custody-verification bypass.
- Medium: DoS of the local services, incorrect dashboard figures (chain truth persists independently).

## Anything to leave alone
- Test fixtures containing deliberately fake keys; demo/seed state under state/army-grid/pnl.
