# Threat model

## What this project does and where untrusted input enters
Cranpose is a Rust library for building user interfaces from composable functions. Applications link its crates and run on desktop, Android, iOS and the web. The framework runs inside the process of the application and opens no network port. `workers/cors-proxy` holds a small Cloudflare Worker for the web demo.

Untrusted input reaches the framework through data that an application loads or passes in:
- Text strings, which go through layout, shaping, hyphenation and editing code in `crates/cranpose-ui` and `crates/cranpose-foundation`.
- Images, SVG, fonts and asset files, through `crates/cranpose-assets`, `crates/cranpose-fonts` and the render crates.
- Audio files and streams, decoded in `crates/cranpose-media` with symphonia.
- Fluent localization catalogs, parsed in `crates/cranpose-localization`.
- HTTP responses and file picker results, through `crates/cranpose-services`.
- Calls across the UniFFI boundary from native host views, in `crates/cranpose-native`.
- Platform events such as pointer, keyboard, clipboard and accessibility requests, in `crates/cranpose-platform` and `crates/cranpose-app-shell`.

## Components that matter most / least
Most: `cranpose-media`, `cranpose-assets`, `cranpose-localization`, `cranpose-services`, the text and image code in `cranpose-ui`, `cranpose-native`, the web platform crate, and `workers/cors-proxy`.

Least: `cranpose-testing`, `apps/`, `xtask`, `benchmarks/`, `tools/`, `scripts/`, `plugins/` and the Gradle and Xcode files under `android/` and `platforms/`. `forks/` holds patched copies of upstream crates; report a bug there only when the bug comes from the Cranpose patch. `crates/cranpose-fonts-cjk-data-*` contain font bytes only.

## How to exercise it
Build from `/src` with the workspace in `Cargo.toml`. The image does not build the demo apps under `apps/` or `xtask`; the shell variable `SKIP_APPS` holds the `--exclude` flags for them. Each crate keeps tests in a `tests` or `test` directory that drive the public API. The scan environment has no GPU, so use the `renderer-pixels` software renderer or test the logic without rendering.

## How you rate severity
- Memory corruption that safe code can reach with untrusted input is high. If the attacker controls the corruption and it can lead to code execution, it is critical.
- The workspace sets `unsafe_code = "deny"` and a few modules allow it. A report about those modules needs a reproducer that shows a real effect.
- A panic or abort caused by untrusted data in a decoder, parser or text path is medium.
- Memory or CPU use that grows without bound from a small input is medium.
- Path traversal in asset or file handling, and SSRF or open relay behavior in the HTTP code or the worker, are high.
- A panic that only wrong use of the API by the application author can cause is low.

## Anything to leave alone
- Do not report problems in third-party crates listed in `Cargo.lock`, unless the way Cranpose calls them causes the problem.
- Do not report missing hardening in demo apps, benchmarks and scripts.
- Do not report `unwrap` or `expect` in tests.
- Do not report differences in performance or in rendered pixels.
