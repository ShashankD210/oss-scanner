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

Do not run `npm run dev`, `npm run eval`, camera-report scripts, provisioning/observability scripts, or deployment commands without `--dry-run`. Even `/check` and `/draft` perform camera and classifier requests; "dry-run" in those HTTP routes means no publication, not no network. The cron and `/post` routes can publish. Do not exercise these against any live service. Research tools under `scripts/`, including Python image/alignment tooling using `cv2`, are retained for source audit only: their Python dependencies are not pinned or installed in this image. The evaluator, camera-report, `ai-*`, and provisioning scripts must not be run against live endpoints.

### Actual local Worker and JPEG smoke

For a real program smoke beyond mocked unit tests, save the following throwaway harness as `/tmp/oss-mountain-smoke.mjs`. It launches the unchanged Wrangler bundle through the installed locked Miniflare/workerd runtime, makes genuine loopback HTTP requests with synthetic bindings and local KV, and rejects every outbound Worker request. It additionally executes the actual Photon WebAssembly JPEG encoder/decoder using synthetic pixels through the supported Node entrypoint. It does not exercise the full image assembly pipeline, production KV, or publishing.

This harness was verified offline against the built image: all 14 real HTTP checks passed with zero outbound attempts, and Photon encoded and decoded a synthetic 16×16 JPEG (715 bytes). The runtime uses an explicit ESModule/CompiledWasm manifest including the actual emitted Photon file, not unsupported Miniflare converter module rules.

Run it inside the scanner's offline image with `node /tmp/oss-mountain-smoke.mjs`, or from the host with:

```sh
docker run --rm --network none \
  -v /tmp/oss-mountain-smoke.mjs:/tmp/oss-mountain-smoke.mjs:ro \
  oss-scanner-bsky-mountain-out node /tmp/oss-mountain-smoke.mjs
```

No secrets or live endpoint configuration should be supplied. The tested paths stop before camera/model/social activity; attempting any such fetch fails the harness.

```js
import assert from 'node:assert/strict';
import { createRequire } from 'node:module';
import { readdir } from 'node:fs/promises';
import { pathToFileURL } from 'node:url';

const require = createRequire('/src/package.json');
const { Miniflare, convertV4MiniflareOptions } = await import(pathToFileURL(require.resolve('miniflare')).href);
const { PhotonImage } = await import(pathToFileURL(require.resolve('@cf-wasm/photon/node')).href);
const bundleDir = '/src/.oss-scanner-build';
const files = await readdir(bundleDir);
assert(files.includes('index.js'), `Expected real Wrangler bundle index.js; found ${files.join(', ')}`);
const wasmFiles = files.filter((file) => file.endsWith('.wasm'));
assert(wasmFiles.length > 0, 'Actual Worker bundle must retain Photon WASM');
const modules = [
  { type: 'ESModule', path: `${bundleDir}/index.js` },
  ...wasmFiles.map((file) => ({ type: 'CompiledWasm', path: `${bundleDir}/${file}` })),
];
let outboundAttempts = 0;
const token = 'local-smoke-only-not-a-secret';
const mf = new Miniflare(convertV4MiniflareOptions({
  name: 'mountain-offline',
  rootPath: '/src',
  cf: false,
  telemetry: { enabled: false },
  host: '127.0.0.1',
  port: 0,
  modules,
  modulesRoot: bundleDir,
  compatibilityDate: '2026-08-17',
  compatibilityFlags: ['nodejs_compat'],
  kvNamespaces: ['STATE'],
  resourcePersistencePath: '/tmp/mountain-smoke-state',
  bindings: {
    DEV_TOKEN: token,
    POSTING_ENABLED: 'false',
    OPENAI_API_KEY: '',
    BSKY_IDENTIFIER: '',
    BSKY_APP_PASSWORD: '',
    OPENAI_API_URL: 'https://model.invalid',
    BSKY_SERVICE_URL: 'https://social.invalid',
    MODEL_ID: 'offline-test',
    IMAGE_MODE: 'stitched',
    CLASSIFIER_REASONING_EFFORT: 'medium',
  },
  outboundService: async (request) => {
    outboundAttempts++;
    throw new Error(`Forbidden outbound request in local runtime smoke: ${request.url}`);
  },
}));
let checks = 0;
try {
  const base = await mf.ready;
  assert(['127.0.0.1', 'localhost'].includes(base.hostname));
  async function request(path, init = {}, expectedStatus, contentType = 'application/problem+json') {
    const response = await fetch(new URL(path, base), init);
    assert.equal(response.status, expectedStatus, `${init.method ?? 'GET'} ${path}`);
    assert(response.headers.get('content-type')?.includes(contentType), path);
    checks++;
    return response;
  }
  const auth = { Authorization: `Bearer ${token}` };
  await request('/status', {}, 401);
  await request('/check', { headers: { Authorization: 'Bearer incorrect' } }, 401);
  await request('/post', {}, 405);
  await request('/post', { method: 'POST' }, 401);
  await request('/check', { method: 'POST', headers: auth }, 405);
  await request('/does-not-exist', { headers: auth }, 404);
  await request('/post', { method: 'POST', headers: auth }, 400);
  await request('/post?at=not-a-date', { method: 'POST', headers: auth }, 400);
  await request('/post?at=2026-08-17T12:00:00Z', { method: 'POST', headers: auth }, 503);
  await request('/draft', { method: 'PUT' }, 405, 'text/html');
  const login = await request('/draft?at=%3Cscript%3Eevil%3C%2Fscript%3E', {}, 200, 'text/html');
  const loginBody = await login.text();
  assert(loginBody.includes('&lt;script&gt;evil&lt;/script&gt;'));
  assert(!loginBody.includes('<script>evil</script>'));
  await request('/draft', { headers: auth }, 400, 'text/html');
  await request('/draft?at=not-a-date', { headers: auth }, 400, 'text/html');
  const state = await request('/status', { headers: auth }, 200, 'application/json');
  assert.equal(typeof await state.json(), 'object');
  assert.equal(outboundAttempts, 0, 'All tested paths must stop before any outbound request');
  console.log(`PASS actual unchanged Worker: ${checks} local HTTP auth/method/input/state checks; zero outbound calls`);
} finally {
  await mf.dispose();
}

const width = 16;
const height = 16;
const pixels = new Uint8Array(width * height * 4);
for (let y = 0; y < height; y++) {
  for (let x = 0; x < width; x++) {
    const i = (y * width + x) * 4;
    pixels[i] = x * 16;
    pixels[i + 1] = y * 16;
    pixels[i + 2] = 128;
    pixels[i + 3] = 255;
  }
}
const image = new PhotonImage(pixels, width, height);
let decoded;
try {
  const jpeg = image.get_bytes_jpeg(90);
  assert(jpeg.length > 4);
  assert.equal(jpeg[0], 0xff);
  assert.equal(jpeg[1], 0xd8);
  decoded = PhotonImage.new_from_byteslice(jpeg);
  assert.equal(decoded.get_width(), width);
  assert.equal(decoded.get_height(), height);
  assert.equal(decoded.get_raw_pixels().length, width * height * 4);
  console.log(`PASS actual Photon WASM: synthetic ${width}x${height} JPEG encoded and decoded (${jpeg.length} bytes)`);
} finally {
  decoded?.free();
  image.free();
}
```

## Severity priorities

Unauthorized publication, development-token bypass, account credential/token exposure, or attacker-controlled secret exfiltration is high severity. Exploitable memory corruption in reachable image/WASM processing deserves high or critical severity according to impact and control. Remote-triggered resource exhaustion, repeated paid model requests, or uncontrolled duplicate posting can be medium or high depending on reachability, limits, and recovery. Ordinary model mistakes about visibility, aesthetic image alignment differences, or missing availability of an external service alone are not security findings; a concrete security boundary violation is needed.

## Safety and licensing limits

Do not supply real OpenAI/Bluesky/Cloudflare/Azure credentials, import `.dev.vars` or `.env` files, contact the camera, model API, AT Protocol endpoints, or telemetry services, provision resources, deploy, or publish. The checked-in service URLs, namespace IDs, assets, and infrastructure scripts are available for source audit only. Use synthetic data, injected fetchers, and local state when extending investigation.

The authorized root MIT grant covers original project software, not all repository content. Space Needle camera frames/reference images, derived camera imagery, and third-party dependencies are expressly excluded and are not relicensed; preserve the README's camera attribution and all dependency license notices. Camera imagery redistribution rights are not established by this enrollment. The repository is inspired by another visibility bot but describes an independent implementation, not copied original-service imagery. GitHub's automatic license metadata reports SPDX `NOASSERTION` for the scoped root LICENSE; this is a manual-review caveat, not an automatic MIT classification or a grant of rights over excluded materials.
