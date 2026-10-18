# Threat model — Oikonome

## What this project does and where untrusted input enters
Oikonome is a self-hosted personal-finance app: a Python 3.14 / FastAPI server (`server/oikonome/`)
with a Postgres database, a React web app it serves (`webapp/`), a background worker (`server/oikonome/jobs/`)
and an Expo mobile app (`mobile/`) that talks to the same HTTP API. One instance can serve several
households ("tenants"); every domain table carries Postgres row-level security keyed to the tenant,
and the app role cannot do DDL or reach superuser.

Untrusted input enters through:
- **The HTTP API and pages** (`server/oikonome/web/api.py`, `pages.py`, `app.py`): signup/login
  (Argon2id passwords, TOTP, passkeys/WebAuthn, recovery codes), the household's own JSON doors, the
  admin console (`adminconsole.py`, operator-only, passkey or break-glass token), scoped API tokens
  for collector scripts (`api_tokens`), widget and device tokens, invites and email-confirmation links.
- **File importers** (`server/oikonome/sync/{csvimport,ofximport,qifimport,mintimport,ynabimport,pdfimport}.py`,
  `web/bulk_import.py`): statements and competitor exports uploaded by a signed-in member. PDFs are
  parsed in a sandboxed child process (`sync/pdf_sandbox.py`). Uploads are size- and rate-bounded.
- **Bank aggregator data** (`sync/plaid.py`, `sync/mx.py`, `sync/simplefin.py`, `sync/adopt.py`):
  account and transaction payloads from a third party, the Plaid webhook receiver
  (`web/plaid_webhook.py`, signature-verified), Hosted Link sessions and their polling.
- **Receipts and merchant text**: user-uploaded receipt images and the merchant/description strings
  that reach the categorizer and the LLM assistant (`engine/assistant.py`, `engine/llm_categorize.py`)
  — an operator-configured model endpoint; prompt content is household data, never trusted as code.
- **Outbound doors** (`web/netguard.py` guards every operator-supplied URL): webhooks, the logo proxy,
  SMTP and LLM endpoints, SimpleFIN claim URLs — SSRF and DNS-rebinding are in scope.
- **Mail**: verification, reset and daily-report emails rendered from household data (`notify/`,
  `web/todayview.py`, `web/lens_email.py`); the recipients list is owner-controlled.

## Components that matter most / least
- **Most:** tenant isolation (RLS, `db/tenancy.py`, every `tenant_connect` caller), authentication and
  step-up (`web/app.py`, `web/security.py`, passkeys, recovery), the envelope encryption of secrets
  under the master key (`db/crypto.py`), the full-export and database-dump doors, the institution-cap
  and billing-adjacent link paths (`sync/plaid.py` `link_item`, `jobs/plaid_links.py`), the importers
  and the PDF sandbox, `netguard.py`, the admin console and its host-agent commands, the webhook
  signing, the rate limiters and auto-ban (`web/security.py`).
- **In scope but lower:** the money math (`engine/budget.py`, `engine/forecast.py`, `engine/bills.py`)
  — wrong numbers are correctness bugs, not security ones, unless cross-tenant.
- **Out of scope:** `server/oikonome/engine/merchant_seed_data.py` (generated brand table), `docs/`,
  the demo-instance seeding (`server/oikonome/demo.py`, a synthetic household), `mobile/android/` and
  `mobile/ios/` prebuild output, third-party aggregators' own behaviour, an operator's deployment
  misconfiguration (the `/doctor` page exists for that).

## How to exercise it
- `oikonome-tests` (installed by the Dockerfile) starts the local Postgres on 127.0.0.1:5433, runs
  `make lint` and the full server suite (`make test`, ~8,600 tests, 15–40 minutes; it creates its own
  test databases and the least-privilege app role) and the mobile logic tests
  (`node --experimental-strip-types mobile/scripts/pure-logic-tests.mjs`).
- One module: `cd /src/server && .venv/bin/python -m unittest tests.<module>` after
  `service postgresql start`. The tests in `server/tests/` are named for the behaviour they protect;
  `test_scope_invariants.py` and the `test_*rls*`/tenancy tests are the isolation proofs.
- Run the server against the test database for live probing (no aggregator, no mail needed):
  `cd /src/server && OIKONOME_DEV=1 OIKONOME_DSN=postgresql://oikonome_app:apppass@127.0.0.1:5433/oikonome_test
  OIKONOME_ADMIN_DSN=postgresql://postgres:devpass@127.0.0.1:5433/oikonome_test
  OIKONOME_MASTER_KEY=$(python -c 'import os,base64;print(base64.urlsafe_b64encode(os.urandom(32)).decode())')
  .venv/bin/python -m uvicorn oikonome.web.app:app --port 8129` after the suite has created
  `oikonome_test` (or run one test module first). `OIKONOME_DEV=1` opens `/api/signup` so you can
  mint households; `OIKONOME_HOSTED=1 OIKONOME_HOSTED_BILLING=none OIKONOME_OPEN_SIGNUP=1` adds the
  multi-household (hosted) shape. The web app is served at `/app/`; the API under `/api/`.
- Importer corpora: `server/tests/fixtures/` (synthetic statements and exports).
- Known offline/root-dependent results of the full suite inside this image (8,633 tests, 860 s):
  `test_auth_and_tenant_isolation.test_ssrf_guard_hosted_vs_selfhost` needs DNS (it resolves a
  public host to prove it is allowed); `test_reset_and_uninstall_keep_the_master_key.test_what_is_left_behind_is_named`
  expects a non-root runner; `test_institution_cap.test_link_disconnect_relink_churn_reaches_the_cap`
  and `test_item_slots.test_an_item_refused_at_the_cap_still_holds_its_place` need an operator gate
  add-on that this repository does not carry (a known gap). Everything else passes offline.

## How you rate severity
- **Critical:** any cross-tenant read or write (RLS bypass, a `tenant_connect` with the wrong tenant,
  an admin-role connection reachable from a household request); authentication bypass or a step-up
  (password/TOTP/passkey) that can be skipped; plaintext exposure of an envelope-encrypted secret
  (aggregator tokens, TOTP seeds, API keys) or of the master key; remote code execution in the
  server or the worker; SSRF reaching the container's network or metadata endpoints.
- **High:** a member or viewer reaching an owner-only door (bank linking, billing-adjacent link
  paths, member management, erasure, export, the database dump); an unauthenticated door that
  writes; an importer or receipt parser that can be made to read arbitrary files, exhaust memory
  or hang the worker; stored XSS in the web app reachable by another household member; a webhook
  signature or token-scope check that can be bypassed; an institution-cap or billing bypass on the
  hosted shape (an attacker minting aggregator Items the operator pays for).
- **Medium:** self-XSS, open redirects, information leaks limited to the attacker's own household,
  rate-limit gaps that enable brute force at a realistic cost, denial of service needing an
  authenticated member, timing side channels on secrets.
- **Low:** issues needing an already-compromised operator host or the postgres superuser, verbose
  errors without secrets, hardening suggestions.
- Please include a reproducer against the test server above where you can; a proposed patch in the
  project's style (small functions, tests named for the behaviour) is welcome.

## Anything to leave alone
- The demo instance mode (`OIKONOME_DEMO=1`) deliberately exposes a shared login with most doors
  refused (`web/demoguard.py`); reports about the demo account seeing its own synthetic data are not
  findings — a demo login reaching a non-demo household, or a credential door, is.
- The one-shot migration container runs as the postgres superuser by design; the app and worker
  roles do not.
- Rate limits are tuned for a household, not a public SaaS; "limit is generous" alone is not a
  finding unless it enables a concrete attack.
