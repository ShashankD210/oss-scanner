# Threat model

## What Valet does and where untrusted input enters

Valet is a non-custodial Bitcoin and Lightning wallet for Android, written in Scala 2.11. It is a fork of
Simple Bitcoin Wallet (SBW). It supports a special feature: hosted channels whose balance is tied to a
fiat currency through a "ticker" and a satoshi "rate" in the channel state
(https://github.com/standardsats/fiat-channels-rfc, https://github.com/standardsats/hosted-channels-rfc).

The full stack is in this repository, under `app/src/main/java/`:

- `finance/valet/`: the Android app. Activities, SQLite on Android, local backup, Tor.
- `immortan/`: the IMMORTAN Lightning library. Channel state machines (normal and hosted), payment state
  machines, path finding, gossip sync, LNURL, input parsing, encrypted storage.
- `fr/acinq/eclair/`: a fork of eclair. Wire codecs, Sphinx onions, BOLT 11, the router, the Electrum client
  and the on-chain wallet, commitments and transactions.
- `fr/acinq/bitcoin/`: a fork of bitcoin-lib. Scripts, transactions, BIP 32 and BIP 39, PSBT, Bech32, Base58.
- Copies of third-party code: `com/google/zxing` (QR decoding), `com/sparrowwallet/hummingbird` (animated UR
  QR codes), `co/nstant/in/cbor` (CBOR), and small UI widgets.

The wallet keeps the seed, the keys and the channel state on the device. It connects to Lightning peers,
Electrum servers and some HTTPS APIs. The built-in Tor is optional (setting `ensureTor`, off by default).

Untrusted input enters from:

- Lightning peers. Every message after the Noise handshake (`immortan/TransportHandler.scala`,
  `immortan/crypto/Noise.scala`, `fr/acinq/eclair/wire/`): channel open and accept, commitment updates, HTLCs,
  `update_fee`, closing, error messages, and gossip (channel announcements, channel updates, query replies) that
  `immortan/SyncMaster.scala` and `immortan/PathFinder.scala` use.
- Hosts of hosted and fiat channels (`immortan/ChannelHosted.scala`, `immortan/ChannelHostedData.scala`).
  The host can refuse to pay, because that is the hosted channel trust model. But the host must not change the
  client balance, the rate or the capacity without a valid client signature. The host must also not make the
  client sign a state that the client did not agree to, or make the cross-signed state that the client keeps
  as proof invalid. Rate updates, margin and resize proposals, and state overrides come from the host.
- Other nodes on a payment route. Onion failure messages, incoming HTLCs and onions (`immortan/fsm/`,
  `fr/acinq/eclair/crypto/Sphinx.scala`, `fr/acinq/eclair/wire/PaymentOnion.scala`).
- Electrum servers (`fr/acinq/eclair/blockchain/electrum/`). Headers, Merkle proofs, transaction history
  and script hash status. A server can lie. It must not make the wallet accept a fake
  transaction or a fake header chain, lose funds, or crash.
- HTTPS APIs: fee estimates (`immortan/utils/FeeRates.scala`: blockstream.info, mempool.space, bitgo.com) and
  fiat rates (`immortan/utils/FiatRates.scala`: coingecko.com, blockchain.info, bitpay.com). Treat each reply as
  attacker data.
- LNURL servers and Lightning Addresses (`immortan/utils/LNUrl.scala`): LNURL-pay, -withdraw, -channel,
  -hosted-channel and -auth. This includes the invoices that a server returns, the metadata, and the success
  actions (AES-encrypted messages, `immortan/utils/AES.scala`).
- Data that the user scans, pastes or opens (`immortan/utils/InputParser.scala`): QR codes, animated UR codes
  (PSBTs from hardware wallets, `fr/acinq/bitcoin/Psbt.scala`), NFC NDEF records, text that other apps share
  (`ACTION_SEND`), and links with the `lightning:`, `bitcoin:` and `blw:` schemes. This includes BOLT 11 invoices
  (`fr/acinq/eclair/payment/Bolt11Invoice.scala`), BIP 21 URIs (`immortan/utils/uri/`) and node links.
- Other apps on the device. The exported activities `MainActivity` and `HubActivity`, and the `FileProvider` in
  `app/src/main/AndroidManifest.xml` and `app/src/main/res/xml/filepaths.xml`.
- Backup files that the user restores (`finance/valet/utils/LocalBackup.scala`, `immortan/wire/ExtCodecs.scala`).

Trusted: the user, the Android OS and root, the data in the app's private storage, the
graph snapshot in the app assets, and the Tor binary from tor-android.

## Components that matter most / least

Most important:

- Channel state and commitments: `immortan/Channel.scala`, `immortan/ChannelNormal.scala`,
  `immortan/ChannelHosted.scala`, `immortan/ChannelHostedData.scala`, `immortan/ChannelMaster.scala`,
  `fr/acinq/eclair/channel/`, `fr/acinq/eclair/transactions/`. For example: acceptance of a revoked or wrong
  state, wrong HTLC amounts or expiries, a wrong penalty or force-close transaction, fee rates out of bounds.
- Payments: `immortan/fsm/`, `fr/acinq/eclair/payment/`, `fr/acinq/eclair/crypto/Sphinx.scala`.
- Wire and TLV decoding: `fr/acinq/eclair/wire/`, `immortan/wire/ExtCodecs.scala`.
- Keys and cryptography: `immortan/crypto/`, `immortan/LightningNodeKeys.scala`, `immortan/utils/AES.scala`,
  `fr/acinq/eclair/crypto/`, `fr/acinq/bitcoin/DeterministicWallet.scala`, `fr/acinq/bitcoin/MnemonicCode.scala`,
  `fr/acinq/bitcoin/Crypto.scala`.
- The on-chain wallet: `fr/acinq/eclair/blockchain/electrum/` (address derivation, coin selection, change, RBF
  and CPFP), `fr/acinq/bitcoin/Psbt.scala`, `fr/acinq/bitcoin/Script.scala`, `fr/acinq/bitcoin/Transaction.scala`.
- Input parsing: `immortan/utils/InputParser.scala`, `immortan/utils/LNUrl.scala`, `immortan/utils/uri/`.
- Storage: `immortan/sqlite/`, `finance/valet/sqlite/`, if SQL is built from peer or server data.
- App entry points: `finance/valet/MainActivity.scala` (intents), `finance/valet/HubActivity.scala`,
  `finance/valet/utils/LocalBackup.scala`, `finance/valet/TorConnectionProvider.scala`,
  `finance/valet/SettingsActivity.scala` (backup and restore).

Less important:

- The router and gossip sync (`fr/acinq/eclair/router/`, `immortan/PathFinder.scala`, `immortan/SyncMaster.scala`).
  Crashes and resource exhaustion from gossip are in scope. Routes that are not optimal are not.
- The layout code in the activities, and `app/src/main/res/`.
- The copies of third-party code (zxing, hummingbird, cbor, UI widgets). Report a problem in them only if Valet
  gives them attacker data, for example QR decoding.

Out of scope:

- `fastlane/`, `tools/`, the documents (`*.md`), `flake.nix`, `Containerfile`, `sign-apk.sh`, `sign-bundle.sh`,
  and the Gradle tasks for reproducible builds and signing in `app/build.gradle`.
- Maven dependencies (akka, netty, okhttp, bouncycastle, tor-android and others). Report a problem in them only if
  Valet uses them incorrectly, or if you show that Valet reaches the vulnerable code.

## How to exercise it

- The image has JDK 21 for Gradle, JDK 11 at `/usr/lib/jvm/temurin-11-jdk` for the Scala compiler, and the
  Android SDK at `/opt/android-sdk`. All dependencies are in `/root/.gradle`, so use `--offline`.
- `/root/.gradle/gradle.properties` sets the Gradle heap to 4 GB, because the agent machine has 8 GB. Do not
  start more than one Gradle build at a time.
- Build the APK: `./gradlew --offline --no-daemon :app:assembleMainnetDebug`. The APK is in
  `app/build/outputs/apk/mainnet/debug/`.
- Unit tests are JUnit 4 tests in Scala. Put them under `app/src/test/java/`, because this project keeps its
  Scala sources under `src/*/java` (see the comment at the top of `app/build.gradle`). Run them with
  `./gradlew --offline --no-daemon :app:testMainnetDebugUnitTest --tests '<class>'`. They run on the JVM with a
  stub `android.jar`. Code in `immortan/` and `fr/acinq/` that does not call Android APIs works there. Most code
  in `finance/valet/` needs a device.
- Example unit test:

  ```scala
  package immortan.utils

  import org.junit.Assert.assertTrue
  import org.junit.Test

  class InputParserSpec {
    @Test def rejectsGarbage(): Unit =
      assertTrue(scala.util.Try(InputParser.parse("lnbc1garbage")).isFailure)
  }
  ```

- The instrumented tests in `app/src/androidTest/` need an emulator. They do not run in this image, but they
  show how to call the code (for example `GossipQueriesSpec.scala`, `DBSpec.scala`, `JsonSpec.scala`,
  `LnUrlAuthSpec.scala`).
- Most findings can be shown with a unit test that decodes crafted bytes with the codecs in
  `fr/acinq/eclair/wire/`, or that calls the parser, the wallet or a state machine directly with crafted data.

## How you rate severity

Valet holds the money of its users. Use these levels:

- Critical: theft or permanent loss of funds by a remote attacker (a peer, a host, an Electrum server, an LNURL
  server, a payee, or a link or QR code) when the user does only usual actions. Disclosure of the seed or of
  private keys. Remote code execution. Leak or exfiltration of private keys, signature nonce re-use.
- High: loss of funds that needs a less usual setup or a specific user action that looks safe, for example to
  pay an invoice or scan a QR code. A host that changes the client balance, rate or capacity of a hosted or fiat
  channel without a valid client signature. An app on the device that reads the seed or the backup key, or that
  sends a payment without user confirmation. A leak of the IP address of the user when Tor is on. An LNURL-auth
  key that one domain can use for a different domain.
- Medium: a crash or a denial of service caused by remote data that continues after a restart, for example a
  gossip message or an Electrum reply that crashes the app at each start, or a channel that cannot recover.
  Privacy leaks of addresses, node IDs or balances to parties other than those the design expects. Wrong amounts
  on the screen that do not change what the wallet sends.
- Low: a crash that a restart repairs. Problems that need root or physical access to an unlocked device.

A report must name the attacker (peer, host, routing node, Electrum server, HTTPS API, LNURL server, payee,
other app on the device) and show the data that the attacker sends.

## Anything to leave alone

- A host of a hosted channel that refuses to route payment or goes offline. That is the trust model of hosted channels.
- Electrum servers learn the addresses of the wallet and can hide transactions. That is the Electrum model. Report
  it only if the wallet loses funds or accepts false data because of it.
- `minifyEnabled false` and `shrinkResources false` are intentional for reproducible builds. Cleartext traffic to
  `.onion` hosts (`network_security_config.xml`) is intentional, because Tor encrypts it.
- Findings from Android lint, and outdated dependency versions without a reachable vulnerability.
- Patches must compile with Scala 2.11.12 and target Java 8. Keep patches small. Do not upgrade dependencies, and
  do not change the reproducible build tasks in `app/build.gradle`.
