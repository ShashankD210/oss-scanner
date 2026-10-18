# Threat model: linux-mac

Build system for custom Linux kernels purpose-built for Apple Mac hardware (notably the
Mac Pro 6,1): shell build scripts, per-model kernel configs, kernel patches, and
packaging. build.sh fetches upstream kernel source, applies patches from patches/, and
builds with the model's config.

## Untrusted input
- Downloaded kernel source and toolchain components (supply chain).
- Patch files and configs applied during the build.
- Script arguments and environment (model names, versions, paths).

## In scope
- Whether the build verifies kernel source integrity (checksums/signatures) before use.
- Command injection and path traversal in build scripts (model/version arguments reach
  the filesystem and shell).
- Patch-application safety: malicious or corrupted patch context should fail loudly,
  not produce a silently weakened kernel.
- Security-relevant kernel config regressions in configs/ (e.g. accidentally disabled
  hardening options).

## Out of scope
- Upstream kernel vulnerabilities themselves; the hardware.

## Severity guidance
- Build path that yields a compromised kernel (unverified source, applied malicious
  patch): critical.
- Command injection in build scripts: high.
- Disabled security hardening in shipped configs: medium to high.
- Build breakage without security impact: low.

## Report format
Minimal reproducer plus a small candidate patch; one issue per report, deduplicated by
root cause.
