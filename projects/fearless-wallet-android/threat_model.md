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

## Android implementation

Entry points include `app/src/main/AndroidManifest.xml`, the account import/export
and recovery flows under `feature-account-impl`, and transaction requests under
`feature-wallet-impl`, `feature-walletconnect-impl` and `feature-tonconnect-impl`.
The in-app WebView bridges in `feature-tonconnect-impl` cross a dApp/native trust
boundary. Review confirmation and account/chain binding before execution.

Key custody and durable failure handling are in
`common/src/main/java/jp/co/soramitsu/common/data/storage/encrypt/` and
`common/src/main/java/jp/co/soramitsu/common/data/secrets/`. Portable wallet-material
parsers and staged receive flows live in
`feature-account-impl/src/main/java/jp/co/soramitsu/account/impl/data/repository/`.
Passkey backup code is in `public-shared-features-backup/`. Android Keystore,
Credential Manager, Google Drive and Play Integrity are external trust boundaries;
source interfaces and error transitions remain in scope even when unavailable.

## Audit image

The complete committed Android checkout and `.git` are at `/src`. JDK 21,
Kotlin 2.1.10, JUnit 4.13.2 and Hamcrest 1.3 are installed. The image builds the
actual portable semantic material and asset-row codec source files and
runs their original JUnit tests. `scanner-core-build` rebuilds these directly
from `/src`; `scanner-core-test` reruns the built tests offline. No implementation
stubs or modified app code are used.

This Linux image intentionally targets the portable parser boundary. It does not
contain an Android APK, emulator, Android SDK/NDK, hardware keystore or a full
Gradle dependency cache. Other Android/UI/native targets require platform setup
and are available for source review here. Full device/backup/release qualification
is separate. Dependency revisions for the full app remain in
`config/android-runtime-source-pins.json`; the harness does not replace those pins.
