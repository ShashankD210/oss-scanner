# Threat model — roshpinacare-sys/SovereignConsole (sovereign cockpit)

## What this project does and where untrusted input enters
- A Next.js 16 App Router console ("the wall") that measures a multi-account economy: it reads the estate engine APIs, GitHub APIs and public Steem RPC, renders live dashboards, and exposes a chat console backed by an LLM.
- Untrusted input: upstream API responses (chain RPC, GitHub), user chat messages, HTTP query/body parameters on its API routes.

## Components that matter most / least
- Most: API routes that spawn processes (estate re-arm), routes touching credential files (git credential store, the token file under upload/), the chat route (prompt injection into tool use), authentication/session handling.
- Least: static UI components, charts, screenshots.

## How to exercise it
- `bun install`, `bunx prisma generate`, `bun run build` (standalone output); `bun run dev` boots the wall on :3000; the API routes under src/app/api/* are the primary attack surface.

## How you rate severity
- Critical: secret exfiltration (token/WIF), command injection into spawn paths, unauthorized value movement.
- High: SSRF from API routes to internal services, auth bypass on state-changing routes.
- Medium: XSS in rendered dashboards, DoS of the console.

## Anything to leave alone
- None — the whole tree is in scope.
