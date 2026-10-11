# Conduit security audit context

## Project and scope

Conduit is a decentralized commerce platform built on Nostr. This repository
contains the Market buyer application, Merchant seller application, Store
Builder shell, shared TypeScript packages, API functions, and supporting
deployable code. Audit the public code on the current `main` branch.

Start with `SECURITY.md`, `docs/ARCHITECTURE.md`, and the existing contracts in
`docs/specs/`. Protocol-specific context is in
`docs/knowledge/decentralized-network-product-posture.md` and
`docs/knowledge/external-nostr-references.md`. Use implemented code as evidence
when documentation and behavior differ.

## Inputs and trust boundaries

- Treat relay responses, Nostr events and tags, product metadata, media URLs,
  profile data, encrypted messages, and imported data as adversarial inputs.
- A valid signature does not by itself establish authority, freshness, or
  payment settlement. Inspect signer/session identity, event provenance,
  replay handling, deletion, partial relay failure, and durable retry behavior.
- Pay particular attention to `packages/core/src`, app routes and rendering,
  `functions/api`, `apps/anon-zap-signer`, and `apps/posthog-proxy`. Repository
  automation, dependency integrity, and build provenance are also in scope.
- Review signer and wallet boundaries for unauthorized signing or payment,
  credential exposure, account confusion, and unintended persistence. Nostr
  account keys and device-local wallet recovery material are distinct assets.
- Check for script injection, unsafe URL handling, private-data disclosure,
  and telemetry that could expose identity, message, order, or payment data.

## Build and local reproduction

The checkout and installed dependencies are in `/src`. The image includes
Node.js, Bun 1.3.5, Chromium, WebKit, and their system dependencies. Packages and
apps are built with `bun run build:preview`; this does not deploy anything.

From `/src`, use `bun test`, `bun run typecheck`, and `bun run telemetry:check`.
For browser coverage, use `bun run test:e2e`. Tests can start loopback app and
relay servers. Browser binaries are downloaded during the online image build.
Any test that needs a public relay or external provider must use a local fixture
for an offline reproduction. Mock signer or payment tests do not establish live
cryptographic interoperability, relay delivery, or payment settlement.

## Reports and safe testing

Follow `SECURITY.md`. Use synthetic data and local fixtures. Do not access live
accounts, publish harmful events, move mainnet funds, or test private
infrastructure. Private services, credentials, customer data, and operational
runbooks outside this public repository are out of scope.

For each finding, include the scanned commit, affected code path, attacker
preconditions, concrete impact, and a minimal offline reproducer. Explain which
existing check is missing or bypassed. Group findings by root cause and propose
a focused regression test and patch when possible. Distinguish demonstrated
exploits from hypotheses; do not infer settlement from a successful UI state or
infer global event absence from an empty or unavailable relay result.

Severity should reflect demonstrated impact and attacker prerequisites.
Unauthorized signing or payment, credential compromise, and disclosure of
private messages or commerce data deserve particular attention. Severity and
disclosure timing remain subject to maintainer validation under `SECURITY.md`.
