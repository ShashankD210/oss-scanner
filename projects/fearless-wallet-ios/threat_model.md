# Fearless Wallet security scope

Fearless Wallet is a non-custodial multi-chain wallet. Protect mnemonic phrases,
private keys, decrypted wallet material, backup keys and password-derived keys.
Authorization must bind the displayed account, chain, recipient, amount, fee,
transaction or message bytes, and requesting application. Public account identity
must never become signing authority. Cancellation, crashes, storage corruption
and ambiguous partial writes must not silently replace keys or widen authority.

Assume remote RPC/indexer responses, token metadata, chain registries, dApp
requests, deep links, QR contents, imported wallet JSON, encrypted backups, cloud
responses and application-local persisted bytes can be malicious or malformed.
A compromised device OS or malicious replacement browser is outside this model;
attacker-controlled dApps, RPC services and import files are in scope.

Prioritize unauthorized secret disclosure or signing, cross-origin authority
confusion, arbitrary transaction substitution, key loss or rollback during
migration/restore, authentication or AEAD context bypass, and exploitable memory
corruption. Parser denial of service matters when remotely reachable with a
bounded attacker input. Distinguish a reachable defect from a missing runtime
service, deliberately disabled feature or incomplete release qualification.

Use synthetic mnemonics, keys, backup payloads and local fixtures. Do not access
funded wallets, live signing services, production cloud accounts or release keys.
Give an offline reproducer with the precise attacker-controlled input and
expected/actual authorization or secret-handling behavior. Include a regression
test and avoid logging secret material. Dependencies keep their original licenses.

## iOS implementation

Untrusted input enters through dApp/WalletConnect requests under
`fearless/ApplicationLayer/Services/WalletConnect/`, URL import handlers under
`fearless/Common/URLHandling/`, remote chain/RPC data, and imported/recovered
wallet material. Signing, key derivation, secure storage and migration are in
`fearless/Common/Crypto/`, `fearless/Common/Model/`, and
`fearless/Common/Storage/`. Review the `PasskeyBackup*` and `IOSPortableWallet*`
models for byte bounds, canonical encoding, context authentication, secret
lifetime and partial-commit recovery.

Apple Keychain, AuthenticationServices/App Attest, CloudKit, UIKit and the
system browser are platform boundaries. A scanner lacking those services must
not interpret that absence as an authentication bypass or successful device test.
The application's shared dependency pin is in `config/shared-features-source.json`.

## Audit image

The full committed checkout and `.git` are at `/src`. Swift 6.1 is installed;
the separate SwiftPM harness is under `/opt/scanner`. The harness compiles
app-owned portable material codecs, their validation/asset-row helpers and the
Android-source proof parser directly from the source files. It runs all ten
portable semantic material tests and all five Android-source proof tests.

Separate harness copies adapt the tests for Linux: the semantic suite's
`import CryptoKit` becomes Apple's API-compatible `Crypto` module from pinned
`swift-crypto` 3.9.1, and six long proof-fixture array concatenations become
explicitly typed `[[UInt8]]` chunks joined with `flatMap`. The latter avoids
Swift 6.1 type-checker limits while retaining every byte fragment, test and
assertion. The build asserts that exactly six fixture expressions are adapted.
Application files stay byte-identical. `scanner-core-build` refreshes these
copies and rebuilds/tests edited source; `scanner-core-test` reruns the existing
tests offline. SwiftPM source and dependencies are retained in
`/opt/scanner/.build`.

The complete UIKit app cannot be built or launched on Linux. This image does not
pretend to run Xcode/simulator, Keychain, App Attest, CloudKit, the device signer,
or their full test suites. Those implementation paths remain source-review targets.
No private CocoaPods overlay or signing credentials are copied into the image.
