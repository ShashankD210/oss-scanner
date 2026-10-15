# Threat model: hoprd

## What this project does and where untrusted input enters

`hoprd` is the node daemon and REST API for the HOPR privacy-preserving mixnet. HOPR provides network-level metadata privacy for point-to-point data transmission by routing packets across multiple hops using Sphinx onion packet encryption. Relay nodes are incentivized via probabilistic micropayment tickets backed by on-chain payment channels.

The scanned branch is `main`. The workspace contains four member crates:

- `hoprd`: The primary daemon binary (`hoprd`), configuration validation tool (`hoprd-cfg`), strategy engine (channel auto-funding, ticket redemption, channel lifecycle), the jemalloc global allocator (default; heap statistics and profiling are opt-in features), telemetry (OpenTelemetry, Prometheus), and process lifecycle management.
- `rest-api` (`hoprd-api`): The Axum HTTP/HTTP2 REST API service exposing management endpoints under `/api/v4/`: account addresses, balances and withdrawal (`POST /account/withdraw` moves funds), payment channels (open, fund, close), network state (ticket price, win probability, connected and announced peers, graph), node version, configuration, info and status, peer info and ping, ticket statistics and redemption, and Session management (creating, listing, adjusting and closing the TCP/UDP listeners that carry Session data; the data itself never passes through the REST API). It also serves OpenAPI documentation (Scalar and Swagger UI).
- `rest-api-client` (`hoprd-api-client`): Progenitor-generated typed Rust client library for interacting with the `hoprd` REST API.
- `localcluster` (`hoprd-localcluster`): Local cluster runner: starts 1 to 5 `hoprd` child processes against a chain (Anvil + Blokli), funds their Safes, opens channels between them, and can inject link latency through per-node UDP relays.

### Untrusted Input Boundaries

Treat the following as attacker-controlled:

**1. P2P Transport Layer (QUIC/UDP and TCP/Noise/Yamux):**
- Every UDP datagram arriving on the P2P listen socket (default `/ip4/0.0.0.0/udp/9091/quic-v1`), including pre-handshake datagrams from arbitrary Internet hosts.
- Every TCP connection on `tcp/9091`, which the node listens on instead of QUIC when `HOPRD_NAT=true` or `DAPPNODE=true` (TCP with Noise and Yamux is always compiled in).
- Everything received from peers during and after the transport handshake: QUIC frames, libp2p Noise/TLS handshakes, Identify messages, AutoNAT responses, and multiaddresses.
- All HOPR mixnet packets: Sphinx onion packets, packet headers, routing tags, MACs, SURBs (Single-Use Reply Blocks), and forwarded application payloads.
- Network routing and peer discovery announcements processed by HOPR discovery and network graph components.
- Micropayment tickets transferred across channels: ticket signatures, challenge hashes, win probabilities, and balance claims.

**2. Mixnet Session Entry and Exit Boundaries:**
- **Entry Listeners:** Application TCP/UDP listeners created via the REST API or session configuration accept arbitrary application data from local or remote clients. This application traffic is separate from administrative API authentication.
- **Exit Forwarding (`hopr-session-server-forwarder`):** Exit nodes resolve session-specified destination targets and establish outbound TCP/UDP connections on behalf of mixnet users. By default in `hopr-session-server-forwarder`, target allowlisting is enabled with an empty list (`use_target_allow_list=true`), preventing all outbound Exit forwarding until destinations are explicitly permitted. Setting `use_target_allow_list=false` intentionally allows arbitrary internet targets; bypassing a configured allowlist is a vulnerability.
- Remote paying peers and mixnet session clients must be treated as untrusted and potentially adversarial.

**3. REST API Service (HTTP / Axum):**
- When enabled, the REST API binds to `127.0.0.1:3001` by default. Authentication defaults to `Auth::None` unless configured. When token authentication is configured, requests must supply a valid token via `X-Auth-Token` or `Authorization: Bearer <token>`. An unauthenticated deployment is not itself a bypass; bypassing a configured token gate is.
- **Unauthenticated endpoints (outside the token gate):** OpenAPI documentation (`/scalar`, `/swagger-ui`, `/api-docs/openapi.json`) and health/readiness probes (`/healthyz`, `/readyz`, `/startedz`, `/eligiblez`).
- **Authenticated endpoints (behind the token gate when configured):** All `/api/v4/` endpoints and the Prometheus metrics route (`/metrics`).
- **Browser origins:** every router sets a `CorsLayer` with `allow_origin(Any)`. With `Auth::None`, any web page loaded by a browser on the node's host can call the API and read its responses, including `POST /account/withdraw`.
- Malformed HTTP requests, invalid chunked encoding, oversized headers or bodies, and unexpected content types.

**4. PIX (Protocol for Incentivizing eXit nodes) Session Settlement:**
PIX (`hopr-protocol-pix`, `hopr-transport-session`, `hopr-strategy`) pays an Exit for the return-path traffic it delivers. For each Session cycle the Entry funds a deposit to a Session Stealth Address (SSA), and the Exit can only recover that address's private key from secret shares carried in the SURBs it spends. PIX is opt-in: the default `hoprd` build compiles no deposit pool, the `Pix` strategy is not in `hopr_default_strategies`, the Entry must request the `UsePIX` Session capability, and an Exit only rejects non-PIX Sessions when `hopr.network.incoming_session_pix.enforce_pix` is set (default `false`). The release Docker image is built with `strategy-pix-curvy`.
- **Session offer & admission (Exit):** The Entry packs `PixParams` (curve `PixSuite`, `polys_per_ssa`, threshold `shares_per_poly`, `surplus_shares`) into one 32-bit word in the Session start `additional_data`; its values come from the Entry's `hopr.network.pix` config (`num_ssa_parts`, `ssa_part_size`, `additional_shares`). The Exit decodes it (`PixParams::try_from_u32`), refuses a mismatched curve suite, checks the derived byte quota (batch size x per-SSA quota while `allow_dynamic_ssa_batches` is on, the default) against `quota_range_min`/`quota_range_max`, and reserves reconstructor memory against `max_live_cycle_bytes` before admitting the Session. Crafted parameter words must not cause panics or allocations beyond that reservation.
- **SSA request & commitment:** The Exit sends an `SsaRequest` for a batch of up to `ssas_per_request` SSAs (hard-clamped to `MAX_SSA_BATCH_SIZE` = 9, at most `MAX_OVERLAPPING_BATCHES` = 2 batches live) carrying its commitment `E = e*G`. The Entry splits a fresh secret `s` additively into `polys_per_ssa` sub-secrets, Shamir-shares each one, and replies with one constant-term commitment `C_j = s_j*G` per polynomial plus a Schnorr proof of knowledge of `s` (`SsaCommitmentProof`). The Exit decodes each point with a prime-order subgroup check, recomputes `sum(C_j)` itself, verifies the proof, and derives the deposit address from `sum(C_j) + E`. The deposit key `e + s` is known to neither side alone; the proof stops an Entry from choosing commitments that cancel `E` and sweeping its own deposit.
- **Deposit & confirmation:** The Entry's `PixStrategy` deposits `price_per_byte * quota`, refused above `max_ssa_allocation` or `max_spend_per_window`. The pool is chosen at compile time: `CurvyDepositPool` (`strategy-pix-curvy`, production; BabyJubJub keys, deposit is a zk allocation inside the Curvy shielded vault) or `NonAnonymousDepositPool` (`strategy-pix-test`, test-only; plain wxHOPR transfer to a secp256k1 address). On the Exit, the pool watches for the deposit (Curvy: Blokli's note index by default, or Curvy's own indexer with `note_source: curvy_indexer`; plain: the address balance) and notifies the Session, whose deposit observer passes it to the supervisor as `DepositConfirmed`.
- **Share transport:** When the Entry builds a SURB with a multi-hop return path, it attaches one share encrypted under the first return relayer's acknowledgement challenge solution (0-hop return paths carry no shares, so the Entry refuses `UsePIX` for them in its own `new_session`; the Exit cannot see the return-path length, so this check does not stop a modified Entry). The Exit extracts the encrypted share when it spends the SURB and can only decrypt it once that relayer's acknowledgement arrives (`acknowledge_shares`). Shares are not verified individually: each reconstructed polynomial is checked once against its commitment (`verify_reconstructed`), and the summed key against `sum(C_j) + E`.
- **Supervision & failure (Exit):** `SessionPixSupervisor` and a `ServiceGate` control egress. Deadlines: `max_ssa_delivery_time` (20 s) and `max_deposit_wait` (60 s), both scaled by batch size, plus `max_recovery_time` (4 h). Anti-free-riding limits: service before a confirmed deposit is capped (`max_predeposit_packets`), and egress blocks after `max_served_without_progress` (2048) packets without share progress. A missed deadline, an unobservable deposit, or unverifiable shares close the Session with `ClosureReason::PixFailure`.
- **Recovery & sweep:** Once every polynomial of a cycle reconstructs, the Exit emits `PrivateKeyRecovered`; `PixStrategy` optionally persists the key (`pix_recovery_db_path`), then sweeps the deposit into the node Safe through the pool (`withdraw_deposit`). Unfinished sweeps are replayed on restart only when both `pix_recovery_db_path` and `pix_recovery_password_env` are set; without that store, a failed sweep loses the key. Exit-originated fill traffic (`fill_enabled`, `drain_after_close`) lets the Exit finish a funded cycle when the application goes quiet, because an unfinished cycle is not refunded.
- **Exit resource bounds:** Each admitted Session reserves `peak_cycle_bytes` (~41 MiB adversarial worst case per cycle at upstream's default 8192 x 64 dimensions, ~7.8 MiB for a conforming Entry) x batch size x `MAX_OVERLAPPING_BATCHES` against `max_live_cycle_bytes` (default 3 GiB). Shares awaiting acknowledgement are capped node-wide by `max_ack_buffer_bytes` (1 GiB), per peer by `max_awaiting_acks`, in the number of peers by `max_tracked_peers`, and per cycle by the deferred-ack caps.
- **Entry exposure:** Entries process Exit-controlled `SsaRequest`s, where one inbound packet can trigger a commitment burst and an on-chain deposit. Bounded by `hopr.network.pix.max_ssas_per_request` (default 2), the successor gate (`min_emission_for_early_recovery`), and `MAX_COMMITMENT_RETRANSMISSIONS` (8).

**5. Blokli Indexer & On-chain Connector:**
- The node synchronizes on-chain state (channel events, ticket redeems, balances) through Blokli via `create_trustful_hopr_blokli_connector`.
- Deserialization of indexer event streams and HTTP responses from the configured Blokli indexer endpoint.

**6. Node Configuration & Local CLI:**
- YAML configuration files parsed at startup via `hoprd --configurationFilePath <path>` or validated via `hoprd-cfg --validate <path>`.
- Environment variable overrides.

## Components that matter most / least

### Most Critical (Tier 1)
1. **Mixnet Packet Processing & Sphinx Cryptography:**
   - Packet decoding and decryption, MAC validation, replay protection (`TagBloomFilter`), tag allocation, mixing, and next-hop forwarding (`hopr-protocol-hopr`, `hopr-crypto-packet`, `hopr-transport`, `hopr-transport-tag-allocator`, `hopr-transport-mixer`, `hopr-transport-session`).
   - Any flaw that allows packet de-anonymization, route tracing, packet tampering without detection, or unauthorized payload reading.
2. **Payment Channels & Ticket Accounting:**
   - Ticket validation (signature, win probability, channel balance: `hopr-crypto-packet`, `validation.rs`), ticket storage and redemption (`hopr-ticket-manager`), and the on-chain calls behind them (`hopr-chain-connector`).
   - Flaws allowing double-spending of tickets, forging ticket signatures, or draining channel balances.
3. **Cryptographic Identity & Keystore Management:**
   - Node identity generation, keystore encryption/decryption, password handling, and private key storage (key loading in `hoprd/src/main.rs`, identity settings in `hoprd/src/config.rs`, the `HoprKeys` keystore in `hopr-types`).
4. **Session Exit Gateway & Target Access Control:**
   - Authorization of Exit connections against configured destination allowlists in `hopr-session-server-forwarder`. SSRF against internal network resources or host loopback via session forwarders.
5. **PIX Settlement (`hopr-protocol-pix`, `hopr-transport-session`, `hopr-strategy` PIX pools):**
   - `PixParams` decoding and admission, SSA commitment and proof verification, deposit address and key derivation, share decryption and polynomial reconstruction, the `SessionPixSupervisor` and `ServiceGate`, and deposit/sweep transactions in `CurvyDepositPool` (production) and `NonAnonymousDepositPool` (test-only).
   - Any flaw that lets an Entry receive Exit service without a confirmed deposit or without delivering shares, lets anyone other than the serving Exit learn an SSA private key, lets an Exit decrypt shares without spending the SURBs and receiving the acknowledgements that release them, or links an Entry to an Exit through Curvy pool deposits and sweeps.

### High Importance (Tier 2)
1. **REST API Core (`hoprd-api`):**
   - Authentication verification (`apiToken`), request validation, error reporting without leaking secret state, and resistance to unauthenticated denial-of-service.
2. **Strategy Engine (`hoprd/src/strategy.rs`, `hopr-strategy`):**
   - Automated funding, channel closure finalization, and ticket redemption strategies.
3. **Network Graph & Routing (`hopr-network-graph`):**
   - Path selection, topology updates, and resistance to route poisoning or eclipse attacks.

### Lower Importance / Out of Scope (Tier 3)
1. **Supporting CLI & Test Tools:**
   - `hoprd-localcluster`: Test cluster orchestration harness.
   - `hoprd-api-schema`: Static OpenAPI generator utility.
   - Profiling scripts under `profiling/` and deployment sample files under `deploy/`.
2. **Documentation & Release Files:**
   - `docs/`, `.github/`, `scripts/`, `flake.nix`, `AGENTS.md`, and markdown files.

### Cross-Repository Hoprnet Dependencies
The HOPR protocol stack is divided across multiple repositories maintained by the HOPR team. Locked dependency versions reachable through `hoprd` are in scope:
- `hoprnet/hoprnet`: Core protocol primitives (`hopr-lib`, `hopr-crypto-packet`, `hopr-protocol-hopr`, `hopr-protocol-session`, `hopr-protocol-app`, `hopr-protocol-start`, `hopr-protocol-pix`, `hopr-transport`, `hopr-transport-mixer`, `hopr-transport-probe`, `hopr-transport-session`, `hopr-transport-tag-allocator`, `hopr-utils-session`).
- `hoprnet/hopr-impls`: Channel, network graph, ticket manager, and forwarder implementations (`hopr-chain-connector`, `hopr-ct-full-network`, `hopr-network-graph`, `hopr-session-server-forwarder`, `hopr-ticket-manager`, `hopr-transport-p2p`).
- `hoprnet/hopr-strategy`: Strategy logic (`hopr-strategy`).
- `hoprnet/hopr-api`: API types (`hopr-api`).
- `hoprnet/hopr-types`: Shared protocol types (`hopr-types`).
- `hoprnet/hopr-utilities`: Shared utility functions (`hopr-utilities`).
- `hoprnet/contracts`: EVM contract bindings (`hopr-bindings`).
- `hoprnet/blokli-client`: Blokli indexer client library (`blokli-client`).
- `hoprnet/babyjubjub-ec`: BabyJubJub elliptic curve implementation (`babyjubjub-ec`). The repository is not public; the source is the crates.io package in the cargo registry.
- `hoprnet/rust-stream-ext-concurrent`: Concurrent stream extensions used by `hopr-transport` (`rust-stream-ext-concurrent`).

Vulnerabilities found in these dependencies reachable via `hoprd` should be reported with clear tracing to the upstream crate and repository.

The `curvy-*` crates (`0xCurvy/rs-sdk`, `0xCurvy/rs-core`: SDK, prover, witness calculation) are third-party and are treated like any other third-party library (see "Anything to leave alone"). How `hopr-strategy`'s Curvy pool uses them is in scope.

## How to exercise it

The image is a checkout of `main` at `/src`. All workspace packages and test binaries are precompiled in the `dev` profile (opt-level 1, debug assertions and symbols enabled). The image has no network access but carries a local chain (Anvil, the HOPR and Curvy contracts, Blokli), so full nodes and clusters run inside it.

### Running Tests
- **Fast unit tests across all workspace crates (runs offline in <1s):**
  ```bash
  cargo nextest run --locked --workspace --lib
  ```
  or
  ```bash
  cargo test --locked --workspace --lib
  ```
- **Precompiled test binaries:**
  Integration test binaries are located under `/src/target/debug/deps/` and can be inspected or run with standard test arguments. Tests marked `#[ignore]` need the local chain (see below).
- **Select tests with `--workspace`:** the precompiled binaries were built for the whole workspace. Selecting a single package (`-p hoprd-localcluster`) changes feature unification and recompiles for several minutes; use `--workspace` with a filter (`-E 'binary(...)'`) instead.

### Exercising Binaries
- **hoprd daemon:** A node connects to Blokli (default `http://localhost:8080`) before it starts P2P and the REST API, so start the local chain first.
  ```bash
  /src/target/debug/hoprd --help
  /src/target/debug/hoprd --version
  ```
- **hoprd with the release feature set:** `/src/target/debug/hoprd-pix-curvy` is built with `-F strategy-pix-curvy`, as the release Docker image is (Curvy pool, BabyJubJub PIX suite). Actual Curvy deposits also need zk proving artifacts, which are not in the image.
- **Configuration tool (`hoprd-cfg`):**
  ```bash
  # Print default YAML configuration:
  /src/target/debug/hoprd-cfg --default

  # Validate a configuration file:
  /src/target/debug/hoprd-cfg --validate /path/to/hoprd.yaml
  ```
- **REST API Schema extractor:**
  ```bash
  /src/target/debug/hoprd-api-schema
  ```
- **Local cluster runner:**
  ```bash
  /src/target/debug/hoprd-localcluster --help
  ```

### Local Chain, Nodes & Clusters
- **Local chain:** `hopr-chain-up` starts Anvil, deploys the HOPR and Curvy contracts, starts Blokli at `$HOPRD_CHAIN_URL` (`http://127.0.0.1:8080`), and returns once Blokli is ready (about 75 s on 2 CPUs). Running it again replaces the chain with a fresh one. The log is `/tmp/hopr-chain/chain.log`.
- **Local cluster (`hoprd-localcluster`):** Starts 1 to 5 `hoprd` processes against the chain (`HOPRD_CHAIN_URL` is set in the image), funds their Safes and opens channels between them. A 2-node cluster is up about a minute after the chain. REST APIs listen on `http://localhost:3000` + node index without a token; `--latency` adds per-link delay; Ctrl+C stops the cluster.
  ```bash
  hopr-chain-up
  /src/target/debug/hoprd-localcluster --size 3 --hoprd-bin /src/target/debug/hoprd
  curl -s http://localhost:3000/api/v4/node/version
  ```
- **PIX build in this image:** `localcluster` depends on `hoprd` with `strategy-pix-test`, and the workspace build unifies features, so `/src/target/debug/hoprd` carries the plain test pool (`NonAnonymousDepositPool`) and the secp256k1 PIX suite, not the production Curvy/BabyJubJub build (that is `hoprd-pix-curvy`). PIX sources are in the fetched hoprnet git checkout (`/usr/local/cargo/git/checkouts/hoprnet-*/6c66ddd/protocols/pix` and `transport/session`); `cargo test -p hopr-protocol-pix` does not work from this workspace because it is a non-member with dev-dependencies.
- **PIX cluster tests:** `localcluster/tests/session_pix.rs` (Session refusal, three-SSA batching, deposit sweep into the Exit Safe) and `session_pix_soak.rs` are `#[ignore]`d. They use the local chain and the prebuilt `hoprd` (`HOPRD_BIN` is set in the image) and must run one at a time. The sweep test passes offline in about 3 minutes on 2 CPUs:
  ```bash
  hopr-chain-up
  cargo nextest run --locked --workspace --run-ignored ignored-only -j 1 --no-capture -E 'binary(session_pix) & test(sweeps_recovered)'
  ```
- **Needs network or a container runtime (does not run here):** `/src/localcluster/scripts/pix-demo.sh` (a terminal dashboard around `session_pix_soak` that starts the chain container) and `curvy-localcluster.sh` (brings up the Curvy stack via docker compose and downloads zk proving artifacts).
- **Larger Local Cluster Testing Suite ([`hoprnet/hopr-integration-tests`](https://github.com/hoprnet/hopr-integration-tests/), not in this image):** For end-to-end integration testing in a larger local cluster, upstream uses the `hopr-integration-tests` framework. Each run brings up a local EVM chain (`anvil` + `bloklid`), deploys the contracts, boots one `hoprd-localcluster` with 2 to 5 nodes in a full-mesh payment channel topology, attaches edge clients (`edgli`), and validates multi-hop traffic, PIX deposit and sweep settlement (test pool), packet delivery guarantees (>=99% arrival), exit origination, and relayer failure recovery under realistic conditions.

## How you rate severity

The workspace sets `panic = "abort"` across dev and release profiles, so any unhandled panic or assertion failure immediately terminates the entire `hoprd` daemon process.

- **Critical**:
  - Remote code execution (RCE) or memory corruption in any daemon or API component.
  - De-anonymization of mixnet traffic: breaking Sphinx packet confidentiality, correlating sender and receiver metadata across hops, or identifying mixnet paths.
  - Direct fund theft or ticket counterfeiting: claiming unearned ticket payments, double-spending micropayment tickets, or extracting channel funds without authorization.
  - Theft of PIX deposits: recovering or sweeping an SSA private key without the shares the protocol releases to the serving Exit (e.g. an Entry choosing commitments that cancel the Exit's `E`, a forged `SsaCommitmentProof`, or decrypting shares without the first-relayer acknowledgement), or `hopr-strategy` pool logic that loses or misdirects deposits.
  - Exposure of node private keys, identity files, or sensitive cryptographic material.
  - Unauthenticated bypass of a configured REST API token gate (`apiToken`).

- **High**:
  - Unauthenticated Denial of Service: crashing (`panic`, abort) or hanging `hoprd` via a single malicious UDP packet, transport handshake message, or unauthenticated HTTP request.
  - Unbounded memory or CPU exhaustion from cheap, unauthenticated network traffic.
  - Replay attacks on Sphinx packets or payment tickets resulting in state corruption or network disruption.
  - PIX free-riding: receiving Exit service beyond the `max_predeposit_packets` / `max_served_without_progress` limits without a confirmed deposit or share progress, or evading `SessionPixSupervisor` deadlines.
  - Exhausting Exit memory or CPU beyond the `max_live_cycle_bytes` / `max_ack_buffer_bytes` budgets via crafted PIX offers, commitments, or shares; or amplifying `SsaRequest`s into Entry commitment bursts or deposits beyond `max_ssas_per_request` and the successor gate.
  - Linking an Entry to an Exit on chain through Curvy pool deposits and sweeps.
  - A protocol flaw that leaves a funded SSA deposit permanently unrecoverable (beyond one party simply ceasing to participate).
  - Bypassing the configured Exit target allowlist in `hopr-session-server-forwarder`, enabling unauthorized egress or SSRF.
  - Cross-origin access from a web page (CORS, DNS rebinding) to the REST API of a node that runs it with `Auth::None`, e.g. triggering `POST /account/withdraw` from a browser on the node's host.
  - Route manipulation or routing table poisoning that enables traffic interception or systematic dropping.

- **Medium**:
  - Denial of service requiring an authenticated API session or active payment channel relationship.
  - Griefing attacks on ticket redemption (e.g. forcing redemption failure right before channel expiration).
  - Information leaks of non-sensitive diagnostic or timing data that does not undermine mixnet anonymity.

- **Low**:
  - Issues requiring a compromised local disk, malicious local configuration file, or privileged local system access.
  - Bugs restricted to test harnesses (`hoprd-localcluster`) or documentation examples.
  - Harmless metric discrepancies or benign diagnostic logging issues.

## How to report

- Provide an offline reproducer that executes against the pre-built binaries or test suite in this container. Anything that needs a running node can use `hopr-chain-up` and `hoprd-localcluster`.
- Specify the attacker position (unauthenticated P2P network peer, compromised relay hop, unauthenticated HTTP client, authenticated API user, or paying session client).
- State any non-default configuration required.
- Identify the owning crate and repository where the root cause resides.
- Propose a minimal patch against `main` accompanied by a regression test where practical.

## Anything to leave alone

- Timing analysis and traffic volume correlations that are intrinsic to mixnets in the absence of cover traffic (HOPR addresses this via cover traffic protocols, not single-packet mechanics).
- Unauthenticated access to metrics or health endpoints (`/healthyz`, `/readyz`, `/startedz`, `/eligiblez`) when authentication is not configured for the node.
- Upstream vulnerabilities in third-party libraries (Rust standard library, Tokio, rustls and aws-lc-rs, ring, libp2p, the `curvy-*` crates) unless `hoprd` misuse explicitly introduces the vulnerability.
