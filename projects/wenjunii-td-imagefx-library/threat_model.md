# TD ImageFX Library

This project distributes versioned TouchDesigner image effects and a portable,
zero-runtime-dependency Python package manager. The primary security surface is
the package/feed/update path; follow the repository's `SECURITY.md`.

## Inputs and boundaries

Treat feed JSON, manifests, ZIP archives, package identifiers, semantic versions,
source URLs, project lockfiles, and imported show/preset files as untrusted.
Prioritize traversal, symlink and special-file escapes, archive collisions and
bombs, malformed JSON, redirects, source identity confusion, digest/manifest
binding failures, and partial or mutable installs. Relevant code includes
`src/tdimagefx/archive.py`, `feed.py`, `paths.py`, `manifest.py`, `registry.py`,
`state.py`, `lockfile.py`, and the TouchDesigner updater integration.

Update discovery is notify-only and must not install, import, activate, or execute
remote code. Local sources need explicit enablement and confinement. Staging
must bind the artifact digest, manifest digest, identity, and exact version.
Published package versions are immutable. URLs in status/registry files must
not disclose embedded credentials, queries, or fragments.

## Offline checks

The checkout is `/src`, with `PYTHONPATH=/src/src`. Run:

```sh
python -m unittest discover -v
python tools/verify_repository.py
python -m compileall -q src tools touchdesigner/extensions
```

The Linux image exercises the portable package core and source-based integration
tests. Proprietary TouchDesigner, live `.toe`/`.tox` rendering, GPUs, private
media, and native host APIs are unavailable. Test updater boundaries with local
fixtures and mocks; do not publish releases or mutate historical packages.

## Severity and reporting

Prioritize execution or arbitrary writes from an attacker-controlled update,
trust/digest bypass, and disclosure of secrets. Explain which user action or
local-source setting is required. Distinguish intentional loading of a trusted
effect from execution reached during untrusted discovery/staging. Provide a
minimal offline malicious-package fixture and regression test. Fix vulnerable
packages using a new exact version; keep reports private until resolved.
