# Threat model

## Purpose and untrusted inputs

This project is a Cloudflare Worker implementing a Mount Rainier visibility bot. Scheduled invocations discover Space Needle PanoCam frames, assemble/crop/watermark images, classify them through the OpenAI Responses API, update Workers KV decision state, and optionally publish image posts to an AT Protocol service. The README describes a particular bot deployment; this enrollment makes no claim of broader adoption.

Treat remote camera HTTP status/headers/bodies, JPEG dimensions and pixels, timestamps and frame identifiers, classifier response JSON/text, and AT Protocol responses as untrusted. A compromised camera/CDN or malicious image can reach image decoding and panorama processing. Model descriptions are untrusted content, not authority to publish or disclose secrets. HTTP paths, query parameters (`at`, `frame`, `raw`, `refs`, `post`), form fields, and bearer tokens also cross a boundary. Bundled camera references under `assets/` are input data, not original project code.

## Assets and boundaries

- `src/index.ts` contains scheduled execution, HTTP routing, authentication, state transitions, HTML drafts, and forced publishing. Inspect authentication before work or sensitive output, timestamp/frame validation, HTML escaping, credential reflection in draft forms, and the interaction of the development token with `POSTING_ENABLED`.
- `src/frames.ts`, image-rendering code, and `src/panorama-alignment.ts` process remotely fetched frames. Inspect response limits, decoded dimensions, allocations, slice assembly, circular registration, crops, and rejection of unusable panoramas. Photon (`@cf-wasm/photon`) is a third-party WebAssembly image dependency; its installed code is retained for analysis under its own license.
- `src/classify.ts` constructs outbound image/model requests and parses classification. Inspect JSON/schema validation, bounds on text and confidence, request/response sizes, attacker-controlled descriptions entering HTML or public posts, and cost/amplification paths. Sending camera images to the configured model service is intentional; leaking credentials or unrelated sensitive state is not.
- `src/bsky.ts` handles app-password login, access tokens, blob upload, and record creation. Inspect credential scoping, header handling, error propagation, and publishing only through authorized paths with image attribution and alt text.
- `src/decide.ts` and KV state control transition and heartbeat publication. Inspect replay/duplicate posting, corrupt state, and forced-post authorization. Model uncertainty is not itself an authorization grant.
- Worker bindings include `STATE` and `ASSETS`; secret/configuration fields include `DEV_TOKEN`, `OPENAI_API_KEY`, `BSKY_IDENTIFIER`, `BSKY_APP_PASSWORD`, and model/service URLs. Operators configure these URLs; unauthenticated requesters must not gain authority to change them. Telemetry, errors, diagnostic pages, and model payloads must not expose secrets.

## Build and offline exercise

The Dockerfile copies the entire supplied public checkout to `/src` and installs the committed npm lockfile with development dependencies online. Its subsequent BuildKit `RUN --network=none` type-checks the project and bundles the actual Worker using Wrangler's deployment **dry run**, enforcing isolation even during this build step. It keeps source, tests, bundled assets, Photon WebAssembly, esbuild/workerd dependencies, and the bundle under `/src/.oss-scanner-build` for offline inspection. No cloud deployment or camera/model/social request is part of the build. No project test/build was executed while authoring this enrollment.

After the image is built, run in `/src` without network access:

```sh
npm test
npm run check
./node_modules/.bin/wrangler deploy --dry-run --outdir /src/.oss-scanner-build
```

These commands use already-installed local binaries; no `npx` download is required. The existing `vitest.config.ts` runs `src/**/*.test.ts` in Node. The frame-discovery, classifier, and Bluesky tests inject in-memory fetchers; the decision, panorama-alignment, and RFC 9457 problem tests operate on synthetic data. They require neither credentials nor live upstreams. Tests cover protocol request/response construction, decision gating, alignment algorithms, and structured errors, but do not constitute an end-to-end Workers runtime, actual Photon JPEG decoder, HTTP authentication, live KV, or real publishing test. Static review and additional scanner-driven offline inputs are appropriate for those gaps. No permanent test files have been added merely for enrollment.

Do not run `npm run dev`, `npm run eval`, camera-report scripts, provisioning/observability scripts, or deployment commands without `--dry-run`. Even `/check` and `/draft` perform camera and classifier requests; "dry-run" in those HTTP routes means no publication, not no network. The cron and `/post` routes can publish. Do not exercise these against any live service.

## Severity priorities

Unauthorized publication, development-token bypass, account credential/token exposure, or attacker-controlled secret exfiltration is high severity. Exploitable memory corruption in reachable image/WASM processing deserves high or critical severity according to impact and control. Remote-triggered resource exhaustion, repeated paid model requests, or uncontrolled duplicate posting can be medium or high depending on reachability, limits, and recovery. Ordinary model mistakes about visibility, aesthetic image alignment differences, or missing availability of an external service alone are not security findings; a concrete security boundary violation is needed.

## Safety and licensing limits

Do not supply real OpenAI/Bluesky/Cloudflare/Azure credentials, import `.dev.vars` or `.env` files, contact the camera, model API, AT Protocol endpoints, or telemetry services, provision resources, deploy, or publish. The checked-in service URLs, namespace IDs, assets, and infrastructure scripts are available for source audit only. Use synthetic data, injected fetchers, and local state when extending investigation.

The authorized root MIT license covers original project code. Space Needle camera frames/reference images and third-party dependencies are not relicensed; preserve the README's camera attribution and all dependency license notices. The repository is inspired by another visibility bot but describes an independent implementation, not copied original-service imagery.
