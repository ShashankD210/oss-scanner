# Threat model: Nothing But Data (NBD)

Read by Anthropic's OSS Scanner before it scans the NBD repository. Reports go to the
`primary_contact` of the enrolment; do not open public issues for findings (see `SECURITY.md`).
This file is public: it states intent and scope, it is not a list of known unpatched issues.

## What NBD is, and why it matters

NBD (binary `nbd`, Rust 1.97.1 pinned, `#![forbid(unsafe_code)]`, ANSSI Secure Rust guide) is a
single-process daemon that joins UDP multicast groups and relays every datagram, unmodified,
to one Apache Kafka broker through `rdkafka` (bundled `librdkafka`, C). Each multicast group
(a "provider") maps to one Kafka topic; the topic name comes from the configuration, never
from packet content. It targets unattended, security-sensitive production deployments with
sub-millisecond latency (systemd `DynamicUser`, empty capability set). A bug in NBD can drop,
corrupt, misroute or expose data flowing between a multicast segment and a Kafka cluster, or
take the relay down, so availability and integrity matter as much as confidentiality, and so does not harming the
networks it is attached to.

## Where untrusted input enters

1. **Multicast datagrams** (`src/providers.rs`): any host able to send to a subscribed group,
   with spoofable source addresses. Payload is fully attacker-controlled, 0 to 65507 bytes, at
   any rate the network allows. In production the payload is opaque: it is received into a
   `BytesMut`, split off with `split_to().freeze()` and handed to the sink without being parsed.
2. **The Kafka path** (`src/sinks.rs`, `librdkafka`): a network attacker when TLS is not
   configured, or a malicious or compromised broker. Broker responses are parsed by `librdkafka`.
3. **The metrics HTTP listener** (feature `metrics-exporter`, `src/main.rs`): any host that can
   reach `[metrics].interface:port`. Unauthenticated plain HTTP.
4. **The local host** (unprivileged local user): files, environment, `/proc`, logs, core dumps.
   This is where Kafka credentials, TLS key passwords and OAuth secrets could leak.
5. **The configuration file and CLI** (`src/config.rs`, `src/args.rs`): written by the operator,
   so trusted for intent but not for well-formedness. Secrets can be literal, `{ env = "VAR" }`
   or `{ file = "/path" }` (`SecretSource`); files holding secrets must be mode `0600` on Unix.
6. **Supply chain**: crates (`Cargo.lock`), `build.rs`, `deny.toml`, `.github/workflows/*`
   (release pipeline included), published release binaries.

## By design, not findings

- Multicast has no authentication or integrity. NBD forwards what arrives; protection is network
  segmentation (see `SECURITY.md`). Whether a crafted datagram can do anything *beyond* being
  forwarded verbatim (crash, hang, unbounded resource use, corrupt or misroute other messages)
  is in scope.
- With no `[kafka.tls]` / `[kafka.auth]` section the Kafka connection is plaintext, as
  documented. A configuration that requests TLS or SASL and silently ends up without it, or a
  weaker protocol than requested, is in scope.
- The metrics endpoint is unauthenticated and unencrypted by design; deployments restrict it at
  network level. What it exposes beyond documented metric names and labels is in scope.
- Packet loss under sustained very high rates is a documented hardware/OS ceiling
  (`PERFORMANCE.md`), and the metrics exporter has a measurable cost. Loss triggered by a few
  crafted packets is in scope.
- `BenchSink`, `IPerfSink` (`src/sinks.rs`) and everything under `tests/` are test and benchmark
  code; the production binary only uses the Kafka sink. Issues reachable only from test code are
  informational.
- Known advisories in third-party crates already surfaced by `cargo audit` / `cargo deny` in CI:
  report them only if NBD's usage makes them exploitable.
- Out of scope: `images/`, documentation, the ASCII art in `src/about.rs`. Windows is a
  supported build target but Linux is the deployment target; report Windows-only issues at lower
  priority unless the security impact differs.

## Impact on the multicast ("in") network

NBD is meant to be a passive receiver on the segment that carries the multicast feeds. That
network may be shared with other consumers and with time-critical flows, so NBD must never
degrade it: a bug must not turn the daemon into a source of flooding, storms or control-plane
churn. A bug has few levers here, which makes the question tractable.

What NBD may do on the in-network, and nothing else:

- Request group membership. The kernel emits the IGMP/MLD reports when a socket joins a group;
  NBD opens plain UDP sockets only (`socket2`, no raw sockets, no `unsafe`, empty capability
  set in the documented systemd unit), so it cannot forge or craft control-plane packets. What
  a bug can influence is which group, on which interface, how often and for how long.
- Receive. Multicast has no flow control: a slow, stalled or crashed NBD cannot slow the
  senders, it only loses packets locally. NBD must never send anything towards the in-network
  (no acknowledgement, no retransmission request, no republishing into a group, no forwarding
  between the two segments).

What a bug could trigger (in scope; look for code paths and configurations that lead here):

1. **Joining the wrong thing**: a wrong group, or the right group on the wrong interface (for
   example a kernel-chosen default interface used because the configured one is missing,
   invalid or of the wrong IP family). Joins and the feed itself would then be pulled onto a
   network that was not meant to carry them (such as the Kafka-facing one), extending IGMP
   snooping and PIM state and consuming bandwidth there. Silent fallbacks and silent
   "corrections" of configuration values are in scope: a critical deployment expects an invalid
   configuration to stop the daemon at start-up, not to be repaired quietly.
2. **Membership churn**: repeated join/leave cycles caused by process or task restart loops, a
   crash triggered by crafted datagrams (see "Where untrusted input enters", item 1) followed by
   a supervisor restart, retries without backoff, or many providers restarting together. Each
   cycle produces IGMP/MLD reports and leaves, snooping table updates, group-specific queries
   and PIM join/prune activity. With the documented unit the rate is bounded by `RestartSec`
   (5 s) per provider; other supervisors or settings change that bound. A remotely triggerable
   crash is therefore also a network finding: say so in the report.
3. **Transmitting onto the in-network**: any datagram leaving through the interface attached to
   the multicast segment other than membership reports, whether sent by NBD or by its
   dependencies (Kafka client, DNS, OAuth or Kerberos endpoints, metrics) because of routing,
   bind or interface choices on a multi-homed host; and anything that could feed data back into
   a subscribed group and create a loop.
4. **Blurred boundary between the two networks**: NBD is the only bridge between the multicast
   segment and the Kafka side. Which datagrams does each provider socket actually accept (bind
   address, device binding, group filtering, a port shared by several providers with
   `SO_REUSEPORT`)? Can traffic that did not arrive as multicast on the intended interface end
   up in a topic? Which interface does the metrics listener bind to?
5. **Memberships held while NBD is not consuming** (a wedged provider task) keep the feed
   flowing to the host for nothing. Low impact, but report it if a hang is reachable.

Out of scope: how switches and routers treat the traffic (IGMP snooping design, storm control,
PIM rendezvous point placement, port ACLs, IGMP limits). Those are the network operator's
controls. Report what NBD itself does that makes such controls necessary, ineffective or easy
to trigger, and assume NBD must not rely on them.

## Where to look first

1. `src/providers.rs`: socket setup (`socket2`, IGMP/MLD join, buffer sizing), the receive loop,
   zero-copy buffer handling (`BytesMut::split_to`, `freeze`, `reserve`), empty and oversized
   datagrams, in-flight delivery tracking, cancellation and draining, behaviour on persistent
   socket errors, and which datagrams each socket accepts.
2. `src/config.rs` (largest file): two-stage TOML parse and validation, `SecretSource`
   resolution, `check_secrets` / file-permission enforcement, automatic `security.protocol`
   selection, TLS / SASL / Kerberos / OAuth handling, DNS resolution of the broker, and how
   `group`, `interface` and `port` are validated, defaulted or silently replaced.
3. `src/sinks.rs` (Kafka sink) and `src/main.rs`: startup order, graceful shutdown, panic hook,
   logging, metrics exporter start-up.
4. How NBD configures `librdkafka` (property names and values, defaults, certificate and hostname
   verification, secret lifetime in memory).
5. `.github/workflows/*`, `build.rs`, `deny.toml`: workflow injection from inputs, refs or
   tag names, unpinned actions, privileged triggers, reproducibility.

Questions worth answering with a reproducer rather than by reading alone:

- Can one datagram, or a short burst, crash, panic, hang or wedge a provider task, or make the
  daemon exit or stop draining? Does a failure in one provider affect the others?
- Can small datagrams pin large buffers or make memory grow without bound? Is behaviour correct
  for 0-byte, 1-byte, exactly-`message_size`, larger-than-`message_size` and 65507-byte datagrams?
- Are forwarded bytes always identical to received bytes (no truncation, reuse or overlap)?
- Under broker slowness or outage, is backpressure bounded, are drops counted, does shutdown
  still complete?
- Can any secret reach logs, error messages, panic output, metrics, `Debug` output or the
  process environment unnecessarily? Can the `0600` check be bypassed or raced (symlinks,
  ownership, time of check vs time of use)?
- Can configuration produce a weaker Kafka security setup than the operator asked for, for
  example when a section is present but its cargo feature is not compiled in?
- Can the configuration end up with a membership on a different interface, IP family, group or
  scope than the one requested, without the daemon refusing to start?
- Does a start, crash, restart cycle stay at one join and one leave per group? Does a
  persistent socket error make a provider loop without backoff or re-join repeatedly?
- Does NBD ever transmit a datagram on the in-network side, and which datagrams can each
  provider socket receive besides its own group on its own interface?
- Does every cargo feature combination compile and behave safely (default, `metrics-exporter`,
  `kafka-tls`, `kafka-auth-gssapi`, `kafka-auth-oauth`, all together)?

## Building and exercising the project (offline)

The image already contains the checkout in `/src`, all dependencies (`cargo fetch`), debug and
release builds with `--all-features`, compiled tests, `cargo-audit` and `cargo-deny` with their
advisory databases, `cargo-fuzz` with a nightly toolchain, debuggers (`gdb`, `valgrind`,
`strace`), `tcpdump`, `iperf` / `iperf3`, `socat` and Python 3. Cargo is set to offline mode.

```sh
cd /src
cargo test --locked --all-features --lib
cargo test --locked --all-features --test config          # configuration parsing and validation
cargo test --locked --all-features --test simple_latency  # quick end-to-end smoke test
# stress, spike, multi_channel_stress and multi_channel_iperf_stress run for minutes and need
# iperf: do not run them unless a finding requires it.

cargo audit --no-fetch ; cargo deny --offline check       # dependency advisories, licences, bans
cargo +nightly fuzz init                                  # e.g. a target on config parsing
```

Running the daemon end to end on loopback (no Kafka broker is present; deliveries fail by
design, which is useful to exercise the receive path, backpressure and shutdown):

```sh
ip link set lo multicast on; ip route add 224.0.0.0/4 dev lo   # if the lab needs it (root)
./target/debug/nbd --check-config-file /opt/nbd-lab/config.toml
./target/debug/nbd --config-file /opt/nbd-lab/config.toml &
python3 /opt/nbd-lab/send_multicast.py --count 1000 --size 1400
python3 /opt/nbd-lab/send_multicast.py --size 0
curl -s http://127.0.0.1:9000/metrics | head
```

Network footprint of the daemon (IGMP is not emitted on loopback, so observe the system calls;
with a dummy or veth interface and `tcpdump -ni <if> igmp` you can also observe the reports):

```sh
strace -f -e trace=network -o /tmp/nbd.strace ./target/debug/nbd --config-file /opt/nbd-lab/config.toml
grep -E 'ADD_MEMBERSHIP|DROP_MEMBERSHIP' /tmp/nbd.strace   # which group, which interface, how often
grep -E 'sendto|sendmsg' /tmp/nbd.strace                   # UDP sockets should never transmit
cat /proc/net/igmp /proc/net/igmp6; ip maddr show           # memberships currently held
```

`/opt/nbd-lab/config.toml` (created by the image) is a minimal valid configuration; the Kafka section of
`README.md` documents every parameter.

## How to rate severity

- **Critical**: remote code execution or memory corruption reachable from a multicast datagram,
  Kafka broker traffic or the metrics port; disclosure of Kafka credentials, TLS key passwords or
  OAuth secrets to an unprivileged remote party; TLS or SASL silently not applied although
  configured.
- **High**: a single datagram or a short burst (no sustained flood) that crashes, aborts, hangs or
  permanently wedges the production binary or one of its providers; payload corruption,
  truncation, cross-message leakage or delivery to the wrong topic caused by crafted traffic;
  secrets exposed to a local unprivileged user or written to logs, metrics or panic output;
  bypass or race of the secret-file permission check; script injection in a release or CI
  workflow reachable without maintainer privileges; a code path or configuration handling that,
  without any operator mistake, makes NBD join a group or interface it was not configured for,
  transmit onto the in-network, or churn memberships at a rate not bounded by a supervisor delay;
  a remotely triggerable crash or restart cycle (also when its only visible effect is membership
  churn on the in-network).
- **Medium**: resource exhaustion that needs sustained traffic from a host already on the
  multicast segment; log injection or forging; metric label or cardinality abuse; secrets kept in
  memory or process environment longer than necessary; workflow injection that needs repository
  write access; hardening gaps with a concrete exploitation path; fail-open configuration
  handling that silently redirects or broadens membership after an operator typo (wrong
  interface, IP family, group or scope).
- **Low**: panics or unwraps reachable only from operator-supplied malformed configuration or
  CLI arguments; memberships held by a wedged but otherwise harmless provider; documentation
  inconsistencies with a security flavour; issues only in test and
  benchmark code; missing hardening without an exploitation path.

Without a demonstrated trigger, cap memory-safety suspicions in pure-safe NBD code at Medium and
say clearly what is missing; for `librdkafka` and other dependencies, report what NBD's
configuration makes reachable and say whether the fix belongs upstream.

## What a good report looks like

- One finding per root cause; deduplicate aggressively and list every affected code path.
- A reproducer that runs offline in this image (a script using `send_multicast.py`, a `tests/`
  case or a fuzz target), with the exact command, expected result and observed result.
- The affected file and lines, the root cause, and the severity from the rubric above with a
  one-line justification.
- A **minimal** patch that demonstrates the fix, plus a clear explanation of why it is correct.
  The maintainer prefers to write the final fix personally, so rationale matters more than
  polish; please do not refactor around the fix.
- Patch constraints: no `unsafe`; no `unwrap`/`expect`/unchecked indexing in production paths;
  errors go through `NbdError`; the per-packet hot path must not gain allocations, locks or
  syscalls (check with `tests/stress.rs` and `tests/spike.rs` if the patch touches
  `providers.rs` or `sinks.rs`); `cargo fmt` and `cargo clippy --all-targets --all-features -- -D warnings`
  must stay clean.
