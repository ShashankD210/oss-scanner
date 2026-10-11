# passkey threat model

## Scope and assets

This small project bridges WebAuthn requests from Chrome in WSL to the Windows WebAuthn API. No broad adoption claim is made. The Manifest V3 extension in `extension/background.js` uses `chrome.webAuthenticationProxy`, tab information and native messaging. The Windows-only Go host in `host/main.go` accepts length-prefixed JSON over standard input, calls `webauthn.dll` through pinned Go dependencies, and returns serialized credentials. The normal flow is webpage → browser proxy → extension → native messaging host → Windows authenticator/provider → relying party.

Assets include the binding among origin, relying-party ID, challenge and signed client data; correct user-verification and registration/assertion semantics; confidentiality of credential identifiers and user information; correct cancellation and request association; availability of the browser and host; and integrity of native-host registration and installed executables. Private authenticator keys are managed by Windows/authenticator providers, not deliberately stored by this repository. Claims about their protection require Windows/provider analysis beyond this Linux image.

## Adversarial inputs and boundaries

Webpages can initiate WebAuthn operations with attacker-chosen relying-party IDs, challenges, user fields, algorithms, credential IDs and extension options. Audit the extension's tab-origin inference and its fallback to `https://<rpId>`, including tab changes, invalid IDs, subdomain handling and cross-origin frames. The proxy API does not expose the requesting frame origin; cross-origin iframe support is a documented limitation, not a proven secure binding. The host creates the exact `clientDataJSON` signed by Windows and marks `crossOrigin` false. Review that boundary carefully rather than assuming the browser or authenticator validates everything passed through the proxy.

Treat incoming native messages, JSON/base64url fields, declared frame lengths, cancellation messages and native responses as adversarial. Examine oversized/truncated frames, allocation limits, malformed options, integer/timeout conversion, response validation, concurrency, port disconnects and request lifetimes. The host runs as the logged-in user, not in a new sandbox. A same-user process may launch it directly or alter registration, so native messaging is not an authentication boundary by itself. Credential listing/debug output and installer logs also expose potentially sensitive metadata.

Windows DLLs, Windows Hello and third-party passkey providers are external dependencies. Review the project's data conversions, COSE/CBOR handling and public-key export, but do not claim Linux testing validates Windows user consent, device behavior, cancellation, plugin-provider selection, or relying-party signature verification. Ignored transport hints and unsupported WebAuthn extensions (including PRF/hmac-secret/largeBlob) are deliberate scope limitations that should not silently change security semantics.

`install.sh` registers the native host, copies the executable onto the Windows filesystem, creates/reuses a non-exportable user code-signing key, imports a self-signed certificate into the CurrentUser Trusted Root store and signs the executable. This expands user trust to code signed by that key. Audit this script statically for paths, quoting, registration allowlists and trust implications, but NEVER execute it in the scanner environment. No certificate installation, Authenticode signing, browser-profile changes or real account authentication is authorized for this enrollment.

## Build and offline capabilities

The image copies the checkout to `/src`, retains all dependencies downloaded from `host/go.mod`/`host/go.sum` in `/go/pkg/mod`, and cross-compiles an unsigned Windows/amd64 `/src/dist/passkey-bridge.exe`. The pinned Go 1.26.9 toolchain satisfies the repository's Go 1.26.3 minimum, with automatic toolchain downloads disabled. Compilation uses `-mod=readonly`, retains Go debug information and uses the installer's Windows GUI subsystem flag without its stripping flags. Build failures are not suppressed.

The extension has no npm manifest, external JS dependencies or bundler: its JavaScript and JSON are the distributable sources. Node is included for syntax inspection and fixture-based JavaScript harnesses; syntax/JSON validation is not Chrome integration testing. The repository currently contains no test suite in the inspected host/extension directories. Source and dependency audits and Windows cross-compilation are available offline; no Windows executable can run in this Linux container. There is no Wine, Windows desktop, WSL interop, Chrome authentication-proxy integration, Windows DLL runtime, TPM, provider plugin or authenticator device provisioned here. Never invoke the host's `-test` or `-list` against real credentials/devices.

Offline commands:

```sh
node --check /src/extension/background.js
node -e 'JSON.parse(require("fs").readFileSync("/src/extension/manifest.json", "utf8"))'
cd /src/host
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 GOPROXY=off GOSUMDB=off GOTOOLCHAIN=local go build -mod=readonly -trimpath -ldflags="-H windowsgui" -o /tmp/passkey-bridge-offline.exe .
file /tmp/passkey-bridge-offline.exe
```

JavaScript behavior can be audited with a disposable Node harness that stubs the `chrome` event APIs/native port and injects synthetic options; this is not an existing upstream test or a substitute for Windows/Chrome execution. Changes to native Go code can be cross-compiled using the cached dependencies without network access. Do not use plain Linux `go test ./...`: the only host source is Windows build-tagged, and any future Windows test binary would also require a Windows runtime.

## Local verification commands

With the target repository separately checked out at `/tmp/passkey`, run from the scanner registry root:

```sh
docker build -f projects/pedropaulovc-passkey/Dockerfile -t oss-scanner-passkey /tmp/passkey
docker run --rm --network none oss-scanner-passkey bash -lc 'node --check extension/background.js && node -e '\''JSON.parse(require("fs").readFileSync("extension/manifest.json", "utf8"))'\'' && cd host && GOOS=windows GOARCH=amd64 go build -mod=readonly -trimpath -ldflags="-H windowsgui" -o /tmp/passkey-bridge-offline.exe . && file /tmp/passkey-bridge-offline.exe'
```

No host-home mounts or credentials are needed. These are proposed verification commands, not assertions that Docker or offline smoke checks have been run. Real Windows integration requires a separately authorized Windows environment and remains outside this Linux enrollment image.
