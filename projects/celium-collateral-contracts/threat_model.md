# Threat model

## What this project does

`src/Collateral.sol` is a collateral contract for a Bittensor subnet. It runs on the Bittensor EVM. The native coin is TAO.
Miners lock TAO in it for each executor (a GPU machine, keyed by a `bytes16` UUID). The validator can slash that TAO.
One contract is deployed per validator and subnet.

`celium_collateral_contracts/` is a Python library and set of CLI scripts. Miners and validators use it to call the contract.

## Roles

- **Miner.** Any address. The first `deposit(executorId)` for an executor makes the caller its owner.
  Only the owner can deposit more for that executor or start a reclaim.
- **Trustee (the validator).** One address, set in the constructor. It cannot change.
  Only the trustee can call `slashCollateral` and `denyReclaimRequest`.
- **Anyone.** Can call `finalizeReclaim` after the deny window ends. The TAO always goes to the miner stored in the reclaim.
- **Owner / admin.** There is none. The contract has no owner, no pause, no setters and no upgrade path.
  Constructor values (`NETUID`, `TRUSTEE`, `BURN_ADDRESS`, `MIN_COLLATERAL_INCREASE`, `DECISION_TIMEOUT`) are fixed after deploy.
  `deployUUPS.js` is an old proxy script. It does not match the current contract and is out of scope.

## Funds flow

1. `deposit` adds `msg.value` to `collaterals[executorId]`. `receive` and `fallback` revert.
2. `reclaimCollateral` locks all collateral of the executor that is not already in a pending reclaim. It starts a timer of `DECISION_TIMEOUT` seconds.
3. Before the timer ends, the trustee can `denyReclaimRequest`. The collateral stays in the contract.
4. After the timer ends, `finalizeReclaim` sends the amount to the miner. If a slash made the collateral too small, it reverts.
5. At any time, `slashCollateral` sends up to the full collateral of an executor to `BURN_ADDRESS`.

TAO must leave the contract only through steps 4 and 5.

## Where untrusted input enters

- Every external call from a non-trustee address: `deposit`, `reclaimCollateral`, `finalizeReclaim`, and direct sends to the contract.
  Miners are untrusted. A miner can be a contract with a hostile `receive` (reentrancy, revert, gas grief).
- `executorId`, `url`, `urlContentMd5Checksum` and `reclaimRequestId` are caller-controlled.
- Python tooling: chain data (events, RPC replies) and the content at `url`, which `calculate_md5_checksum` downloads.
  Treat both as untrusted.

## How you rate severity

- **Critical:**
  - Theft of collateral: any path that sends TAO to someone other than the executor's owner or `BURN_ADDRESS`.
  - A miner reclaims more TAO than they deposited, or reclaims TAO that belongs to another executor or miner.
  - A non-trustee address slashes, denies, or otherwise reduces another miner's collateral.
  - A miner takes ownership of an executor that another miner already owns while it holds collateral.
  - Collateral locked forever with no path out, caused by an outside party, not by the trustee.
- **High:**
  - Accounting drift between `collaterals`, pending reclaims and the real contract balance that can lead to a Critical later.
  - A miner escapes a slash, for example by finalizing a reclaim with TAO that should have been slashable.
  - A miner blocks the trustee from slashing or denying (revert or gas grief on the trustee path).
- **Medium:** a miner blocks only their own reclaim or another miner's reclaim for a limited time.
  Wrong events or wrong values in events that off-chain tools trust.
- **Low:** gas waste, view function errors, and bugs in the Python CLI that need the user's own bad input.
- Python tooling: a bug that leaks a private key, or that signs a transaction other than the one the user asked for, is High.
  Code execution from chain data or URL content is Critical.

## Trusted by design (not a vulnerability)

- The trustee can slash any amount of any executor at any time, with no proof. This is intended.
- The trustee can deny every reclaim, so a miner's TAO can stay locked while the trustee keeps denying. This is intended.
- The deployer chooses the constructor values. Bad values chosen at deploy time are not a bug.
- `url` and `urlContentMd5Checksum` are not checked on-chain. They are only a record for humans.
- MD5 is used only as a content tag, not for security.

## How to exercise it

- `forge test -vv` from `/src` runs the unit tests (`test/Collateral.t.sol`) and the fuzz tests (`test/CollateralFuzz.t.sol`).
  `test/CollateralTestBase.sol` has the setup. Add a new test there to show an exploit.
- `anvil` is installed. You can deploy the contract locally with `forge create`.
- `pytest unittests/test_address_conversion.py` runs the offline Python tests.
  `unittests/test_collateral_flow.py` needs a live subtensor node and `btcli`. It cannot run offline.

## In scope / out of scope

- In scope: `src/Collateral.sol` (main target), and `celium_collateral_contracts/` (secondary).
- Out of scope: `libs/` (forge-std and OpenZeppelin), `deployUUPS.js`, `deploy*.sh`, `build.sh`, `test/`, `unittests/`.
- Out of scope: bugs in the Bittensor EVM, in Subtensor, or in solc.
- Do not report: floating pragma, missing NatSpec, style, or gas tips without a security impact.

## How reports should look

Give the attacker's role, the call sequence, and a Foundry test that shows the TAO loss or wrong state.
A proposed patch is welcome.
