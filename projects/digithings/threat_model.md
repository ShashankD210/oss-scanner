# digithings — scanner threat model

This file tells the OSS Scanner what digithings is, which code is worth adversarial
attention, what counts as severe, and how we would like findings reported. It is
maintainer-authored guidance, not a claim that the code is correct.

Our own model of the system is in `SECURITY.md` at the repository root (STRIDE table,
non-negotiable defaults, residual risks). This file complements it: it points at the
code, not the policy.

## What digithings is

A self-hosted, single-tenant-per-host platform for building conversational agents.
A chat request reaches **digigraph** (LangGraph orchestration, OpenAI-compatible HTTP
API), which selects MCP tools and calls sibling services. Authentication is
**digikey** (RS256 JWT + scoped API keys). Retrieval is **digisearch**; outbound
fetching is **digifetch**; audit, CORS and HTTP client behaviour are shared through
**digibase**. **digiquant** runs backtests and holds (currently stubbed) broker
adapters. The default deployment binds loopback; remote access is expected to go
through a gateway.

## In scope, in priority order

| # | Component | Path | Why an attacker cares |
|---|-----------|------|-----------------------|
| 1 | **digikey** | `digikey/` | The auth plane. RS256 verification, JWKS fetch/rotation/caching, scoped API keys, per-IP rate limiting, bcrypt admin paths. A bug here is a bypass of every other control. |
| 2 | **digigraph** | `digigraph/` | Chat text, conversation history and retrieved documents reach an LLM whose output selects tool calls. MCP tool registry/allowlist, tool-argument validation, debug/thread endpoints, streaming and session handling. |
| 3 | **digibase** | `digibase/` | Shared by every service: `http_client` timeouts and redirects, `cors` allowlist, `audit.redact_mapping`, error handlers. A defect here is a defect in all services. |
| 4 | **digisearch** | `digisearch/` | Index and document ingestion (Azure/Chroma backends), query construction, result assembly. Untrusted document text is an injection vector into digigraph prompts. |
| 5 | **digifetch** | `digifetch/` | Fetches attacker-supplied URLs and parses attacker-supplied HTML/PDF. Look for SSRF (scheme, redirect, DNS-rebinding, private-address reachability), parser abuse, size/timeout limits, header injection. |
| 6 | **digivault** | `digivault/` | Parses untrusted markdown: frontmatter (YAML), wikilinks, backlinks, path handling on read/write. |
| 7 | **digiquant** | `digiquant/` | Backtest/optimize pipeline and the broker adapters (`ib`, `alpaca`, `quantconnect`). The adapters must raise `NotImplementedError`; anything that lets a chat request reach order placement, or that widens the vault's credential handling, is critical. |
| 8 | **digillm, digiskills, digitrace, digiclaw** | same | Provider routing and key handling, skill loading, span/audit writers, heartbeat and scheduler. Lower priority; still in scope. |

## Treat as adversarial input

- Chat messages, conversation history, tool-call arguments and any JSON body
  (every handler uses Pydantic v2 with `extra="forbid"` — check that it actually does).
- Retrieved document text and document metadata from digisearch.
- HTML, PDF and HTTP responses fetched by digifetch, including redirect chains.
- `Authorization` headers, JWTs (including `alg`, `kid`, `exp`/`nbf`, claim typing)
  and API keys; JWKS documents fetched from a configured URL.
- Markdown and frontmatter read by digivault.
- Paths and filenames supplied through any API, including engagement paths.
- Environment variables and config files that name URLs, hosts or keys.

## Out of scope

- **Third-party dependencies.** Report those upstream, or to us as a supply-chain
  note if there is a concrete reachable path. `digichat/` is a separate nested
  repository and the Next.js frontend is not built in this image; UI-level issues
  are out of scope for this scan.
- **Deployment and infrastructure.** `docker-compose*.yml`, the Cloudflare and
  Fly configuration, `scripts/`, CI workflows and the ops runbooks are present in
  the repository but are not the target of this scan.
- **The `data/`, `projects/` and `.claude/` trees.** Client data, local research
  artifacts and agent tooling. `projects/` is gitignored and never in this image.
- **"Not designed for the public internet."** See the review-cadence note in
  `SECURITY.md`: the platform assumes loopback or a hardened gateway. "Endpoint is
  reachable if you expose it publicly" is not by itself a finding; a missing control
  that the code claims to have **is**.
- **Prompt-injection-as-a-class without a concrete consequence.** We know retrieved
  text is unsanitised; that is a documented residual risk. What we want is a
  reproducible path from injected text to a specific tool call, credential read,
  egress request or audit-record falsification.

## Severity rubric

- **Critical** — pre-auth RCE or auth bypass in digikey; JWT/JWKS forgery or
  acceptance of an unsigned/`none`-alg token; secret or credential disclosure to an
  unauthenticated caller; a path that lets a chat request place a broker order;
  cross-tenant read of another engagement's data.
- **High** — authenticated privilege escalation or scope escape; SSRF reaching
  cloud metadata or internal services from digifetch; prompt injection with a
  demonstrated side-effecting tool call; secrets written into logs, spans or audit
  events; unbounded resource use triggerable by one request.
- **Medium** — DoS requiring authentication or modest volume; information disclosure
  of non-secret internals (version, stack traces, config shape); missing rate limit
  on an authenticated path; CSRF/CORS misconfiguration with a demonstrable victim.
- **Low** — defence-in-depth gaps, hardening, weak defaults that the documentation
  already warns about, and correctness bugs with no security consequence.

## How we would like findings reported

- Lead with the **consequence and the reachable path**: which endpoint or entry
  point, which trust boundary is crossed, what the attacker gains. A CVE-shaped
  narrative with no reachable trigger is not useful to us.
- Include the **exact file and line range** in this repository, and say which
  component from the table above it belongs to. Our harness rejects findings that
  land in a dependency or outside the scanned repository.
- For auth and injection findings, a **minimal PoC** (a `curl`, a pytest, or a short
  script) is worth more than prose. Where a PoC would need credentials, say which
  scope or role is required.
- **Patch style:** a minimal, reviewable diff against `develop` that fixes the
  specific issue, with a test that fails before the patch. We do not want a
  refactor attached to a security fix; we will not merge one.
- **Deduplicate** at the level of "same root cause, same fix". Two endpoints sharing
  one helper is one finding with two call sites, not two findings. Do not split a
  single missing check across the files it touches.

## Attribution

We are happy to credit findings. If we patch a vulnerability you report we will
reference the report ID in the commit message, in the form
`Discovered by Anthropic's OSS Scanner, as vulnerability ANT-…`, unless you ask us
not to.
