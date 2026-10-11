# Wenjun's OSS Scanner enrollment

All eight selected repositories were submitted to Anthropic on October 10, 2026,
using **wenjunfoto@gmail.com** as the public security contact. The enrollment PRs
are open and awaiting Anthropic's manual review; no project acceptance or active
scan is confirmed. This repository configures the hosted service and does not
contain the scanner engine.

The maintainer accepted the service terms, affirmed the inclusion criteria, and
authorized the one-time [CLA signature](https://github.com/anthropics/oss-scanner/pull/393#issuecomment-6104855478).
All eight PRs have passing CLA, terms and automated triage workflow checks.
Automated triage flags limited public adoption/criticality signals; passing
checks do not guarantee eligibility or acceptance.

## Submitted projects and verification

Each project passes the upstream schema validator and full `tools/check` build.
The results below are from the finished Linux amd64 images with Docker networking
disabled, using the recorded source commits on `main`.

| Repository | Enrollment PR | Tested main commit | Offline verification |
| --- | --- | --- | --- |
| [ai-media-tools](https://github.com/wenjunii/ai-media-tools) | [#393](https://github.com/anthropics/oss-scanner/pull/393) | [`f3a531e`](https://github.com/wenjunii/ai-media-tools/commit/f3a531e4a2de84f36d0a381e38b859d826bd5311) | 182 Python tests, Node library suite and archive verification. |
| [hometown-xr](https://github.com/wenjunii/hometown-xr) | [#394](https://github.com/anthropics/oss-scanner/pull/394) | [`93ef094`](https://github.com/wenjunii/hometown-xr/commit/93ef0947dd462d209ce020ac020bf799e0553960) | 197 tests and CLI help. Dependency-policy review remains overdue. |
| [td-imagefx-library](https://github.com/wenjunii/td-imagefx-library) | [#395](https://github.com/anthropics/oss-scanner/pull/395) | [`5b9211f`](https://github.com/wenjunii/td-imagefx-library/commit/5b9211f1a9af6c62b548614a5fd5d006ad33f3e3) | 353 tests and repository verification. |
| [voice-to-visual-sdtd](https://github.com/wenjunii/voice-to-visual-sdtd) | [#398](https://github.com/anthropics/oss-scanner/pull/398) | [`713b7a3`](https://github.com/wenjunii/voice-to-visual-sdtd/commit/713b7a353751d731628167e01511f823ebd41c20) | 243 tests; Linux and Windows CI passed. |
| [wenjun](https://github.com/wenjunii/wenjun) | [#400](https://github.com/anthropics/oss-scanner/pull/400) | [`0a61707`](https://github.com/wenjunii/wenjun/commit/0a617071967a41050711fec8b7e8c8d95274061b) | Content/image checks, 9 Node tests and Vite production build; website CI and audit passed. |
| [flexgpu-touchdesigner](https://github.com/wenjunii/flexgpu-touchdesigner) | [#399](https://github.com/anthropics/oss-scanner/pull/399) | [`18b7706`](https://github.com/wenjunii/flexgpu-touchdesigner/commit/18b7706f1567a6f76c3838d8bcae0b82b409d66f) | 471 tests run, 6 expected skips; Linux and Windows CI passed. |
| [Lifeography](https://github.com/wenjunii/Lifeography) | [#396](https://github.com/anthropics/oss-scanner/pull/396) | [`b98a2d3`](https://github.com/wenjunii/Lifeography/commit/b98a2d3b02c1ef87aba0fa7414fdbfd0857778cd) | 20 cached script/style resources and offline HTTP smoke check; full browser behavior unverified. |
| [home-podcast](https://github.com/wenjunii/home-podcast) | [#397](https://github.com/anthropics/oss-scanner/pull/397) | [`2b2d876`](https://github.com/wenjunii/home-podcast/commit/2b2d87684b9f0670092f4f2908a49251e1cb9fa7) | 133 tests; FFmpeg included. |

The configurations point to `#main`; the tested commits above identify this
verification snapshot. Each enrollment PR adds only `project.yaml`, `Dockerfile`
and `threat_model.md` in its own `projects/wenjunii-<name>/` directory.

## Source changes and coverage limits

- [Voice PR #24](https://github.com/wenjunii/voice-to-visual-sdtd/pull/24) is merged:
  Windows keyboard polling is optional on headless Linux hosts.
- [FlexGPU PR #22](https://github.com/wenjunii/flexgpu-touchdesigner/pull/22) is
  merged: Linux child-process liveness handles zombies and exit races. One local
  offline attempt hit a Rosetta emulator assertion; the same image passed on
  retry. Native Linux and Windows CI also passed.
- [Wenjun PR #6](https://github.com/wenjunii/wenjun/pull/6) is merged: the requested MIT license was added, and the sole affected lockfile dependency,
  `source-map-js`, was updated from 1.2.1 to 1.2.2 to resolve the website audit
  failure. GitHub detects the repository's MIT license.
- Hometown's dependency-security review policy expired on **2026-08-31**. The
  policy was not renewed or represented as passing; it remains a separate
  security-review item. Its source and synthetic tests can be built for auditing.
- Lifeography has no existing browser test suite. Cached scripts/styles and an
  offline HTTP smoke check do not establish full browser or external-API behavior.
- Validation did not use credentials, private model weights, paid generation,
  production ingestion, live email/publication, GPUs or licensed creative
  applications. Synthetic tests are not live TouchDesigner/audio/hardware tests.

Successful builds and tests are not vulnerability scans or a security verdict.

## Reproducing the local checks

Local receipts under `.local/verification/` record source commits read from image
Git metadata, configuration hashes, image IDs, exit codes and timestamps. Logs
are in `.local/logs/`; earlier attempts and candidate-branch results are preserved
separately. These files are ignored by Git and are not in the enrollment PRs.

The Mac uses project-local ARM64 Docker CLI, Buildx, Lima and Colima binaries in
`.local/toolchain/`, with release hashes recorded in `.local/toolchain-manifest.json`.
The `oss-scanner` Colima VM uses 4 CPUs, 8 GiB RAM, no host-directory mounts and
Rosetta for amd64 builds. It has no automatic startup configured and is stopped
after verification. Its runtime is under `~/.colima/`; Docker configuration stays
local to this checkout. No GitHub or provider credentials enter the containers.

```sh
source .local/env.sh
colima start oss-scanner --activate=false --ssh-config=false --mount none
python3 .local/run_checks.py td-imagefx-library
colima stop oss-scanner
```

The local helper runs only explicitly named projects. On another machine, install
Git, Python with PyYAML, and a working Docker runtime, then use the upstream tools:

```sh
python3 tools/validate.py
python3 tools/check --no-shell wenjunii-td-imagefx-library
docker run --rm --network none --workdir /src \
  oss-scanner-check/built:wenjunii-td-imagefx-library \
  python -m unittest discover -v
```

## Enrollment branches and next step

Anthropic requires one project per PR. The preparation branch contains all eight;
the individual fork branches below each add only their own three project files.

| Project | Enrollment branch |
| --- | --- |
| ai-media-tools | [enroll/wenjunii-ai-media-tools](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-ai-media-tools) |
| hometown-xr | [enroll/wenjunii-hometown-xr](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-hometown-xr) |
| td-imagefx-library | [enroll/wenjunii-td-imagefx-library](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-td-imagefx-library) |
| voice-to-visual-sdtd | [enroll/wenjunii-voice-to-visual-sdtd](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-voice-to-visual-sdtd) |
| wenjun | [enroll/wenjunii-wenjun](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-wenjun) |
| flexgpu-touchdesigner | [enroll/wenjunii-flexgpu-touchdesigner](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-flexgpu-touchdesigner) |
| Lifeography | [enroll/wenjunii-lifeography](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-lifeography) |
| home-podcast | [enroll/wenjunii-home-podcast](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-home-podcast) |

Anthropic must decide whether to accept each project. Service cadence and scan
execution are controlled by Anthropic, not by these configurations. No local
scheduler or self-hosted scanning service was created.

See the upstream [README](README.md), [contribution instructions](CONTRIBUTING.md),
and [service FAQ](https://red.anthropic.com/oss-scanner/) for the process.
