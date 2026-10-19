# Threat model

## What this project does and where untrusted input enters

ViperCapture is a self-hosted browser rendering API. Callers send a URL, HTML, or Markdown and get screenshots, PDFs, video, HTML, Markdown, metadata, or archives back. Rendering runs in headless Chromium, Firefox, and WebKit behind the HTTP API in `vipercapture/main.py`.

Untrusted input enters as:

- Capture URLs (`url`, `base_url`, navigations, and subresources). These are the SSRF, redirect, DNS-rebinding, and private-network paths, including link-local and metadata addresses. `PublicUrlValidator` in `vipercapture/render_engine.py` and `WebhookDispatcher` in `vipercapture/webhooks.py` are the checks.
- Render and job request bodies: `POST /v1/render`, `/v1/jobs`, bulk jobs, schedules, and the ScreenshotOne and Urlbox adapters. Bodies carry selectors, headers, cookies, scripts, cleanup rules, and output options (`vipercapture/render_contract.py`).
- Auth tokens and API keys: `Authorization`, the admin token, project API keys (`vipercapture/control_plane.py`), and HMAC signed render URLs (`vipercapture/signed_urls.py`).
- Proxy credentials on a render (`network.proxy` in the contract, passed through to Playwright).
- Uploaded and returned files: baseline images, imported browser sessions and profiles, artifact paths, ZIP/HAR/WARC/MHTML archives, and media ffmpeg writes.

## Components that matter most / least

Most: `vipercapture/render_engine.py`, `vipercapture/render_contract.py`, `vipercapture/webhooks.py`, `vipercapture/main.py`, `vipercapture/signed_urls.py`, `vipercapture/control_plane.py`, `vipercapture/async_jobs.py`, `vipercapture/async_job_providers.py`, `vipercapture/s3_artifact_store.py`, `vipercapture/session_import.py`, `vipercapture/content_rendering.py`, and `vipercapture/capture_cli.py`.

Also in scope: `frontend/` (the Vite/React UI, built into `static/app`), `templates/` and `static/` (the local browser UI), `launch.py`, `gui.py`, `updater.py`, `vipercapture/page_cleanup.py`, `vipercapture/visual_diff.py`, `vipercapture/schedules.py`, `vipercapture/render_cache.py`, and `vipercapture/compatibility.py`.

Out of scope: third-party dependencies and `vendor/autoconsent`, dev tooling and CI (`.github/`, `scripts/`, `tests/`), `deploy/` and `integrations/`, the hosted ViperCapture service, and the sites being captured.

## How to exercise it

The checkout is `/src`. Dependencies, browsers (`PLAYWRIGHT_BROWSERS_PATH=/ms-playwright`), ffmpeg, and the frontend build are already in the image.

- The Linux CI slice: `python -m unittest tests.test_launch_installer tests.test_capture_cli`
- The rest of `tests/`: `python -m pytest tests`
- Local smoke, HTML only, no third-party site: `python scripts/smoke.py`
- The API: `uvicorn vipercapture.main:app`. OpenAPI is `/openapi.json`.

## How you rate severity

- Critical: a render or job reaches a private, link-local, or metadata address, including through a redirect or DNS rebinding, or a request runs a command or writes an arbitrary host path.
- High: an auth or API-key bypass, a leak of proxy credentials or stored tokens, or a path traversal in an uploaded or returned archive or media file.
- Medium: the same outcomes only when a non-default setting is on (hosted mode off, scripts allowed) or the caller is already an authenticated project member who should not have that access.
- Low: crashes and resource exhaustion that do not cross a trust boundary.

## Anything to leave alone

- The sites being captured, and bugs that exist only in those pages.
- Third-party libraries, Playwright itself, and the vendored AutoConsent rules, unless ViperCapture's own use of them creates the issue.
- The hosted service. This repository is the open-source engine only.
- Dev tooling, CI, and choices made by this image.
- A self-hosted operator fetching their own internal URLs. Hosted mode is what must reject private targets; self-host mode allows them on purpose.
