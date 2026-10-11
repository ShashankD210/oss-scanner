# Threat model

## Scope and inputs

This small TypeScript CLI/library exports Playwright trace ZIPs to Markdown reports and HTML DOM snapshots. This enrollment makes no claim about downstream adoption. The checkout is at `/src`; the compiled library, declarations, source maps and CLI are in `/src/lib`.

Treat trace archives as adversarial, including ZIP metadata, compressed sizes, member names, `.trace`/`.network` NDJSON records, snapshot DOM, URLs, resource identifiers, attachments and console/error text. A user can be induced to inspect someone else's trace. Legitimate traces can also contain credentials, cookies, request bodies and other sensitive data; exporting does not make these public-safe.

Important code is `src/traceExporter.ts`, `src/zipFile.ts`, `src/stringUtils.ts` and `src/cli.ts`. Review archive/decompression resource exhaustion, malformed event handling, trace-derived destination paths and output overwrites, HTML/script/URL contexts, and Markdown injection. A resource identifier is not trustworthy merely because it is named SHA1. The exporter reads selected assets, not a generic full-archive extraction, so demonstrate a reachable export path rather than assuming conventional unzip behavior.

## Boundaries and severity

The intended filesystem boundary is the user-selected output directory. Attacker-controlled writes outside it, or executable content reaching a privileged consumer unexpectedly, deserve high priority. Sustained resource exhaustion from reasonably sized malicious archives matters, but distinguish decompression amplification from deliberately exporting already enormous trusted recordings. Source paths and a deliberately selected output directory are controlled by the CLI user, not remote service requests.

Generated snapshot HTML reconstructs captured DOM and includes restoration scripts; it is not an inert sanitizer. Open it only in an isolated browser, without sensitive sessions, and with networking disabled. Distinguish expected captured-page behavior from exporter-created injection or escape into a consumer's trust context. URLs absent from captured resources may remain external; use container network isolation. Reports themselves may contain attacker-controlled links or instructions. A downstream LLM should not treat trace text as privileged instructions.

## Offline exercise

The Dockerfile installs locked npm dependencies and matching Chromium plus Linux browser libraries online, then compiles with `npm run build`. No tests run during image construction. It copies the full repository, including the 12 real ZIP fixtures in `test/assets`; do not substitute fabricated archives or prune these assets.

Run inside the built image with networking disabled:

```sh
node lib/cli.js --help
node lib/cli.js test/assets/test-trace1.zip -o /tmp/trace-export
node lib/cli.js test/assets/test-trace-attachments.zip -o /tmp/trace-attachments
node lib/cli.js test/assets/trace-1.31.zip -o /tmp/trace-legacy
test -s /tmp/trace-export/index.md && test -s /tmp/trace-export/network.md
npm run typecheck
npm test -- --workers=2
```

`test/trace-exporter.spec.ts` exercises real archives, reports, attachments, older formats, resources, snapshot CSS, screenshots, scrolling, shadow DOM and click highlighting. Browser tests load local reconstructed HTML. Dependencies and Chromium are preinstalled; no `npx` package downloads or live web services are required. `/tmp` and Playwright's test-results directory must remain writable. These commands are prepared from source, not yet executed as part of enrollment.

## Capability limits

This is a Linux Node/Chromium environment. Windows drive paths, UNC paths, Windows filesystem semantics and macOS behavior are not exercised. It does not run an LLM or contact an LLM provider. It has no credentials, deployed service or privileged user browser. Do not fetch URLs from untrusted traces, contact recorded hosts, or use real customer traces containing secrets. Native dependencies and browser engines remain available for local isolated reproductions; network-disabled tests do not establish complete browser or operating-system security.
