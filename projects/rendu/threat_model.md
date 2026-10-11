# Threat model

## What this project does and where untrusted input enters
Rendu is a desktop Markdown viewer (Tauri 2 + Rust backend, React/TypeScript frontend). It is a local, single-user app with no server component.
Untrusted input: the contents of Markdown files the user opens or drops onto the window (often AI-generated or downloaded documents), including raw HTML, embedded images/links, Mermaid diagrams, KaTeX math and front matter. Also file paths and drag-and-drop events passed to the Rust backend.

## Components that matter most / least
- Most: the Markdown -> HTML rendering pipeline (react-markdown, rehype-raw, rehype-sanitize, Mermaid, KaTeX) and the Tauri IPC commands/capabilities in src-tauri (file reads, opening links/external URLs, path handling).
- Less: UI styling, i18n, PDF export (html2pdf.js).
- Out of scope: third-party dependency vulnerabilities with no reachable path from Rendu (dependabot/cargo audit cover these), e2e test tooling.

## How to exercise it
- Frontend tests: `npm run test:run` (vitest). Backend: `cd src-tauri && cargo test --lib`. The GUI cannot be launched in the offline image; reason about the rendering pipeline and Rust commands directly.

## How you rate severity
- Critical/high: a crafted Markdown file leading to code execution, arbitrary file read/write outside what the user chose, or access to Tauri IPC from rendered content.
- Medium: XSS/script execution in the webview that cannot reach IPC or local files; path traversal limited in impact.
- Low: DoS or hangs from pathological documents.

## Anything to leave alone
- Do not report that opening a local file the user chose reads that file; that is the intended function.
