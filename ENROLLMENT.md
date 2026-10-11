# Wenjun's OSS Scanner enrollment

Eight repositories selected by the maintainer are prepared under `projects/`.
Their public security contact is **wenjunfoto@gmail.com**. Each directory contains
only `project.yaml`, `Dockerfile`, and `threat_model.md`, as required by upstream.
The Dockerfiles and threat models live here so enrollment does not require
changing the source repositories.

This repository configures Anthropic's hosted service. It does not contain the
scanner engine. Preparing a configuration, passing a build, or opening a PR does
not mean that a project has been accepted or scanned.

| Repository | Enrollment directory | Environment and review notes |
| --- | --- | --- |
| [ai-media-tools](https://github.com/wenjunii/ai-media-tools) | `projects/wenjunii-ai-media-tools` | Python collector and Node browser-search tests; live discovery, email and publication excluded. |
| [hometown-xr](https://github.com/wenjunii/hometown-xr) | `projects/wenjunii-hometown-xr` | CPU Python dependencies and synthetic archive tests; GPU/model weights and live crawling excluded. Dependency security review policy expired on 2026-08-31 and was not renewed. |
| [td-imagefx-library](https://github.com/wenjunii/td-imagefx-library) | `projects/wenjunii-td-imagefx-library` | Portable package, archive and updater tests; proprietary TouchDesigner rendering excluded. |
| [voice-to-visual-sdtd](https://github.com/wenjunii/voice-to-visual-sdtd) | `projects/wenjunii-voice-to-visual-sdtd` | Lightweight CI profile. Main is blocked by a Windows-only import; [draft compatibility PR #24](https://github.com/wenjunii/voice-to-visual-sdtd/pull/24) enables Linux checks. |
| [wenjun](https://github.com/wenjunii/wenjun) | `projects/wenjunii-wenjun` | Node 24, locked dependencies, tests and production build. **Hold: no repository license detected.** |
| [flexgpu-touchdesigner](https://github.com/wenjunii/flexgpu-touchdesigner) | `projects/wenjunii-flexgpu-touchdesigner` | Main needs [draft compatibility PR #22](https://github.com/wenjunii/flexgpu-touchdesigner/pull/22). Windows release gate, GPUs, licensed components and live hardware remain outside the Linux image. |
| [Lifeography](https://github.com/wenjunii/Lifeography) | `projects/wenjunii-lifeography` | Static site, browser tools and external script/style cache; HTTP smoke check, no existing full browser test suite. |
| [home-podcast](https://github.com/wenjunii/home-podcast) | `projects/wenjunii-home-podcast` | Portable Python tests and FFmpeg; paid generation, production ingestion and show controls excluded. |

## Verification

All eight enrollment configurations pass the upstream schema validator:

```sh
python3 tools/validate.py
```

The local receipts
in `.local/verification/` record source commit, configuration hashes, image ID,
exit codes and timestamps. Logs are in `.local/logs/`. Those files are ignored by
Git and must not be included in enrollment pull requests.

Verified with Linux amd64 Docker images on October 10, 2026:

| Project | Source checked | Offline verification |
| --- | --- | --- |
| ai-media-tools | [`f3a531e`](https://github.com/wenjunii/ai-media-tools/commit/f3a531e4a2de84f36d0a381e38b859d826bd5311) (`main`) | Passed: 182 Python tests, Node library tests and archive verification. |
| td-imagefx-library | [`5b9211f`](https://github.com/wenjunii/td-imagefx-library/commit/5b9211f1a9af6c62b548614a5fd5d006ad33f3e3) (`main`) | Passed: 353 tests plus repository verification. |
| home-podcast | [`2b2d876`](https://github.com/wenjunii/home-podcast/commit/2b2d87684b9f0670092f4f2908a49251e1cb9fa7) (`main`) | Passed: 133 tests. |
| wenjun | [`8f7e25a`](https://github.com/wenjunii/wenjun/commit/8f7e25a76f8bb8f159efec83e575d72f537fd318) (`main`) | Passed: content/image checks, Node tests and Vite production build. License decision still required. |
| Lifeography | [`b98a2d3`](https://github.com/wenjunii/Lifeography/commit/b98a2d3b02c1ef87aba0fa7414fdbfd0857778cd) (`main`) | Passed: upstream build, 20 cached script/style resources and offline HTTP smoke check. Full browser behavior is unverified. |
| hometown-xr | [`93ef094`](https://github.com/wenjunii/hometown-xr/commit/93ef0947dd462d209ce020ac020bf799e0553960) (`main`) | Passed: upstream build and 197 tests offline. Dependency-policy review is separately overdue. |
| voice-to-visual-sdtd | PR #24, `80818c8` | Passed: upstream build and 243 tests offline. GitHub Windows/Ubuntu CI also passed. Main remains blocked until the fix is merged. |
| flexgpu-touchdesigner | PR #22, `83c96d4` | Passed: upstream build and offline suite (471 tests run, 6 expected skips). GitHub Windows/Ubuntu CI also passed. Main remains blocked until the fix is merged. |

Candidate receipts are stored in `.local/candidate-verification/`, separately
from the main-branch receipts. Source commits were read from the finished images'
Git metadata, and configuration hashes match the published enrollment branches.
Six projects pass on `main`; the other two pass on their compatibility branches.
No scan or vulnerability verdict is claimed.

The current Mac has a project-local ARM64 Docker CLI, Buildx, Lima and Colima
under `.local/toolchain/`; downloaded release hashes are recorded in
`.local/toolchain-manifest.json`. The named `oss-scanner` Colima VM uses 4 CPUs,
8 GiB RAM, no host directory mounts, and Rosetta for Linux amd64 builds. It has no
automatic startup configured and is stopped after verification. The runtime is under `~/.colima/`; the Docker
configuration is local to this checkout. No GitHub or provider credentials are
passed into build or test containers.

For this prepared checkout:

```sh
source .local/env.sh
colima start oss-scanner --activate=false --ssh-config=false --mount none
python3 tools/check --no-shell wenjunii-td-imagefx-library
docker run --rm --network none --workdir /src \
  oss-scanner-check/built:wenjunii-td-imagefx-library \
  python -m unittest discover -v
colima stop oss-scanner
```

The local helper `.local/run_checks.py` accepts one or more repository names and
performs the upstream build followed by the documented offline checks. It does
not submit enrollments, publish code, or send email. On another machine, install
Git, Python with PyYAML, and a working Docker runtime before using `tools/check`.

## Submission requirements

Upstream requires **one project per PR**. The preparation branch contains all
eight; the following published branches each contain only their own three files:

| Project | Prepared enrollment branch |
| --- | --- |
| ai-media-tools | [enroll/wenjunii-ai-media-tools](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-ai-media-tools) |
| hometown-xr | [enroll/wenjunii-hometown-xr](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-hometown-xr) |
| td-imagefx-library | [enroll/wenjunii-td-imagefx-library](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-td-imagefx-library) |
| voice-to-visual-sdtd | [enroll/wenjunii-voice-to-visual-sdtd](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-voice-to-visual-sdtd) |
| wenjun | [enroll/wenjunii-wenjun](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-wenjun) |
| flexgpu-touchdesigner | [enroll/wenjunii-flexgpu-touchdesigner](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-flexgpu-touchdesigner) |
| Lifeography | [enroll/wenjunii-lifeography](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-lifeography) |
| home-podcast | [enroll/wenjunii-home-podcast](https://github.com/wenjunii/oss-scanner/tree/enroll/wenjunii-home-podcast) |

The local descriptions under `.local/pr-drafts/` preserve the upstream checklist.

Before an enrollment is submitted:

1. The maintainer must accept the [OSS Scanner terms](https://red.anthropic.com/oss-scanner/terms/).
2. The maintainer must confirm the inclusion criterion: an established project
   that users depend on, with impact on infrastructure or user security. A public
   repository or a passing Docker build is not evidence of eligibility. These
   drafts do not claim independently verified adoption.
3. `wenjun` needs an open-source license decision; no license was added or assumed.
4. The Docker build and appropriate offline checks must pass, with coverage
   limits disclosed. Static serving is not a full browser test, and mocked
   hardware tests are not end-to-end integration tests.
5. The voice and FlexGPU compatibility changes must be reviewed and merged before
   their `#main` enrollments are submitted. Candidate-branch validation is stored
   separately and does not imply that the current main branches pass.

Hometown's overdue dependency policy remains a separate security-review item.
Its source/tests can be built for auditing without renewing that exception or
changing the declared model dependencies. No security-safety verdict is implied
by successful container verification.

Anthropic manually reviews eligibility and maintainer status. Its first-PR bot
may also request the [Contributor License Agreement](CLA.md); the maintainer must
handle that attestation. The terms and inclusion boxes remain unchecked until
confirmed. No enrollment PR has been submitted and no scan is active.

See the upstream [README](README.md), [contribution instructions](CONTRIBUTING.md),
and [service FAQ](https://red.anthropic.com/oss-scanner/) for the current process.
