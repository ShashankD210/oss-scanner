# Threat model

## What this project does and where untrusted input enters
Cranamp is a music player and skin editor for Windows, macOS, Linux, Android, iOS and the web. It is built on the Cranpose UI framework. A user opens files that come from other people, so file parsers are the main exposure.

Untrusted input:
- Classic skin archives (`.wsz` and `.zip`) with bitmaps, cursors (`.cur`) and text configuration files, parsed in `src/winamp`.
- Layered skin projects (`.cstudio`) that the Skin Studio loads.
- Playlists in M3U, M3U8 and PLS format, from local files and from remote URLs.
- Audio files and radio or HTTP streams, decoded through Cranpose media code.
- Files in the sync folder, parsed in `src/sync`.
- The local agent endpoint in `src/winamp/studio/agent_server.rs`, which listens on `127.0.0.1:18765`. It accepts requests from other programs on the same computer.
- Release metadata for the Android APK updater.

## Components that matter most / least
Most: skin and archive parsing, image and cursor decoding, playlist parsing, the sync code, the agent server and the updater.

Least: the `deploy/`, `site/` and `docs/` directories, the `build.rs` file and the platform packaging in `platform/`. Code from the Cranpose crates is covered by the Cranpose enrolment; report it here only when Cranamp passes input to it in an unsafe way.

## How to exercise it
Build from `/src` with `cargo build`. The integration tests in `test/` and the fixtures in `test/fixtures` cover skins, studio drawing and settings. Skin archives can be fed to the loader directly. There is no GPU or audio device in the scan environment.

## How you rate severity
- Memory corruption, or code execution through a crafted skin, playlist, project or stream, is high or critical.
- A zip entry that writes or reads outside its intended directory is high.
- A request to the agent server from a web page or another local user that changes documents or reads files is high.
- A panic, hang or very large allocation caused by a crafted file is medium.
- A crash that needs a local user to change their own settings files is low.

## Anything to leave alone
- Do not report that streams and remote playlists can point at any URL the user chooses. That is a feature.
- Do not report problems in third-party crates unless Cranamp calls them in an unsafe way.
- Do not report differences in how a skin looks.
