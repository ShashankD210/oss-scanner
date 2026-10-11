# Threat model

## What this project does and where untrusted input enters
Cranpose Showcase is a sample application and a project template for Cranpose. It draws an interactive star chart. It runs on desktop, Android, iOS and the web. It has no server. It loads bundled data from `assets/` and reads user input events.

Untrusted input is limited to touch, mouse and keyboard events, and to data that a developer replaces in `src/data` when the template is turned into a real application.

## Components that matter most / least
Most: `src/data` and `src/presentation`, the code that reads data and turns it into state. Copies of the template carry this code into other applications.

Least: the `android/` and `ios/` project files, `robot-runners/`, `scripts/`, `tests/` and `docs/`. The Cranpose crates are scanned in their own enrolment.

## How to exercise it
Build from `/src` with `cargo build`. Run `cargo test --all-targets`. The `robot-runners` drive the interface through test harness code and need a window system, which the scan environment does not have.

## How you rate severity
- Memory corruption or code execution from crafted data is high or critical.
- A panic caused by input events or by the bundled data is low, because the application only runs locally.
- Template code that is easy to copy into an application and then becomes a vulnerability there is medium.

## Anything to leave alone
- Do not report the lack of authentication or encryption. The sample has no accounts or network access.
- Do not report problems in third-party crates unless this project calls them in an unsafe way.
