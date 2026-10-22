# EntropyLab security audit scope

## Project and operating model

EntropyLab is an open-source, self-contained Bitcoin key and wallet calculator.
It transforms user-supplied entropy and wallet material into seeds, keys,
addresses, descriptors, Bitcoin Core wallet files and related outputs. Incorrect
results or exposure of private material can result in irreversible loss of funds.
The requested target is the maintained `rock` branch, not the deployment branch.

Start with `/src/SECURITY.md`, `/src/CONTRIBUTING.md`, and `/src/AGENTS.md`.
SECURITY.md documents the intended boundaries and existing limitations; those
statements are claims to check against implementation, not evidence of safety.
The recommended use is a verified downloaded `entropylab.html` on a dedicated
offline computer. The hosted site and its service worker are also in scope for
their actual advertised behavior, without assuming they establish an air gap.

## Inputs and trust boundaries

Treat externally supplied input as adversarial, including PSBTs, raw transactions,
descriptors, extended keys, mnemonics, payment URIs, scripts, QR/UR data, and
Journal/Key Manager/nonce-history files. Include malformed, oversized, duplicate,
inconsistent and misleading-but-well-formed inputs. Public metadata and imported
files can contain attacker-controlled display strings. A user may import public
transaction data while unrelated private keys are present in the same session.

The calculator must not invent wallet entropy, silently substitute a different
wallet/payment instruction, leak unrelated secrets into an export, or turn a
partial/unverifiable check into a claim that a transaction is safe. It does not
query a node or broadcast transactions, and externally supplied previous outputs
are not independently established as chain truth.

## Priority components

* `entropylab-wasm/`, `psbt-wasm/`, `vanity-wasm/`: cryptographic and serialization
  correctness, lengths/offsets, allocation and ownership, FFI boundaries, error
  paths, allocator/stack wiping, whole-memory retirement and worker termination.
* `src/js/`: BIP32/BIP39/BIP85 and address/descriptor derivation; Silent Payments;
  PSBT inspection/editing, sighash and nonce analysis; transaction and script
  parsing; Bitcoin Core wallet export; encrypted Journal/Key Manager import and
  export; deterministic encryption and its real security properties.
* Secret lifetime across asynchronous work, failed imports/derivations, key and
  network changes, reveal/conceal, modal closure, Lock, pagehide, back/forward-cache
  restoration and End Session. Look for stale callbacks that repopulate cleared
  secrets, retained mutable copies, or secrets crossing component boundaries.
* Browser code that handles attacker-controlled strings, DOM/HTML sinks,
  clipboard actions, exported files, storage, network requests, CSP and the
  hosted service worker. Include opt-out and translation detection logic while
  respecting its documented best-effort scope.
* Build scripts and dependency integration where project-owned behavior changes
  what is shipped. Pinned JavaScript packages used in production are part of the
  execution surface. Do not assume WASM or dependency APIs are used correctly.

## Exercising the project offline

The audit image keeps a full checkout at `/src`, installed npm dependencies,
the Rust toolchain and wasm32 target, all three Rust dependency caches, Firefox
and Chrome, and the newly built application and WASM artifacts. The final image
build stage rebuilds with networking disabled. That stage ran at commit
`17ed5e613c49fbfdb01cd9ed13446380af4623c1` through `tools/check entropylab`:
`npm ci --offline --ignore-scripts`, all three WASM builds, `npm run build` and
`npm run verify` completed with no network. Re-check it on the commit under audit
rather than trusting this record.

From `/src`:

```sh
npm run build:wasm
npm run build
npm run verify
npm run test:ci
npm run test:browser
```

The Dockerfile builds WASM, builds the HTML and verifies it. Run the test suites
above in the offline audit shell. At the commit above, `npm run test:ci` passed
offline (2,404 tests, no failures, 8 skipped). `npm run test:browser` passed in
both Firefox and Chrome/Chromium, 135 checks each, on a four-CPU machine; the
Firefox run is CPU-bound and its page-level waits expire on a two-CPU machine,
where Chrome/Chromium still completes. Give the browser suite at least four CPUs
and the machine to itself. `npm run reproduce` is available for independent
build comparisons. Browser fixtures are in `test/`; use the existing harness and
its loopback server or `file://` as appropriate, not an internet deployment.
`BROWSER_TEST_NO_SANDBOX=1` is an audit-container setting, not a recommended
security configuration for users. Rust crate source and build outputs remain
available for new offline tests and instrumentation.

Published Bitcoin/BIP vectors and public fixtures are appropriate test material.
Never use real private keys, funded wallets or actual user backups. Rebuild
generated artifacts from their sources when proposing patches.

## Severity and evidence

Prioritize reproducible silent key/address/descriptor or wallet-export errors
that can cause loss of funds; disclosure of private material across a supported
trust boundary; attacker-controlled script execution; and cryptographic failures
that permit recovery of secrets. Rate severity from a concrete scenario and its
prerequisites. Distinguish single-tab denial of service, misleading warnings and
loss of private material rather than assigning every parser issue high severity.

For each report provide the affected commit and paths, minimal public input or
proof of concept, exact offline command or browser steps, observed versus
expected behavior, impact and assumptions, and a focused regression test or
patch where possible. Use an independent standard, published vector or trusted
implementation to establish expected cryptographic/consensus results. Identify
unverified hypotheses, threat-model assumptions and possible duplicates.

## Documented limits and external systems

An already compromised operating system, browser or malicious extension can read
secrets. The page cannot erase OS swap/hibernation/crash dumps, remote clipboard
copies, or all browser-managed immutable strings/BigInts/DOM editing histories.
Translation detection is after the fact; opt-outs depend on browser/extension
cooperation. Low-entropy user inputs and an empty Journal password have explicit
limitations. Existing PSBT analysis has documented search/type/resource bounds.

Do not report these limitations alone as novel exploits. Do report a concrete
implementation defect that exceeds an admitted boundary, contradicts a guarantee,
or makes an intended check silently succeed without meeting its prerequisites.
Do not attack live services, unrelated repositories, real users or wallets. Keep
vulnerability details private and send them to the configured security contact.
