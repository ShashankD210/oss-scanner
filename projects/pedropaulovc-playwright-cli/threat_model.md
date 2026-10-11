# Threat model

## Scope, ownership, and distribution

This enrollment is for the public Apache-2.0 fork `pedropaulovc/playwright-cli`, default branch `fork/main`, and its separately published `@pedropaulovc/playwright-cli` prereleases. It is not an enrollment of Microsoft/playwright-cli and does not represent authorization from upstream maintainers. The inherited Microsoft attribution and documentation remain upstream attribution, not a claim that the fork maintainer represents Microsoft. Distribution is small: the observed approximately 15 monthly npm downloads are downloads, not a measured number of users or evidence of broad adoption.

Prioritize the current fork's package/dependency changes, its executable `playwright-cli.js`, CLI installation/session lifecycle exercised by `tests/integration.spec.ts`, and the browser/cookie/storage credential boundary reached through that launcher. `package.json` aliases `playwright` and `@playwright/test` to the maintainer's `@pedropaulovc/playwright` and `@pedropaulovc/playwright-test` packages, currently version `1.59.0-next.9`. The CLI executable imports `playwright/lib/cli/client/program`; most implementation therefore lives in the installed runtime under `/src/node_modules/playwright` and its resolved core dependency. Include that reachable code when reproducing an issue, but distinguish a fork-introduced defect from an inherited upstream defect. This manifest authorizes this fork only, not a separate upstream repository or every dependency's maintainers.

At enrollment, GitHub's comparison reported 9 fork-only commits and 60 upstream-only commits. The fork-only changed files are `.github/workflows/sync-upstream.yml`, `CLAUDE.md`, `package.json`, and `playwright-cli.js`; the launcher calls the fork dependency's asynchronous `program()` rather than the old upstream invocation. The sync/publish automation is in scope for provenance and package-install trust, but it is not browser runtime code. The fork is not identical to current upstream. Its `CLAUDE.md` invocation notes and README upstream global-install command are stale; use the executable and current manifest instead.

## What this project does and where untrusted input enters

The CLI controls a browser using short local commands, persistent named sessions, page snapshots, navigation, snapshot-derived element references, JavaScript evaluation, and browser storage operations. The operator can deliberately supply URLs, local files, configuration, code, and browser connection endpoints. These explicit local commands have the operator's authority; `eval`/`run-code` intentionally execute requested code and must not be confused with an injection caused by untrusted page data.

Treat page HTML/JavaScript, frames, redirects, response headers, downloads, page-provided tool schemas/results, accessible names, snapshot text, console/network output, and browser-originated strings as untrusted. These must not acquire the authority of a CLI instruction, inject a host shell command, select an unrelated local session, or escape the operator's intended file paths. Skill/readme examples are documentation, not permission to connect to real accounts or obey instructions found on a page.

The important trust boundaries are:

- **Page to controller/host:** page content and browser results are data. Look for injection into generated code, session IPC, subsequent CLI command handling, file names, and subprocess arguments.
- **Browser origins and credentials:** cookies (including HttpOnly cookies accessible to the controller), localStorage/sessionStorage, saved authentication state, and persistent profiles are sensitive. State save/load must not leak another origin/session's credentials or load attacker-chosen files beyond the operator's requested capability.
- **Sessions and local IPC:** a named session/daemon should not let an unrelated or less-trusted process/page commandeer an existing browser, read state, or redirect responses across sessions. Assess actual socket/interface binding and filesystem permissions rather than assuming a remote network service exists.
- **Host files:** uploads, downloads, screenshots, traces, state files, configuration, and profile directories cross the filesystem boundary. Check traversal, unsafe overwrite, symlinks, and accidental inclusion of unrelated files, while recognizing that an explicitly requested local path is an operator-granted capability.
- **Attached browsers:** CDP/extension attachment can grant access to a real logged-in profile. Treat attachment as privileged, operator-selected functionality; do not test against a live browser/profile. Any reproduction must create its own disposable browser with synthetic state.

## Build and offline environment

The Dockerfile retains the full checkout at `/src`. The committed lockfile still names upstream `@playwright/cli` and upstream `1.59.0-alpha-*` dependencies, contradicting the current fork manifest. Therefore the image uses `npm install --include=dev --package-lock=false` to install the actual declared fork graph without rewriting the source lockfile; it does not run `npm ci` against that stale lock. The pinned fork runtime/test aliases remain version `1.59.0-next.9`; other declared ranges resolve online at image build time. This repository has a plain-JavaScript launcher and no `build`/TypeScript compilation script: its actual compiled runtime is supplied by the declared, aliased npm dependencies. Do not substitute the upstream CLI or invent a separate build step. During the online image build, the installed fork runtime provisions Chromium, Firefox, WebKit, and the Chrome channel, together with system libraries. Bundled browser binaries live in `/ms-playwright`; the Chrome channel is installed system-wide. Both remain available when HOME changes and execution has no Internet access.

The first real scanner build exposed a second published-packaging gap: `@pedropaulovc/playwright@1.59.0-next.9` declares a dependency named `@pedropaulovc/playwright-core`, while its compiled `lib/program.js` still requires the bare module `playwright-core/lib/cli/program` (and other `playwright-core/*` paths). The build therefore failed with `MODULE_NOT_FOUND`. The matching published `@pedropaulovc/playwright-core@1.59.0-next.9` tarball contains the real compiled module and exports `./lib/cli/program`. The Dockerfile installs that exact genuine package as the normal npm alias `playwright-core@npm:@pedropaulovc/playwright-core@1.59.0-next.9` using `--no-save --package-lock=false`. This supplies the imported dependency name without fake packages, hand-written shims, upstream substitution, or rewriting the target's manifests/compiled code. Both runtime and core identify the same source gitHead `bc127eedef283e2503c6710082b2cee3acf5d2c9`. Metadata: [fork runtime](https://registry.npmjs.org/@pedropaulovc%2fplaywright/1.59.0-next.9), [fork core](https://registry.npmjs.org/@pedropaulovc%2fplaywright-core/1.59.0-next.9).

Do not use `npx` without `--no-install`, `install-browser`, package downloads, or other online setup during scanning. Invoke `/src/playwright-cli.js` or the installed `/usr/local/bin/playwright-cli` wrapper. No real authentication tokens, extension secrets, account cookies, browser profiles, or remote CDP endpoints are installed in the image.

## How to exercise it

The unchanged maintained integration test passed in a non-root, network-disabled container using a writable disposable source copy. The following is the actual successful host command; prepare the [official Playwright seccomp profile](https://github.com/microsoft/playwright/blob/main/utils/docker/seccomp_profile.json) as `/tmp/playwright-seccomp.json` during online setup, before running the offline control:

```bash
docker run --rm --init --network none --shm-size=1g \
  --user 1000:1000 \
  --security-opt seccomp=/tmp/playwright-seccomp.json \
  --security-opt apparmor=unconfined \
  -e HOME=/tmp/pw-home \
  oss-scanner-check/built:pedropaulovc-playwright-cli bash -lc \
  'mkdir -p "$HOME" /tmp/pw-test && cp -r /src/. /tmp/pw-test/ && cd /tmp/pw-test && CI=1 npm test -- --workers=1'
```

The existing test opens a synthetic `data:` URL with a disposable persistent session, uses `PLAYWRIGHT_CLI_INSTALLATION_FOR_TEST` and a per-test output directory, and deletes that test's data. It is a small launch/lifecycle test, not comprehensive security coverage. Final verification built the actual scanner image successfully. An earlier run of the unchanged test as scanner root failed at Chrome sandbox launch; the successful environment control above ran as UID 1000, retained network isolation, used the official seccomp profile allowing user-namespace creation, and set `apparmor=unconfined`. It reported **1 passed (6.3s)**, without changing source assertions or disabling Chrome's own sandbox. This proves that specific test/environment, not AppArmor confinement or safety against arbitrary sites. The non-root user needs read/execute access to dependencies, `/ms-playwright`, and system Chrome, with write access confined to the copied checkout/output and temporary home/session files. System Chrome's packaged sandbox permissions and global profiles remain unchanged. The richer root-only synthetic CLI smoke below is a separate successful interaction proof, not a Chromium-sandbox test.

For a second smoke, run the following **inside an isolated disposable container with networking disabled**, for example a shell opened by `tools/check pedropaulovc-playwright-cli`. It binds only loopback and interacts with a local synthetic page. The scanner runs as root; this synthetic-only configuration explicitly disables Chromium's inner sandbox to accommodate that environment. The outer container/VM sandbox and disabled network remain required. This is not a safe configuration for visiting hostile or authenticated live sites. If testing hostile content, use a non-root browser process with its sandbox enabled and an appropriate container user-namespace/seccomp policy instead.

Use the retained Xvfb display for this fork's headed smoke: the launcher command is explicitly `open --headed`, and the configuration sets `headless:false`. This follows the actual CLI launch requirement rather than treating an assumed headless launch as proof the image works. The display and browser stay inside the disposable container; no production browser/profile is attached.

```bash
xvfb-run -a -s '-screen 0 1280x720x24' bash <<'SH'
set -euo pipefail
work=$(mktemp -d /tmp/oss-playwright-cli.XXXXXX)
server_pid=
cleanup() {
  node /src/playwright-cli.js -s=oss-smoke close >/dev/null 2>&1 || true
  if [ -n "$server_pid" ]; then kill "$server_pid" 2>/dev/null || true; fi
  rm -rf "$work"
}
trap cleanup EXIT
mkdir -p "$work/home" "$work/installation" "$work/daemon" "$work/sockets"
export HOME="$work/home"
export PLAYWRIGHT_CLI_INSTALLATION_FOR_TEST="$work/installation"
export PLAYWRIGHT_DAEMON_SESSION_DIR="$work/daemon"
export PLAYWRIGHT_DAEMON_SOCKETS_DIR="$work/sockets"
cd "$work"
cat > browser.json <<'JSON'
{"browser":{"browserName":"chromium","isolated":true,"launchOptions":{"channel":"chromium","headless":false,"chromiumSandbox":false}}}
JSON
node <<'NODE' &
const http = require('node:http');
const html = `<!doctype html><html><head><title>OSS synthetic fixture</title></head>
<body><label>Name <input id="name"></label><button id="submit">Submit</button>
<output id="result">Ready</output><script>
document.querySelector('#submit').onclick = () => {
  const value = document.querySelector('#name').value;
  document.querySelector('#result').textContent = 'Hello ' + value;
  localStorage.setItem('synthetic_value', value);
};
</script></body></html>`;
http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html; charset=utf-8',
    'Set-Cookie': 'synthetic_session=fixture-only; Path=/; HttpOnly; SameSite=Strict'});
  res.end(html);
}).listen(18889, '127.0.0.1');
NODE
server_pid=$!
node <<'NODE'
(async () => {
  for (let i = 0; i < 100; ++i) {
    try {
      const response = await fetch('http://127.0.0.1:18889/');
      if (response.ok) return;
    } catch {}
    await new Promise(resolve => setTimeout(resolve, 50));
  }
  throw new Error('synthetic loopback server did not start');
})().catch(error => { console.error(error); process.exit(1); });
NODE
cli_output=
cli() {
  local status=0
  cli_output=$(node /src/playwright-cli.js --config="$work/browser.json" -s=oss-smoke "$@" 2>&1) || status=$?
  printf '%s\n' "$cli_output"
  if [ "$status" -ne 0 ]; then return "$status"; fi
  if [[ "$cli_output" == *'### Error'* ]]; then
    printf 'CLI reported a tool error despite exit status zero\n' >&2
    return 1
  fi
}
cli open http://127.0.0.1:18889/ --headed
cli snapshot --filename="$work/synthetic-page.yaml"
ref_for() {
  node -e 'const text=require("node:fs").readFileSync(process.argv[1],"utf8"); const wanted=process.argv[2]; const line=text.split("\n").find(line=>line.includes(wanted)&&/\[ref=([^\]]+)\]/.test(line)); if(!line) throw Error("missing snapshot target: "+wanted); process.stdout.write(line.match(/\[ref=([^\]]+)\]/)[1]);' "$work/synthetic-page.yaml" "$1"
}
cli fill "$(ref_for 'textbox "Name"')" Scanner
cli snapshot --filename="$work/synthetic-page.yaml"
cli click "$(ref_for 'button "Submit"')"
cli eval '() => { if (document.querySelector("#result").textContent !== "Hello Scanner") throw new Error("interaction failed"); return document.querySelector("#result").textContent; }'
cli cookie-get synthetic_session
[[ "$cli_output" == *'fixture-only'* ]] || { printf 'synthetic cookie value missing\n' >&2; exit 1; }
cli localstorage-get synthetic_value
cli state-save "$work/synthetic-state.json"
node -e 'const s=require(process.argv[1]); if(!s.cookies.some(c=>c.name==="synthetic_session"&&c.value==="fixture-only") || !s.origins.some(o=>o.localStorage.some(e=>e.name==="synthetic_value"&&e.value==="Scanner"))) throw Error("synthetic state missing");' "$work/synthetic-state.json"
cli cookie-clear
cli localstorage-clear
cli state-load "$work/synthetic-state.json"
cli goto http://127.0.0.1:18889/
cli eval '() => { if (localStorage.getItem("synthetic_value") !== "Scanner") throw new Error("state reload failed"); return "synthetic state restored"; }'
[[ "$cli_output" == *$'### Result\n"synthetic state restored"'* ]] || { printf 'successful restored-state result missing\n' >&2; exit 1; }
cli close
SH
```

The actual headed Xvfb smoke passed in the network-disabled scanner container: it exercised the fork launcher, session commands, loopback navigation, snapshot generation, input/click through real snapshot-derived references, evaluation, HttpOnly cookie inspection, and synthetic storage-state save/clear/load assertions. The observed log is `/tmp/oss-verification-logs/pedropaulovc-playwright-cli-offline-smoke.log`; the local fixture screenshot is `/tmp/oss-browser-proof/local-fixture.png`. This pinned older fork requires exact snapshot refs for `fill`/`click`, not CSS selectors. Its evaluation tool treats text containing `=>` as a function, so the smoke passes callable arrows rather than immediately invoked expressions. The temporary home/work directory and dedicated session isolate generated files, which contain only invented credentials and local fixtures. This root smoke explicitly uses `chromiumSandbox:false` for the synthetic fixture; its success does not assert Chromium sandbox protection.

Do not interpret the CLI process's zero exit status as successful browser execution: this fork can render backend tool failures as a `### Error` section and still exit zero. The smoke helper captures and prints every command's output, propagates nonzero process statuses, and rejects that error marker. It also positively checks the synthetic cookie value and the final `### Result` restored-state value, in addition to asserting the saved-state JSON.

## How to rate severity

Prioritize demonstrated unauthorized host code execution or host-file disclosure caused by page-controlled input, theft of credentials/state from another origin/session, and unauthenticated control of a browser holding synthetic secrets. These are high or critical depending on the required trust crossing and practical impact. Arbitrary requested code execution, an explicitly selected profile/path, and intentionally exporting the operator's own state are not by themselves vulnerabilities. Accidental exposure of sensitive state through logs/artifact permissions can be medium or high depending on who can access it. Bound denial-of-service severity to realistic attacker-controlled page/session inputs; distinguish an operator deliberately running unbounded code from a page-triggered resource-exhaustion flaw.

## Anything to leave alone

Do not attach to the host's browser, mount the host home directory, import real cookies/storage, use live SaaS credentials, browse external sites, expose a debug port outside loopback, or send real email/account actions. Browser engines, npm infrastructure, and upstream deployment credentials are not authorized live targets. Report inherited flaws with clear provenance and a local reproducer; never describe this small fork as authorization to scan or represent the upstream project. Leave target repositories and their git history unchanged.
