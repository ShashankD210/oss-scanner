# Threat model

## What this project does and where untrusted input enters

Backend.AI Web UI is the browser and desktop (Electron) front end for Backend.AI, a platform that runs
user workloads in containers on shared GPU clusters. Authorization is enforced by the Backend.AI server;
the UI must not leak a user's credentials or session, and must not let one user's data run in another
user's browser.

It ships as:

- the React app (`react/`) and its component library (`packages/backend.ai-ui`);
- the API client SDK (`packages/backend.ai-client`);
- the Electron desktop app (`electron-app/`);
- a local proxy (`src/wsproxy/`) that connects the user's machine to apps running in their sessions.

Untrusted input reaches the UI from content other users create and this UI renders, from URL parameters
and deep links, from other web origins running in the same browser, and from the network between the UI
and the server. The configured API endpoint and the plugins it serves are trusted, as are the
deployment's `config.toml` and the build.

## Components that matter most / least

Most important: the Electron app and the local proxy (both run on the user's machine), authentication and
session handling in the client SDK, and anywhere other users' content is rendered.

Lower priority: admin-only pages, build and docs tooling, and e2e tests. Out of scope: the Backend.AI
server (a separate repository) and HTTP headers that the deployment configures.

## How to exercise it

- Unit tests (Vitest, offline): `pnpm --filter backend.ai-client test`, `pnpm exec vitest run` (root),
  `cd react && pnpm exec vitest run`, `pnpm --filter backend.ai-ui exec vitest run`.
- The built web app is in `build/web`. There is no Backend.AI server in the image; stub the API with a
  small local HTTP server to show impact.

## How you rate severity

- Critical: code execution on the user's machine, or theft of a user's credentials or session, without
  user action beyond visiting a page.
- High: stored XSS across users; another web origin acting through the user's local components;
  weakening TLS so a network attacker can read or change API traffic.
- Medium: issues that need the user to open a crafted link, such as reflected XSS or open redirects.
- Low: issues that need control of the configured endpoint or deployment config, self-XSS, and missing
  hardening without a demonstrated exploit.

## Anything to leave alone

- Plugins served by the configured endpoint run with full app privileges by design.
- Dependency advisories without a reachable path in this code.
- Missing security headers; the deployment sets those.

## How to report

Please include reproduction steps, the attacker's position, and a proposed patch where possible.
