# Threat model: Tracekit

## What this project does and where untrusted input enters

Tracekit records what AI agents do. A signer process (`tracekitd`, or the v2 signer in `tracekit/signer/`) checks each
tool call against a policy before it runs, signs every call and decision into an append-only hash chain (Ed25519,
Merkle checkpoints cosigned by witnesses), and exports a `.tkb` bundle per run that a verifier checks offline. Its one
claim: **it proves what its capture path recorded, and that the record has not changed since it was signed and
checkpointed.** It does not prove intent, complete coverage, or that a reported tool result is real.

Assume the **agent is the attacker**: a prompt-injected or misaligned model acting through its tools, running as the
agent's OS user. Everything it can reach is untrusted:

- Events sent to the signer's socket or RPC by hooks (`tracekit/hook.py`, `agent_hooks.py`), the Python and TypeScript
  SDKs and the framework adapters (`tracekit/adapters/`, `tracekit/integrations/`, `sdk/typescript/`).
- Tool-call arguments evaluated by the policy engines (`tracekit/policy.py`, `tracekit/policy2/`).
- Traffic through the model proxy and gateways (`proxy.py`, `gateway.py`, `mcp_gateway.py`) and spans received by the
  OpenTelemetry receiver (`otel.py`, `otlp.py`, `otlp_wire.py`).
- Harness transcripts read by the tailer (`tailer.py`, `parsers.py`).
- **Bundles, checkpoints and witness responses given to the verifier** (`tracekit/verify/`, `bundle.py`, `bundle_v2.py`,
  `tracekit/format/`, `tracekit/merkle/`, `witness.py`, `tlog_witness.py`, `rekor.py`): the exporter may be hostile.
- Requests to the network-facing servers: observer UI (`tracekit/ui/`, `view.py`), witness servers
  (`witness_server.py`, `public_witness_server.py`), `netserver.py`, approval flows (`slack_approvals.py`).

Full analysis: `docs/threat-model-laptop.md` (attackers A1–A4, E) and `docs/threat-model-server.md` (S1–S6 and
invariants I0–I10). `SECURITY.md` has the scope we publish.

## Components that matter most / least

Most: the verifiers (v1 and v2, Python and TypeScript), canonical JSON (`rfc8785`/JCS handling, I10), the hash chain,
signatures and checkpoint logic, the signer's run registration, caller identity (`peercred.py`, `tracekit/identity/`,
harness binding via `harness_helper.py`), the policy gate and approvals (exact-argument binding, single use), and
redaction (`privacy.py`).

Less: `demo*.py`, `eval/`, `examples/`, `paper/`, `report/`, `contrib/` (in scope, lower priority), deploy templates.

## How to exercise it

- `python -m pytest -q` runs the suite. `tests/INVARIANTS.md` maps tests to invariants; `tests/golden/` holds frozen
  bundles that must keep verifying; `tests/test_hardening.py` and the `test_e*` files hold attack cases.
- `tracekit demo` makes a run and a bundle; `tracekit verify <bundle.tkb>` checks one.
- TypeScript: `cd sdk/typescript && npm test`.
- Tests marked `root` need root (system-mode isolation) and run as root in this image.

## How you rate severity

- **Critical**: a bundle or checkpoint that was forged, edited, truncated or reordered passes verification; an
  agent-reachable process gets the signing key or makes the signer sign as another identity or run; a policy-denied
  call runs with no signed record of it; an approval is reused or applied to different arguments; remote code
  execution in the signer, a server or the verifier from untrusted input.
- **High**: an agent can hide a gap or forge a gap/tamper record (I6); harness binding bypass from outside the harness;
  redaction leaks secrets into a bundle or UI; auth bypass on the observer, witness or approval servers; the verifier
  accepts input its spec says to reject (duplicate keys, non-finite numbers, lone surrogates, out-of-range integers).
- **Medium**: stored XSS in the observer UI; information leaks that do not break integrity; DoS of the signer or a
  server by an unauthenticated remote party.
- **Low**: DoS by a local user who can already stop the signer; crashes on malformed input that fail closed.

## Anything to leave alone

These are documented limits, not vulnerabilities:

- The **operator** (root on the signer host or the `tracekit` service account) holds the key and can sign any history,
  unless checkpoints went to an independent witness first.
- **Dev mode**'s same-user signer: the agent can read the key and ledger by design; every dev-mode bundle says so.
- The signer cannot tell a real event from a well-formed fake sent by the agent's own user; a valid signature means
  "accepted from this caller", not "this happened". Fabrication is caught by cross-checks (e.g. `capture.gap`), when on.
- Policy rules are regex tripwires: rewording a command so no deny rule matches is a known miss (E3), unless it
  bypasses a rule that should match by the engine's own semantics.
- Actions inside subprocesses the agent starts; tool results the harness reports falsely.
- Please send minimal patches with a test in `tests/` that fails before the fix, and a reproducer we can run offline.
