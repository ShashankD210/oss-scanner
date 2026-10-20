# Threat model: Basalt

## What this project does and where untrusted input enters

Basalt turns a Mac with Apple silicon into a local LLM server. It runs MLX inference on the Metal GPU and exposes an
OpenAI-compatible REST API, an Anthropic-compatible REST API, an MCP server, and optional routing to cloud models
through OpenRouter. We assume all input from network clients is untrusted: HTTP request lines, headers (Host, Origin,
Authorization), JSON bodies, and MCP messages. Model files downloaded from Hugging Face are untrusted as well: weight
files, tokenizer files, chat templates (rendered by a Jinja engine), and configuration JSON. Command-line arguments
and environment variables are set by the operator and are trusted.

## Components that matter most

- The REST server and its middleware chain: authentication, Host and Origin checks, rate limiting, and request
  decoding (Sources/BasaltServer/REST).
- The MCP server (Sources/BasaltServer/MCP).
- Request handling that reaches the inference engine, including prompt and chat-template rendering
  (Sources/BasaltInference, Sources/BasaltCore).
- Model download and loading from Hugging Face (Sources/BasaltCore/HuggingFaceHub.swift).
- The cloud client, which sends the OpenRouter API key as a Bearer token (Sources/BasaltCloud).

## Components that matter less

- The terminal UI (Sources/BasaltUI, Sources/BasaltCLI).
- BasaltApp, the SwiftUI client, which is macOS-only.
- benchmarks/, scripts/, Formula/, docs/.

## How to exercise it

- `swift build -c release` builds the server executable.
- `swift test` runs the suite in Tests/BasaltTests. REST tests run in-process with HummingbirdTesting.
- Running a real model requires Apple silicon and a model download, so the test suite is the cheap path.

## How you rate severity

- Unauthenticated access to the REST or MCP API when the server is bound to a non-loopback address, or any bypass
  of the API key check: high or critical.
- Remote code execution through a crafted HTTP request, MCP message, model file, or chat template: critical.
- A bypass of the Host or Origin checks on a loopback bind (for example DNS rebinding), or leaking an API key: high.
- A single request that crashes or hangs the server: medium.
- Anything that needs the operator to choose an unsafe setting that the documentation already warns about is low.

## Anything to leave alone

- Weaknesses in third-party dependencies pinned in Package.resolved (mlx-swift, mlx-swift-lm, Hummingbird,
  swift-transformers, and the others). Report those upstream.
- The quality, safety, or license of model weights downloaded from Hugging Face.
- The behavior of OpenRouter and other cloud providers.
- Visibility of --api-key on the command line through ps. The help text documents this and recommends
  BASALT_API_KEY instead.
