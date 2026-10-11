# syntax=docker/dockerfile:1
# Build the maintained dogfood checkout, not an installed OMP or upstream release.
FROM debian:bookworm

ARG BUN_VERSION=1.4.2
ARG BAZELISK_VERSION=1.29.0
ENV DEBIAN_FRONTEND=noninteractive \
    BUN_INSTALL=/opt/bun \
    CARGO_HOME=/opt/cargo \
    RUSTUP_HOME=/opt/rustup \
    BAZELISK_HOME=/opt/bazelisk \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium \
    PI_TINY_DEVICE=cpu \
    CARGO_BUILD_JOBS=2 \
    RUSTFLAGS="-C debuginfo=2 -C force-frame-pointers=yes" \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
ENV PATH=/opt/bun/bin:/opt/cargo/bin:/opt/omp-python/bin:${PATH}
ENV PI_NATIVES_DIR=/opt/omp-natives

# Mirrors the repository Dockerfile/infra runner's native prerequisites;
# Chromium permits browser tests on local fixture pages without a later download.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git gh jq zip unzip xz-utils zstd \
      build-essential clang lld llvm libclang-dev cmake ninja-build pkg-config \
      libssl-dev libdbus-1-dev libx11-dev libpipewire-0.3-dev libspa-0.2-dev \
      libcairo2-dev libpango1.0-dev libjpeg-dev libgif-dev librsvg2-dev \
      python3 python3-dev python3-venv python3-pip \
      chromium fonts-liberation fd-find ripgrep imagemagick sqlite3 \
    && ln -s /usr/bin/fdfind /usr/local/bin/fd \
    && ln -s /usr/bin/convert /usr/local/bin/magick \
    && rm -rf /var/lib/apt/lists/*

# The baseline Bun binary also matches the Linux release compile target.
# Bun 1.4.2 and Bazelisk 1.29.0 are pinned by dogfood-release.yml.
RUN curl -fsSL "https://github.com/oven-sh/bun/releases/download/bun-v${BUN_VERSION}/bun-linux-x64-baseline.zip" -o /tmp/bun.zip \
    && unzip /tmp/bun.zip -d /tmp/bun \
    && mkdir -p /opt/bun/bin \
    && install -m755 /tmp/bun/bun-linux-x64-baseline/bun /opt/bun/bin/bun \
    && ln -s /opt/bun/bin/bun /opt/bun/bin/node \
    && rm -rf /tmp/bun /tmp/bun.zip \
    && curl -fsSL https://sh.rustup.rs -o /tmp/rustup-init.sh \
    && sh /tmp/rustup-init.sh -y --no-modify-path --default-toolchain none --profile minimal \
    && rm /tmp/rustup-init.sh \
    && curl -fsSL "https://github.com/bazelbuild/bazelisk/releases/download/v${BAZELISK_VERSION}/bazelisk-linux-amd64" -o /usr/local/bin/bazelisk \
    && chmod +x /usr/local/bin/bazelisk \
    && ln -s /usr/local/bin/bazelisk /usr/local/bin/bazel

# Scanner contract: retain the entire branch checkout, tests, SDKs, and fixtures.
COPY . /src
WORKDIR /src

# Keep real dependency caches in image layers: no disposable BuildKit cache mounts.
# rustup show consumes rust-toolchain.toml (including its components and targets).
RUN rustup show \
    && cargo install --locked cargo-nextest \
    && cargo fetch --locked \
    && bun install --frozen-lockfile

# Use the same shipping Linux native targets as dogfood-release.yml, serially.
# MODULE.bazel supplies pinned Rust/Zig/LLVM/toolchains and the complete crate graph.
# The repository's script installs and version-stamps the addons in native/.
RUN bun scripts/bazel-natives.ts linux-x64-baseline -- --jobs=2 \
    && bun scripts/bazel-natives.ts linux-x64-modern -- --jobs=2

# Fetch every lazy WASM grammar declared by the checked-out generated inventory.
# The production installer verifies pinned decompressed size and SHA-256.
RUN bun -e '\
    import { wasmGrammarFor } from "@oh-my-pi/pi-natives"; \
    import { installGrammar } from "./packages/coding-agent/src/utils/grammars.ts"; \
    const inventory = await Bun.file("./crates/pi-ast/src/language/wasm_grammars.rs").text(); \
    const languages = [...inventory.matchAll(/WasmGrammar::new\(\s*"([^"]+)"/g)].map(match => match[1]); \
    if (!languages.length) throw new Error("Empty WASM grammar inventory"); \
    for (const lang of languages) { \
      const info = wasmGrammarFor({ lang }); \
      if (!info || !(await installGrammar(info))) throw new Error(`Failed to install grammar ${lang}`); \
    } \
    '

# Build the remaining Bun web/extension products through their actual scripts.
# Generate the extension's embedded CLI assets before compiling the binary.
RUN bun --cwd=packages/browser-relay run build \
    && bun --cwd=packages/collab-web run build \
    && bun --cwd=packages/stats run build \
    && bun --cwd=python/robomp/web run build

# Compile the genuine fork updater/build identity. Derive the semver from this
# branch; dogfood.1 is a local scan build identifier, not a publication claim.
RUN source_version="$(bun -p 'require("./packages/coding-agent/package.json").version')" \
    && RELEASE_TARGETS=linux-x64 \
       DOGFOOD_REPOSITORY=pedropaulovc/oh-my-pi \
       DOGFOOD_VERSION="${source_version}-dogfood.1" \
       bun run ci:release:build-binaries


# Install Python SDK and robomp unit-test dependencies without starting services.
RUN python3 -m venv /opt/omp-python \
    && /opt/omp-python/bin/pip install --no-cache-dir \
         -e ./sdk/python/omp-rpc -e './python/robomp[dev]'

# Preinstall optional CPU-side JS/native runtime graphs used by compiled binaries.
# This uses the production install API and package/version keys, not fake modules;
# it does not load model weights, run inference, record audio, or contact providers.
RUN bun -e '\
    import path from "node:path"; \
    import { ensureRuntimeInstalled, getFastembedRuntimeDir, getTinyModelsCacheDir } from "@oh-my-pi/pi-utils"; \
    import { fastembedRuntimeInstallPlan } from "./packages/mnemopi/src/core/fastembed-runtime.ts"; \
    import { getTransformersVersionSpec } from "./packages/coding-agent/src/subprocess/worker-runtime.ts"; \
    import { KOKORO_PACKAGE, KOKORO_VERSION, ONNXRUNTIME_NODE_PACKAGE, ONNXRUNTIME_NODE_VERSION, getTtsRuntimeDir } from "./packages/coding-agent/src/tts/runtime.ts"; \
    const plan = fastembedRuntimeInstallPlan(); \
    await ensureRuntimeInstalled({ runtimeDir: path.join(getFastembedRuntimeDir(), plan.versionKey), install: plan.install, probePackage: "fastembed" }); \
    const version = getTransformersVersionSpec(); \
    const key = version.replace(/[^A-Za-z0-9._-]/g, "_"); \
    const root = path.dirname(getTinyModelsCacheDir()); \
    const install = { dependencies: { "@huggingface/transformers": version }, trustedDependencies: ["onnxruntime-node"] }; \
    for (const name of ["tiny-title-runtime", "stt-runtime"]) { \
      await ensureRuntimeInstalled({ runtimeDir: path.join(root, name, `transformers-${key}`), install, probePackage: "@huggingface/transformers" }); \
    } \
    const manifest = await Bun.file("./packages/coding-agent/package.json").json(); \
    const sherpa = manifest.optionalDependencies?.["sherpa-onnx-node"] ?? manifest.dependencies?.["sherpa-onnx-node"]; \
    if (!sherpa) throw new Error("Missing sherpa-onnx-node source dependency"); \
    await ensureRuntimeInstalled({ runtimeDir: path.join(root, "stt-runtime", `sherpa-${sherpa.replace(/[^A-Za-z0-9._-]/g, "_")}`), install: { dependencies: { "sherpa-onnx-node": sherpa } }, probePackage: "sherpa-onnx-node" }); \
    await ensureRuntimeInstalled({ runtimeDir: getTtsRuntimeDir(), install: { dependencies: { [KOKORO_PACKAGE]: KOKORO_VERSION }, overrides: { [ONNXRUNTIME_NODE_PACKAGE]: ONNXRUNTIME_NODE_VERSION }, trustedDependencies: [ONNXRUNTIME_NODE_PACKAGE] }, probePackage: KOKORO_PACKAGE }); \
    '

# Compile all first-party Rust test executables without running tests. These are
# the same vendored exclusions used by scripts/run-rs-task.ts, not a reduced crate.
RUN CARGO_NET_OFFLINE=true cargo nextest run --no-run --workspace \
      --exclude brush-core --exclude brush-parser --exclude cfg_aliases --exclude napi
ENV CARGO_NET_OFFLINE=true

# Credential-free smoke; leave all build/test drivers for the offline scanner.
RUN packages/coding-agent/binaries/omp-dogfood-linux-x64 --version \
    && packages/coding-agent/binaries/omp-dogfood-linux-x64 --smoke-test \
    && bun run ci:test:smoke
CMD ["/bin/bash"]
