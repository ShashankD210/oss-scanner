# GingerWallet threat model

GingerWallet is an open-source, non-custodial Bitcoin desktop wallet for Windows,
macOS and Linux. It manages wallet keys, constructs and signs transactions, and
supports WabiSabi CoinJoin, PayJoin, Tor and hardware-wallet integration. Loss of
funds or disclosure of wallet activity can directly harm its users.

## Assets and security goals

- Protect seed material, private keys, passwords and other wallet secrets from
  disclosure through protocol responses, logs, errors, temporary files or UI.
- Only sign transactions that match the user's authorized recipients, amounts,
  fees and change. Apply the same principle to CoinJoin and PayJoin proposals.
- Preserve privacy boundaries between wallets, addresses, transactions, peers
  and CoinJoin participants. Treat unintended direct connections when Tor is
  required, and disclosure of wallet ownership, as security concerns.
- Preserve wallet files, transaction state and recoverability when handling
  malformed data, cancellation, crashes and concurrent operations.
- Enforce the configured authentication and exposure boundaries of local RPC
  and any security-sensitive wallet or two-factor authentication operations.

## Untrusted input and useful audit targets

Assume that a remote peer, coordinator, PayJoin endpoint, backend or external
provider may be malicious or compromised. HTTPS does not make their application
payloads trustworthy. Review parsing, bounds, resource use, state transitions,
transaction validation and privacy-sensitive networking at these boundaries.

- `WalletWasabi/Blockchain/Keys`, `Crypto`, `Wallets`, `Io` and `Stores`: key and
  secret handling, wallet serialization, persistence, lifecycle and concurrency.
- `WalletWasabi/Blockchain/TransactionBuilding` and `TransactionProcessing`:
  recipients, fees, change ownership, signing and transaction consistency.
- `WalletWasabi/WabiSabi/Client` and `CoinJoin`: hostile coordinator responses,
  round transitions, credential handling and signing conditions.
- `WalletWasabi/WebClients/PayJoin`: attacker-controlled transaction proposals
  and checks preventing unintended spends, fees or privacy loss.
- `WalletWasabi/BitcoinP2p`, `Blockchain/BlockFilters`, `Filter`, `Backend`,
  `WebClients` and `GingerCommon/Serialization`: network messages, filters,
  transactions, API responses and deserialization.
- `WalletWasabi/Tor` and networking clients: proxy routing, isolation, fallback,
  connection reuse and accidental disclosure of addresses or wallet activity.
- `WalletWasabi/Rpc` and `WalletWasabi.Daemon/Rpc`: configuration, request
  handling, method dispatch, authentication and wallet access.
- `WalletWasabi/Services/TwoFactorAuthenticationService.cs`, the corresponding
  helpers, and `WalletWasabi.Fluent` authorization and two-factor flows.
- `WalletWasabi/Hwi`, `Microservices` and UI import/export paths: subprocess
  arguments, device responses, imported files, payment URIs and PSBT handling.

The wallet is not a defense against an attacker who already controls the user's
OS or can freely read the wallet process's memory. Still investigate attacks
that reach secrets or execute code through a less privileged input or interface.
For local RPC findings, state the configuration, binding, authentication and
attacker access needed. For imported files, state the user interaction required.

The primary target is the wallet in this repository. Backend and WabiSabi code
present here may be relevant to wallet safety, but deployed third-party services,
the separate GingerBackend repository and infrastructure are outside this
enrollment. Do not send traffic to production coordinators, mainnet services or
real users. Use local mocks or Bitcoin regtest and synthetic wallet secrets.

Bundled Tor, Bitcoin Core and HWI binaries and NuGet dependencies are external
components. Review their integration here; distinguish a reproducible wallet
integration flaw from a dependency-only advisory without a reachable impact.

## Severity and report quality

Prioritize demonstrated unauthorized spending, seed/private-key disclosure and
code execution through attacker-controlled inputs. Privacy attacks that link a
user or wallet to transactions are also important. Distinguish remote attacks
from attacks requiring enabled RPC, a malicious local application, file import,
physical access or prior host compromise. State each prerequisite explicitly.

For denial of service, show whether the attacker can affect a victim wallet or
service, what resources are consumed, and whether funds or wallet recovery are
at risk. A parsing exception or an existing dependency advisory alone does not
establish high severity. Avoid assuming that ordinary public blockchain data or
the coordinator's intended protocol knowledge is a confidentiality breach.

Please provide a self-contained reproducer, affected code path, attacker
capabilities, concrete fund/privacy impact and a minimal proposed fix. Prefer a
deterministic regression test using synthetic data in `WalletWasabi.Tests`.
Keep secret material out of reports and do not publish unpatched findings.

## Build and offline tests

The Dockerfile keeps the .NET 8 SDK, sources, build outputs and NuGet cache in
the image. The checkout and `WalletWasabi.sln` are at `/src`. Release builds keep
portable debug symbols for analysis. Core, daemon, desktop and test assemblies
are under their respective `bin/Release/net8.0` directories.

Run the existing unit suite without downloading or restoring anything:

```sh
cd /src
dotnet test WalletWasabi.Tests/WalletWasabi.Tests.csproj \
  --no-restore --no-build --configuration Release --filter "UnitTests" \
  --logger "console;verbosity=minimal" --blame-hang --blame-hang-timeout 5m
```

After editing code or adding tests, rebuild with `dotnet build WalletWasabi.sln
--no-restore --configuration Release -p:DebugType=portable -p:DebugSymbols=true`.
The repository also has integration, regression and acceptance tests that may
need local Bitcoin nodes or other services. The Dockerfile runs the unit suite;
use local regtest/mocks when a reproducer needs those additional components.
