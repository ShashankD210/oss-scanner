# DeskMind threat model

## Project and scan boundary

DeskMind is an Apache-2.0 computer-use application for macOS. It observes
desktop applications, asks the user to clarify or approve actions, and operates
the desktop through a separate helper with Accessibility, Screen Recording,
and Automation permissions. Local inference is supported; current development
also includes an explicitly configured cloud-model path.

This enrolment covers **only https://github.com/deskmind-ai/deskmind**:
the native app, helper, shared policy/protocol code, protocol schemas and
checker, and build/runtime packaging scripts. The related Hands, Brain, Eyes,
Bench, and Harness repositories are separate projects and are not fetched or
enrolled by this Dockerfile. Their interfaces are useful context, but findings
must identify a defect in this repository. Third-party code, model weights,
and upstream model quality are not separate targets.

The security contact is security@deskmind.dev, published in the organization's
security policy:
https://github.com/deskmind-ai/.github/blob/main/SECURITY.md

## Inputs and trust boundaries

- Text, filenames, paths, window titles, accessibility labels, documents, and
  page content observed while completing a task may be attacker-controlled.
  They are data, not authorization to expand the user's requested task.
- Model responses and child-process output must be parsed defensively.
  Suggested actions and reported success do not themselves authorize sending,
  deleting, accessing another folder, or operating another application.
- User approval must remain associated with the intended question, operation,
  and run. Pay attention to stale replies, cancellation, process restarts,
  concurrent requests, and conversions between event formats.
- The main app communicates with the helper through JSON lines over a Unix
  socket under the user's Application Support directory. Local model servers
  use loopback HTTP with a per-install token. Examine the actual enforcement
  rather than assuming a loopback address or a socket is sufficient isolation.
- Downloaded model files and responses from a mirror are untrusted until the
  expected size/hash checks succeed. Examine paths, redirects, partial files,
  replacement behavior, and integrity checks.
- Screenshots, traces, recordings, tokens, and optional cloud credentials are
  sensitive. User-selected cloud processing is intended functionality;
  sending data or credentials beyond that selection is not.

Assume the OS and the installed, signed application are trusted. An attacker
may control content the app encounters or run an unprivileged local process.
A local finding should explain the additional capability gained (for example,
using the helper's TCC permissions), rather than merely demonstrating access
already available to that process. Administrator/root control or arbitrary
replacement of the installed application is not a prerequisite we seek to
defend against.

## Priorities and source map

1. Authorization and execution: `app/Helper/HandsHelper.swift`,
   `app/Helper/Runner.swift`, `app/Helper/HarnessRun.swift`,
   `app/Shared/Protocol.swift`, `app/Shared/HarnessEvents.swift`,
   `app/Shared/FolderPolicy.swift`, and `app/Main/AppScope.swift`.
   Look for confused-deputy behavior, scope escapes, unsafe subprocess
   arguments, stale approvals, and actions that continue after cancellation.
2. Local services and credentials: `app/Helper/ServerAuth.swift`,
   `app/Helper/BrainServer.swift`, `app/Helper/EyesServer.swift`, and
   `app/Main/CloudModel.swift`. Check server identity, token handling, and
   credential exposure across process and service boundaries.
3. Installation, downloads, and sensitive files:
   `app/Shared/HelperInstaller.swift`, `app/Shared/HelperLocation.swift`,
   `app/Main/ModelDownloader.swift`, `app/Shared/Models.swift`,
   `app/Main/History.swift`, and runtime/build scripts under `app/`.
   Consider symlinks, races, traversal, permissions, and unintended disclosure.
4. Protocol parsing and validation: `protocol/tools/check.py`,
   `protocol/schema/`, and `protocol/agent/operations.yaml`.
   Distinguish the standalone conformance checker from enforcement in a
   running application; a checker bug alone is not a demonstrated scope bypass.

## Offline environment and limitations

The complete checkout is at `/src`. The image includes Swift 6.2, Python 3,
PyYAML, jsonschema, git, and ripgrep. It builds these unmodified Foundation-only
sources into `/src/build/oss-scanner/libDeskMindAuditCore.so`, with debug
information: FileMention, FolderPolicy, Routing, DownloadSource,
RecordingCapture, and HarnessEvents.

Those declarations are internal Swift types. For a custom reproducer, compile
the relevant original `.swift` files together with the reproducer into one
module, using `swiftc -swift-version 5 -g -Onone -parse-as-library`.

Run the existing protocol tests and examples without network access:

```sh
cd /src
python3 -m unittest discover -s protocol/tests -v
python3 protocol/tools/check.py \
  protocol/examples/agent-request.json \
  protocol/examples/agent-request.list.json
```

The Docker build also runs those commands with its network disabled.

**This is a partial Linux audit environment, not a build of the macOS app.**
AppKit, SwiftUI, CoreGraphics, ScreenCaptureKit, ServiceManagement, TCC,
Apple Silicon MLX inference, and actual desktop interaction are unavailable.
The full `app/tests/DecisionTests.swift` suite and `app/tests/e2e/` require
macOS and are not claimed to run here. Model weights, the macOS runtime bundle,
and sibling repositories are intentionally not downloaded.

Review macOS-only code as source and label any OS-dependent exploit assumption
that cannot be validated in the container. Do not treat a Linux-specific
Foundation behavior as evidence of macOS behavior. The normal macOS build and
test instructions are in `app/README.md` and `app/build.sh`.

## Severity and reporting

- Critical/high: reproducible unauthorized execution with the helper's desktop
  permissions; meaningful escape from the confirmed application/folder/action
  scope; unintended disclosure of screen contents, private files, or secrets;
  bypass of required user approval with a concrete impact. Choose severity
  according to prerequisites and demonstrated reach.
- Medium: meaningful denial of service from untrusted input, or a narrower
  integrity/privacy failure without the impact above.
- Low: limited robustness issues or hardening opportunities. A poor model
  decision, incorrect confidence, or misleading "done" alone is not evidence
  of a security vulnerability.

Provide the affected commit, source locations, attacker capabilities, exact
input/reproducer, expected versus actual behavior, crossed trust boundary,
and a minimal suggested patch when possible. Separate reproduced results from
source-based hypotheses and explicitly identify required macOS validation.
Deduplicate by root cause. Use synthetic files and credentials in reproducers;
do not contact real recipients or exercise real user data.
