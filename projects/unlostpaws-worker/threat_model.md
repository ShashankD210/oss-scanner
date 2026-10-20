# Threat model: UnLostPaws vision worker

Async Python 3.12 service. It reads jobs from a Redis Stream, downloads pet photos, runs quality, NSFW
safety, hashing and SigLIP2 embedding stages (torch or ONNX Runtime), and POSTs results to a webhook on the
UnLostPaws Next.js app. It exposes no inbound network port. The web app is a separate project.

## What this project does and where untrusted input enters
- Image bytes are the main untrusted input. Users upload photos to the web app; the worker fetches them from
  URLs in the job and decodes them with Pillow (`app/pipeline/download.py`, `app/pipeline/stages/`).
  Decoder bombs, malformed files, huge dimensions and memory exhaustion are the top concern.
- Job payloads from Redis (`app/queue/consumer.py`, `app/schemas/job.py`): `imageUrls`, `webhookUrl`, ids, `petType`.
  Redis is a trusted, operator-controlled queue, but the worker must still validate the payload shape and survive junk.
- HTTP responses from the image host and the webhook (`httpx`, redirects followed).
- Model artifacts from Hugging Face Hub and the local cache (`app/models/`): loading safety, checksum handling,
  unsafe deserialization.
- Environment configuration (`app/config/`): operator-controlled and trusted.

## Components that matter most
- `app/pipeline/download.py` and `app/pipeline/stages/`: size limits, `Image.MAX_IMAGE_PIXELS`, content-length caps,
  redirect handling, SSRF from `imageUrls` and `webhookUrl`, `rewrite_local_url` in `app/utils/url.py`.
- `app/callback/client.py`: what is sent, where, and with what credentials. The webhook URL carries a secret token
  supplied by the web app, so check that the worker never logs or leaks it.
- `app/queue/consumer.py`: retry, DLQ and poison-message handling; a single bad job must not wedge the consumer.
- `app/models/` (artifact download, ONNX export, registry, manifest): `torch.load`, pickle, `trust_remote_code`,
  path traversal in cache paths, hash verification of downloaded weights.

## Less important or out of scope
- `tools/` (CLI, benchmark, doctor, export, `run.sh`): operator-run developer tooling.
- Dependencies (torch, transformers, onnxruntime) except where this code uses them unsafely.
- Model quality, NSFW recall or false positives. Those are product issues, not security issues.

## How to exercise it
- `pytest` runs offline; `tests/unit/conftest.py` mocks torch, onnxruntime and transformers. Models are NOT
  downloaded, so no real inference runs.
- Reproduce download and decode issues by calling the functions in `app/pipeline/download.py` directly with
  crafted bytes or a local HTTP server. Reproduce queue issues with the mocks used in `tests/unit/test_consumer.py`.

## How I rate severity
- Critical: remote code execution, for example unsafe deserialization of a model file or job payload, or
  command injection.
- High: SSRF that an ordinary web-app user can steer to internal services; an uncapped download or decode that
  one crafted upload can use to crash the worker repeatedly; leaking the webhook secret token.
- Medium: decompression bombs or large images with bounded impact, missing validation that causes a worker crash
  recovered on restart, unbounded redirects, TLS verification disabled where it should not be.
- Low: verbose logging, missing timeouts on non-critical paths.
- Anything that requires write access to Redis or the environment is Low, because those are operator-trusted.

## Anything to leave alone
- `ssl_cert_reqs=None` for `rediss://` in `app/queue/consumer.py`: deliberate, for managed Redis such as Upstash.
- `rewrite_local_url` mapping localhost to `host.docker.internal`: development convenience.

## Reports and patches
- Reproducer as a short Python snippet or pytest test, with the exact file and line.
- Patches should be minimal, keep the existing style (ruff format), and not add dependencies.
