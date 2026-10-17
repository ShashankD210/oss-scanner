# Threat model: LIA

LIA is a self-hosted, multi-account personal assistant. One deployment serves several accounts: each person
connects their own Google, Microsoft and Apple accounts through OAuth (mail, calendar, contacts, tasks, Drive) and
their own API keys for other services. LIA reads that data and acts on it through LLM-driven agents (a planner
pipeline and a ReAct loop, built on LangGraph), with human-in-the-loop confirmation before side effects. The
operator who deploys an instance is its administrator. Some instances, including a public demonstrator, accept
sign-ups from people the operator does not know.

The repository is a monorepo:

- `apps/api`: the backend (Python 3.14, FastAPI, LangGraph, SQLAlchemy on PostgreSQL with pgvector, Redis).
- `apps/web`: the frontend (Next.js 16, React 19, TypeScript).
- `apps/mobile`: Capacitor shells for Android and iOS that load the person's own server in a WebView.

The project's own policy is `SECURITY.md` at the root of the repository; this file refines it for the scanner.

## What must stay protected

- **Another account's data**: e-mails, events, contacts, files, conversations, memories, journals, documents in
  knowledge spaces, meeting and voice transcripts, generated files.
- **Credentials**: OAuth access and refresh tokens and per-account API keys (Fernet-encrypted at rest), session
  cookies, MFA secrets, the deployment's own provider keys, `SECRET_KEY` and `FERNET_KEY`.
- **Acting in someone's name**: sending, forwarding or deleting e-mail, changing a calendar, placing a phone call,
  sharing a file with another account. Every tool declares what it owes the person (`mutation_policy` on its
  manifest), and the effect gate (`apps/api/src/domains/agents/effects`) and the HITL flow enforce it. An action
  that runs without the confirmation its tool declares is a vulnerability.

## Where untrusted input enters

1. **Anonymous network clients.** Registration, password login, OAuth login and callbacks (PKCE, single-use
   `state`), MFA verification, the native-app handoff (`apps/api/src/domains/auth`), public pages, and webhooks:
   Telegram (`apps/api/src/infrastructure/channels/telegram/webhook_handler.py`), the telephony vendor
   (`apps/api/src/domains/telephony/webhook_handler.py`, `apps/api/src/domains/agents/telephony/live_tools_router.py`),
   Google push notifications (`apps/api/src/domains/push_channels`), the push relay
   (`apps/api/src/domains/push_relay`) and the alert webhook (`apps/api/src/domains/diagnostics/webhook_router.py`).
   Check signature or secret verification, replay, and that a webhook cannot act for an account it does not belong to.
2. **One account against another.** Every resource identifier (conversation, attachment, document, knowledge
   space, ticket, routine, reminder, bookmark, connector, MCP server, skill, image) must be checked against its
   owner. Peer connections (`apps/api/src/domains/peers`) and the workboard share specific objects between two
   accounts on purpose: sharing must stay limited to what the owner shared.
3. **An account against the administrator.** Administrative routes require a superuser
   (`apps/api/src/core/security/authorization.py`). A regular account must not reach them, nor another account's
   registers and exports.
4. **Content LIA reads for the person, written by someone else**: incoming e-mails and their attachments,
   calendar invitations, shared documents, web pages fetched by the browsing and web tools, search results, files
   uploaded or synchronised into knowledge spaces (PDF, Office formats, images), RSS feeds, responses from MCP
   servers, third-party skills, messages from peer accounts. A prompt injection in that content must not:
   trigger a side-effecting tool without the confirmation its manifest declares; send data to a destination the
   attacker chooses (a URL, an e-mail recipient, an MCP server, an image or link rendered in the chat); or reach
   the sandbox with more than it is given.
5. **Outbound requests to URLs that a person or a piece of content supplies** (browser, web fetch, MCP servers,
   OAuth discovery for personal MCP servers, feeds, image proxies). The SSRF defences are
   `apps/api/src/infrastructure/utils/url_validation.py` (`validate_url`, `pinned_stream`: the address that was
   checked is the address that is fetched, one validated hop at a time) and
   `apps/api/src/infrastructure/browser/security.py`.
6. **Code execution.** Python written by the model (`run_python_tool`) and skill commands run in a separate
   sandbox container (`apps/api/Dockerfile.sandbox`); its network goes through an egress proxy that swaps per-run
   tokens for the person's keys on declared hosts only. Third-party skills (`apps/api/src/domains/skills`,
   `apps/api/src/domains/skill_library`, trust decided in `apps/api/src/domains/skills/trust.py`) are untrusted.
7. **Rendering in the browser.** Answers are Markdown rendered with sanitisation
   (`apps/web/src/components/chat/MarkdownContent.tsx`), plus deterministic HTML cards
   (`docs/technical/HTML_CARDS.md`), Mermaid and KaTeX; MCP-App widgets run in a sandboxed, opaque-origin iframe;
   words relayed from another account are quoted literally (`apps/api/src/domains/shared/markdown_literal.py`).
   The Content-Security-Policy is built in `apps/web/src/lib/csp.ts`.
8. **Mobile shells** (`apps/mobile`). The sign-in return uses the custom scheme `lia://auth-callback`, whose code
   is bound to a verifier the shell keeps (`apps/api/src/domains/auth/native_handoff.py`), because a custom scheme
   can be intercepted.

Features behind a setting that is off by default (MCP, channels, skills, telephony, push, and others) are in
scope when enabled: enabling a shipped feature is not a misconfiguration.

## How to exercise it offline

- The checkout is at `/src`. The backend's virtualenv is `/src/apps/api/.venv` and is first on `PATH`
  (`python`, `pytest`, `alembic`, `uvicorn`). The frontend has Node.js 24, pnpm 10, its `node_modules`, and a
  production build in `apps/web/.next` unless the build log reports a warning.
- `lia-services start` starts PostgreSQL 17 with pgvector on `127.0.0.1:5432` (role `test`, password `test`,
  database `test_db`) and Redis on `127.0.0.1:6379`, the services the test settings of the image point at.
- Backend unit tests, from `/src/apps/api`:
  `pytest tests/unit -m "not integration and not slow and not e2e and not benchmark and not multiprocess" --no-cov -n 2`.
  Tests that need the database are skipped while the services are stopped.
- Backend integration tests: `lia-services start`, then from `/src/apps/api`:
  `pytest tests/integration -m "not e2e and not benchmark and not multiprocess" --no-cov`.
- A running backend: `lia-api` applies the migrations and serves the API on `http://127.0.0.1:8000` (routes under
  `/api/v1`) with the stub settings of `apps/api/.env.test`. Accounts can be created through
  `POST /api/v1/auth/register`; an administrator through
  `echo '<password>' | python -m scripts.data.create_admin --email <address>` from `/src/apps/api`.
- A running frontend: with `lia-api` up, `lia-web` serves the production build on `http://127.0.0.1:3000` and
  proxies `/api/v1` to the backend, same-origin as in production. Chromium is installed (`/usr/bin/chromium`) for
  browser-side reproductions.
- The test settings differ from a deployment's defaults on purpose: `RATE_LIMIT_ENABLED=false` (the shipped
  default is `true`), no SMTP (verification e-mails are not sent), and no provider key, so embeddings, semantic
  search and knowledge-space indexing are unavailable. Judge a finding against the shipped defaults, not against
  the test settings.
- Frontend tests, from `/src/apps/web`: `pnpm test`.
- There is no network and no provider credential in the image. LLM, e-mail, calendar, search, telephony and
  storage providers must be mocked; the existing suites (`apps/api/tests`, `apps/web/src/**/__tests__`) show how.

## How severity is rated

CVSS v3.1, as `SECURITY.md` uses it, with these anchors:

- **Critical**: code execution on the API host without an account; taking over another account; reading another
  account's OAuth tokens or keys, or `SECRET_KEY` / `FERNET_KEY`; escaping the sandbox to the API host or to
  credentials.
- **High**: reading or changing another account's private data with an ordinary account; becoming administrator;
  stored XSS that runs in another account's session or the administrator's; a prompt injection in content LIA
  reads that makes it act without the confirmation the tool declares, or sends the person's data to a destination
  the attacker chooses; SSRF that reaches internal services (PostgreSQL, Redis, cloud metadata, the sandbox
  proxy); a forged webhook that acts for an account.
- **Medium**: blind SSRF; disclosure of another account's metadata (existence, counts, titles); credentials or a
  person's content written to logs above DEBUG; a single authenticated request that exhausts a worker or the
  database; CSRF on a state-changing route.
- **Low**: a demonstrated weakness with minor impact, such as a verbose error that reveals internal paths.

## Out of scope

- Anything that requires the administrator's privileges or control of the deployment (its `.env`, Compose files
  or host): the administrator is trusted and holds every key.
- A person acting on their own data or their own session: self-XSS, or asking LIA to do something to their own
  account.
- A prompt injection whose only effect is the wording of an answer: no action taken, no data leaving the account,
  no confirmation bypassed.
- Model behaviour as such (hallucinations, refusals, answer quality), and spending within the budget a person or
  the operator configured.
- Vulnerabilities in dependencies without an exploitable path through LIA's code.
- Developer tooling and offline assets: `scripts/` (except `scripts/install/`, the self-host installer, which is
  shipped to users and in scope), `tools/`, `docs/`, and the observability configuration under `infrastructure/`
  (unless a shipped default is exploitable).
- Best-practice hardening without demonstrated impact, and denial of service against development-only endpoints,
  as `SECURITY.md` states.

## Reports and patches

- One report per root cause: the same missing ownership check on several routes of one router is one finding,
  with every route listed.
- State the attacker's position (anonymous, an ordinary account, an author of content LIA reads), the affected
  files and lines, and a minimal offline reproduction, preferably a pytest test using the fixtures of
  `apps/api/tests`.
- Propose a minimal patch with a regression test. The repository's conventions are in `CLAUDE.md` at its root.
