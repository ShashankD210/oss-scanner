# Threat model

## What this project does and where untrusted input enters
httpjet is an experimental Linux web server in Rust. It reads LiteSpeed-compatible XML configuration, speaks LSAPI to
`lsphp` workers, and supports HTTP/1.1, HTTP/2 (native `hj-h2`), HTTP/3 (quinn on io_uring), TLS (rustls), static files,
reverse proxying, rewrites, compression and an opt-in page cache. The binary is `crates/httpjet`; the `hj-*` crates are its
components.

Treat as adversarial anything an unauthenticated network client controls: raw bytes on the HTTP/1, HTTP/2 and HTTP/3
(QUIC) listeners, TLS handshake data, request headers, paths and query strings, request bodies (including
gzip/Brotli/zstd request decompression), range requests, and responses fetched from proxied upstreams.

## Components that matter most / least
- Most: protocol parsers and state machines (`hj-http`, `hj-h2`, HTTP/3 in `crates/httpjet`), `hj-tls`, `hj-static`
  (path resolution), `hj-rewrite`, `hj-acl`, `hj-proxy`, `hj-compress`, `hj-cache` / `hj-pagecache`, `hj-lsapi`,
  `hj-fastcgi`.
- Less: `hj-config` (XML configuration is trusted administrator input), `hj-log`, optional features `acme`, `ocsp`,
  `otel`, `profiling`, `ktls`.
- Out of scope: `vendor/` (third-party code; report only bugs in httpjet's own use of it or in httpjet-specific
  patches), `fuzz/` harness code, `examples/` and `packaging/`.

## How to exercise it
`cargo build --release -p httpjet`, then
`target/release/httpjet --root "$PWD/examples/litespeed" serve --http-addr 127.0.0.1:8080 --https-addr "" --workers 1 --no-php`
and send traffic with curl or raw sockets. It needs io_uring; if the sandbox blocks it, exercise the parser crates
(`hj-http`, `hj-h2`, `hj-lsapi`) directly with unit tests or the targets in `fuzz/fuzz_targets` (nightly + cargo-fuzz).

## How you rate severity
- Memory-unsafety (out-of-bounds, use-after-free, double free) reachable from network input: high at minimum; critical
  if it plausibly gives remote code execution.
- Request smuggling or desync, HTTP/2 or HTTP/3 state-machine bugs that corrupt another client's stream, ACL or
  authentication bypass, path traversal out of the document root, TLS certificate or validation bypass: high.
- Release builds use `panic = "abort"`, so a panic reachable by an unauthenticated client restarts the whole server:
  high.
- Unbounded memory or CPU consumption (decompression bombs, header or frame floods, rapid-reset style attacks): medium,
  or high if a single small request exhausts the process.
- Information leaks of low sensitivity: low.

## Anything to leave alone
- Trust boundary: in the supported production deployment the web tier and `lsphp` workers run as the same
  `nobody:nobody` principal, so code running in a PHP worker is already inside the trusted origin boundary (see
  `SECURITY.md`). Do not report issues whose only prerequisite and impact are the access of that shared principal,
  such as tampering with cache or log files it can write. Issues that cross this boundary without first obtaining
  code execution as that principal are welcome.
- Attacks that require control of the XML configuration, `.htaccess` content written by a trusted site owner, or
  local root.
- `unwrap`/`expect`/indexing clippy warnings, unless you can show a panic reachable from network input.

## Reports and patches
Include the affected revision, a minimal reproducer (a raw request or short script is ideal), and a minimal patch.
