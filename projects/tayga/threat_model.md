# Threat Model for TAYGA

## What this project does and where untrusted input enters

TAYGA is a stateless NAT64 (RFC 6146 / RFC 6052) and 464XLAT CLAT daemon for Linux and FreeBSD. It translates IPv4 packets to IPv6 and vice versa using a TUN network interface device.

Untrusted input enters primarily via raw network traffic through the TUN interface:
- **IPv4 and IPv6 packet headers and payloads**: Raw packets received from the TUN driver (`tun.c`, `experimental_io.c`).
- **ICMP / ICMPv6 packets and error payloads**: Embedded IP headers and payloads inside ICMP messages (`nat64.c`).
- **Transport protocol headers**: TCP/UDP checksum calculations and pseudo-header translations.
- **Fragmented packets**: Packet fragmentation, reassembly, and MTU translation paths.

The daemon runs with network privileges (often as `root` or with `CAP_NET_ADMIN` to configure and read from `/dev/net/tun`), so vulnerabilities in packet handling are high-impact.

## Components that matter most / least

### In scope and high priority:
- **Packet Translation Engine (`nat64.c`)**:
  - IPv4-to-IPv6 and IPv6-to-IPv4 header parsing and translation.
  - Length and boundary checking, malformed options, extension headers, and nested ICMP payloads.
  - Checksum calculation routines (incremental RFC 1624 checksums and full checksum updates).
- **Address Mapping (`addrmap.c`, `dynamic.c`)**:
  - Static map lookups and dynamic pool allocation algorithms.
  - Boundary handling, hash collisions, resource exhaustion, and memory leaks.
- **TUN and Packet I/O (`tun.c`, `experimental_io.c`, `gso.c`)**:
  - Buffer bounds when reading from or writing to the TUN interface.
  - GSO (Generic Segmentation Offload) handling and multi-queue handling.

### Lower priority / out of scope:
- **Configuration file parser (`conffile.c`)**:
  - The configuration file (`tayga.conf`) is supplied locally by the system administrator (trusted user). Syntax errors leading to clean daemon termination or exit on startup are expected and not considered security vulnerabilities.
- **Command-line argument parsing (`tayga.c`)**:
  - Arguments passed at startup by the administrator are trusted input.
- **Development & benchmarking scripts (`scripts/`, `benchmark-*.sh`, `tools/`)**:
  - Out of scope for security auditing.

## How to exercise it

- **Unit tests**: `make test` builds and executes unit test suites covering address mapping, checksum calculations, dynamic allocation, GSO, and TUN packet translation.
- **Manual execution / binary**: `./tayga` binary is produced by `make`.

## How you rate severity

- **Critical**:
  - Remote code execution (RCE) triggered by unauthenticated packets arriving on the TUN interface.
  - Arbitrary memory write or controlled heap/stack corruption via malformed IP/ICMP packets.
- **High**:
  - Out-of-bounds read/write causing remote daemon crash (Denial of Service) via network traffic.
  - Memory safety bugs (buffer overflows, integer overflows leading to invalid memory accesses).
  - State corruption in dynamic address pool mapping leading to traffic hijack or cross-tenant packet leakage.
- **Medium**:
  - Uncontrolled memory consumption (memory leaks) triggered by malformed packet sequences.
  - Incorrect translation logic causing silent packet corruption or routing loops.
  - Information disclosure (leaking uninitialized heap or kernel memory into translated packet payloads).
- **Low**:
  - Minor performance degradation under abnormal packet streams where packets are dropped correctly.

## Anything to leave alone

- Do not report issues requiring root access or control over `/etc/tayga.conf` unless they lead to privilege escalation beyond root capabilities (e.g. kernel exploit triggers).
- Daemon exit upon encountering invalid local configuration at startup is expected behavior.
