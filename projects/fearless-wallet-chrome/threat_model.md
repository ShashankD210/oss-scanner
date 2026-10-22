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

## Chrome extension implementation

The browser page, content script and background service worker are distinct trust
boundaries. Entry points are under `src/extension/entry/`; request transport and
message types are in `src/extension/messaging/`. The background handlers and
`tabAuthorization.ts` under
`src/extension/background/extension-base/src/background/handlers/` must bind
requests to the real sender origin/tab and the authorized account/chain.
A page-provided URL, origin string or request ID alone is not authority.

Review `services/keyring-service/`, `signers/`, `stores/`, RPC adapters and
WalletConnect/TonConnect handlers under the same `extension-base/src` tree.
Password locking, decrypted-key lifetime, export/restore paths, persisted wallet
JSON and transaction confirmation must agree across popup/background boundaries.
Chain/RPC metadata and a successful connection are not permission to sign.
Check actual manifest permissions and content-script matching generated from
`src/extension/makeManifest.cjs`, rather than assuming every page is trusted.

## Audit image

The full committed checkout and `.git` are at `/src`; dependencies are installed
with immutable Yarn 4.17.0 under Node 24.19.0. `yarn build:extension` builds the
Chrome bundle at `dist/extension/chrome`. The image retains source, node_modules,
Yarn caches and compiler tools for offline reproducers. `scanner-core-test`
runs selected original Vitest suites covering tab authorization, wallet-security
provenance, keyring JSON/upgrade behavior and addresses. Use `yarn vitest run`
for additional suites; the full dependency install remains available offline.

Tests use synthetic keys. No Chrome profile, live wallet, extension-store upload,
release signing or release approval is involved. The production bundle keeps the
repository's default Bitcoin/Iroha transfer release flags; tests and source review
can examine those code paths independently of live transfer qualification.
