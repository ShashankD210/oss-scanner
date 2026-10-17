# Threat model

## What this project does and where untrusted input enters

ocf-scheduler-cf-plugin is a cf CLI plugin, written in Go, that drives the OCF Scheduler REST API. It reads the
logged-in user's API endpoint and UAA token from the cf CLI, derives the scheduler endpoint from it
(`api.<domain>` becomes `scheduler.<domain>`, `core/util.go`), and sends the token as a bearer token with each
request (`core/client.go`, `client/`). It creates, schedules, runs, lists and deletes jobs and calls, and it parses
and renders cron expressions locally (`commands/validate-*`, `cmd/render-cron-help`).

Untrusted input enters at:

- **Responses from the scheduler and the CF API.** A hostile or impersonated server controls the JSON that the
  plugin decodes and the names, commands, URLs and messages that it prints to the user's terminal.
- **The network path to the scheduler.** The user's UAA token travels on it.
- **Command-line arguments** (names, commands, URLs, cron expressions). The user is trusted, but the arguments
  are often pasted from scripts and documentation.

Trusted: the cf CLI itself and its configuration on the user's machine.

## Components that matter most / least

- Most: anything that can leak the user's UAA token, or send it anywhere other than the scheduler of the targeted
  foundation (`core/client.go`, `core/util.go`, TLS settings).
- Next: decoding and printing server responses (`client/`, `commands/`), and the cron expression parsing and
  rendering.
- Out of scope: `acceptance/` (needs a live Cloud Foundry), `ci/`, `scripts/`, `mk/`, and the release targets in
  the `Makefile`.

## How to exercise it

- `go test -race $(go list ./... | grep -v /acceptance)` runs the unit tests.
- `/src/ocf-scheduler-cf-plugin` is the built plugin. With no arguments it prints its build metadata. It normally
  runs under `cf`, which is not in the image; the `client/` and `core/` tests show how to drive the HTTP layer
  against a stub server.

## How you rate severity

- High: leaking the user's token, or sending it to a host other than the targeted scheduler, including through
  missing TLS verification.
- Medium: a server response that injects terminal escape sequences or misleads the user about what ran;
  crashes or hangs triggered by a server response.
- Low: crashes or bad output from the user's own command-line input.

## Anything to leave alone

- Nothing specific.
