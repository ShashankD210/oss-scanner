# Threat model: vultisig-relay

## What this project does and where untrusted input enters
vultisig-relay is the message relay that routes MPC keygen, keysign and reshare messages between the devices (and the vultiserver co-signer) in a Vultisig threshold vault. Clients encrypt protocol messages end to end with a per-session key, so the relay is designed to be untrusted for confidentiality. It must still not let one party read, replace, replay or delete another party's messages, or confuse sessions.

Every HTTP endpoint in `server/handler.go` takes attacker input: session IDs, participant IDs, message hashes, message bodies, payloads (`/payload/:hash`, which is stored under a caller-supplied hash and should be hash-verified) and setup messages. Any internet client can call the relay.

## Components that matter most / least
Most: `server/handler.go` (session, message, payload, setup-message and completion handlers), `storage/` (Redis keys, TTLs, key namespacing between sessions and participants), `model/`.
Less: `config/`, docker-compose files.

## How to exercise it
`redis-server --daemonize yes` and then `vultisig-relay` (the `cmd/router` binary). It serves HTTP; drive it with curl.

## How we rate severity

Vultisig is a self-custodial threshold-signature (MPC) wallet. Every signing path can move user funds and there is no way to reverse an on-chain transaction, so we rate by impact on keys and funds:

- **Critical**: recovery of a private key or of enough key shares to sign; signing something other than what the user approved (wrong destination, amount, chain, contract call, or fee); an unauthenticated party causing a signature or a reshare; remote code execution in any component.
- **High**: disclosure of a single key share, vault backup contents, or a vault password; bypass of a share-encryption step; a malicious peer, relay, or server biasing nonces or extracting secret material from protocol messages; cross-session or cross-vault message confusion that completes a ceremony; theft of the email/verification code flow that gates a server-side share.
- **Medium**: transaction-display bugs that hide or misstate a destination, amount, or fee without changing what is signed; denial of service that blocks a user from signing for longer than a session; information leaks that link vaults, addresses, or devices.
- **Low**: crashes from malformed local input, timing leaks without a demonstrated attack, hardening gaps.

A finding with a working reproducer always ranks above the same class without one. We prefer reports that state the concrete precondition (who the attacker is: dApp, relay operator, network attacker, co-signer, or local malware) and the funds or key material at risk.

## Report format

Please include the exact file and line, a minimal reproducer (a test case in the repo’s own test framework is ideal), and a proposed patch. Patches should be minimal and must not weaken existing validation.

Relay-specific notes: because the payload is end-to-end encrypted, reading ciphertext alone is Medium at most. Injecting, reordering, replaying, or deleting messages so that a ceremony completes with a wrong participant set, or so that one session's messages are delivered to another, is High. A payload stored under a hash that does not match its content, which lets a peer swap a transaction payload, is Critical. Unbounded memory or Redis growth from unauthenticated requests is Medium.

## Anything to leave alone
Rate limiting and DDoS protection happen at the edge (Cloudflare), outside this repository. Volumetric DoS is out of scope; algorithmic or amplification DoS inside the relay is in scope.
