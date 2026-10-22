# Babylon Bitcoin staking

## Project and scan scope

Babylon is a Cosmos SDK / CometBFT blockchain implementing Bitcoin staking
and finality. This enrollment targets babylonlabs-io/babylon, tracking main.
Prioritize the core BTC staking protocol and its integration with chain state:

- btcstaking/: Taproot staking, unbonding and slashing scripts, transactions,
  witnesses and transaction validation.
- x/btcstaking/: finality-provider registration, proof of possession, delegation
  creation and activation, covenant signatures, unbonding, parameter versions,
  inclusion proofs and voting-power accounting.
- x/finality/: EOTS signatures, public randomness commitments, voting, evidence,
  slashing and finalization.
- x/btclightclient/ and x/btccheckpoint/: Bitcoin header validation, fork choice,
  transaction inclusion and confirmation depth; checkpoint consumers.
- crypto/: EOTS, Schnorr adaptor signatures, BIP322, BLS and associated parsing.
- Related incentive, checkpointing, epoching and app integration when reachable
  from these paths and relevant to protocol correctness or consensus.

The wallet SDK, frontend, covenant signer, finality-provider daemon and relayer
are separate repositories and are not included in this enrollment.

## Security goals and trust boundaries

Untrusted users can submit chain messages, Bitcoin transactions and witnesses,
headers, inclusion proofs, keys, signatures and encoded cryptographic data.
Finality providers may act maliciously. Treat these inputs as adversarial even
when the protocol requires valid signatures or fees before accepting them.

Verify that only valid, sufficiently confirmed Bitcoin stakes become active;
that voting power and rewards reflect valid delegation state; and that unbonding,
expiry and slashing transitions cannot be bypassed or inconsistently applied.
Check that staking scripts preserve the intended spending paths and enforce
slashing preconditions, and that malformed or mismatched transactions cannot
produce unspendable or unslashable accepted stakes.

Look for signature forgery, missing domain separation, proof-of-possession
bypasses, adaptor-signature misuse, EOTS extraction failures, invalid inclusion
proofs, incorrect parameter-version selection, reorg inconsistencies, integer
overflow, consensus nondeterminism, and attacker-controlled panics or unbounded
resource consumption in consensus processing.

Evaluate attacks under the documented protocol assumptions. Do not assume the
attacker already controls a Cosmos consensus quorum, a covenant signing quorum,
or Bitcoin consensus. Governance-authorized parameter changes are privileged;
show how an unprivileged attacker reaches a vulnerable state if the exploit
otherwise depends on privileged configuration. Finality-provider double signing
is an expected adversarial behavior that the protocol should detect and punish.

## Impact and reporting

Use SECURITY.md as the project's reporting and severity policy. Prioritize loss
or freezing of funds, accepted stakes that cannot be slashed as intended,
unauthorized voting power or rewards, finality safety violations, consensus
splits and remotely triggered chain halts. Distinguish demonstrated impact from
hypothetical consequences and state any confirmation, quorum, configuration or
attacker-cost assumptions explicitly. This document does not change bounty terms.

Provide a minimal local reproducer, affected functions, expected vs actual
behavior, exploit preconditions, and a proposed patch and regression test where
possible. Keep findings private and send them to the configured security alias.
Do not probe mainnet, public testnets, public RPC endpoints or the frontend.

## Build and local tests

The source lives at /src; the executable is /src/build/babylond. The image retains
Go, C/C++ build tools, module dependencies and debug symbols. The standard
production build uses LEDGER_ENABLED=false and BABYLON_BUILD_OPTIONS=nostrip.

During image construction the staking/crypto smoke tests run, and focused test
binaries are compiled into /opt/babylon-tests. Run those binaries with the working
directory set to their source package directory so relative testdata paths work.
Tests can also be run offline from /src with:

    GOPROXY=off GOSUMDB=off go test -count=1 -timeout=10m ./btcstaking/... ./crypto/...
    GOPROXY=off GOSUMDB=off go test -count=1 -timeout=20m ./x/btcstaking/... ./x/finality/... ./x/btclightclient/... ./x/btccheckpoint/...

Container-based end-to-end suites require additional services and are not part
of this image's smoke tests. Prefer deterministic local unit tests, fuzz tests
with bounded resources, or in-memory keeper tests for reproductions.
