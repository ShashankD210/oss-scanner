# Threat model

## Scope and untrusted inputs

This small JavaScript extension builds Chrome/Chromium and Firefox variants from shared source and separate MV3 manifests. It intercepts page-world fetch responses on `https://bsky.app/*` to recover public posts hidden from third-party viewers by block relationships between other accounts. This enrollment makes no installed-user or adoption claim. It is not intended to grant access to private content or accounts.

Treat Bluesky XRPC JSON, post/embed/thread objects, AT-URIs, backlink-index results, remote error responses, page scripts and window messages as untrusted. `src/interceptor.js` rewrites responses and performs public post lookups; opt-in deep recovery also sends viewed post URIs to `constellation.microcosm.blue`. `src/bridge.js` connects extension storage/settings to the page world; `src/popup.js` manages local settings and Firefox optional permissions. Review both `manifests/` files and their different permission/runtime requirements.

## Boundaries and priorities

The MAIN-world interceptor shares a JavaScript environment with the page; page scripts can alter its exposed state and forge its configuration messages. Do not treat page-world state or a matching message namespace as privileged authentication. The isolated bridge, extension storage and popup have a different authority level; look for ways page messages or remote payloads could cross that boundary, gain browser permissions, or leak data beyond the intended public lookups.

Prioritize unauthorized network destinations or credential forwarding, private/account data exposure, bypass of opt-in disclosure/permission expectations, prototype pollution, unsafe handling of adversarial object shapes, content injection introduced by rewritten responses, recursive traversal/resource exhaustion, and unintended behavior on non-target origins. Deep recovery intentionally queries a third-party index only when enabled; test whether that consent boundary holds under page-level interference rather than assuming the UI alone enforces it. Restoring intentionally public records is the documented behavior; do not label it private-data access without evidence. Upstream moderation and deletion outcomes should not be replaced with fabricated data.

## Build products and offline exercise

The image installs locked npm development dependencies and builds both `dist/chrome` and `dist/firefox`, including both versioned ZIP archives. No transpilation or native project compilation is defined by the repository. Full Playwright Chromium and its headless system libraries are downloaded at build time for local browser experiments; distro Firefox ESR is also installed. No test or external account API request runs during construction.

The default Node test suite uses checked-in fixtures and a VM page-like context; popup/bridge tests exercise local browser API doubles. It needs no real browser, account, or network. With Docker networking disabled:

```sh
BSKY_NOT_BLOCKED_LIVE=0 npm test
npm run lint:firefox
```

The Chromium installation can be checked offline with a synthetic page (no extension loaded and no remote navigation):

```sh
node --input-type=module -e 'import { chromium } from "playwright"; const b = await chromium.launch({channel:"chromium",headless:true,args:["--no-sandbox"]}); try { const p = await b.newPage(); await p.setContent("<title>offline</title>"); if (await p.title() !== "offline") throw new Error("unexpected title"); } finally { await b.close(); }'
```

Browser launch uses `--no-sandbox` only in this disposable container smoke, as the repository's Chromium E2E does; do not infer production browser-sandbox coverage from it. For attack exploration, route every request locally and use synthetic fixtures, never a real account/profile.

## Limitations and prohibited exercises

Do not run `test:live`, `test:e2e`, `test:e2e:firefox`, `record-fixtures`, `store:screenshots` or `dev:firefox`: these workflows contact real Bluesky/backlink services. They cannot establish their live assumptions offline. Chromium's `document_start` ordering against the current live Bluesky bundle is not proved by the VM tests or synthetic startup smoke. Firefox's live E2E additionally needs geckodriver; it is deliberately not downloaded because that E2E is out of scope here. Firefox ESR is not proof of compatibility with every supported Firefox/Android version. No store signing/publishing, mobile runtime, native Windows/macOS browser, production deployment, credentials or real-account API calls are included.

## Severity guidance

Cross-origin/extension-authority escalation or disclosure of authenticated/private data from attacker-controlled payloads is high severity, potentially critical for broad exploitability. Bypassing opt-in third-party disclosure is a privacy/security finding with severity based on reachable data and attacker capability. Practical browser hangs or unbounded resource usage are normally medium. Rendering mismatches or a changed public-service assumption alone are not security vulnerabilities.
