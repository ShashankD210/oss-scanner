# Threat model: fossid_compliance_updater

## What this project does and where untrusted input enters
- A CLI (`java -jar target/fossidComplianceUpdater.jar`) run by a compliance engineer. It fetches dependency and
  identified components from a FossID Workbench scan, fills in missing SPDX license, URL and copyright fields, and
  writes them back through the FossID API. It holds a FossID API key, and usually a Gemini token or GitHub PAT.
- Untrusted input, in rough order of exposure:
  1. Component metadata from the FossID scan (package manager, name, version, URL). A scanned repository controls
     these through its manifests, so an attacker who can get a package into a scan controls them.
  2. Responses from public registries (npm, Maven Central, PyPI, crates.io, NuGet, pkg.go.dev, Hackage, etc.) and
     from the GitHub REST API, including base64 LICENSE contents and parent POMs (up to 5 levels).
  3. AI model output (Gemini API, Ollama, or the `gemini` CLI). Model answers may be prompt-injected via (1) and (2);
     the tool treats returned URLs and license IDs as data to validate and then write to FossID.
  4. CLI arguments and environment variables (FOSSID_API_KEY, GEMINI_TOKEN, OLLAMA_TOKEN, GITHUB_PAT).

## Components that matter most / least
- Most: `RegistryInfoFetcher` and `UrlLicenseFetcher` (URLs built from component names; SSRF and request-smuggling
  surface), `GeminiCliClient` (builds a PowerShell command line from a prompt that embeds component data; the
  here-string sanitizer is the only barrier), `FossIdClient` (sends username + API key in every request body; what
  gets written back), `Main` (duplicates stdout/stderr into a log file in the working directory; credentials in logs).
- Less: `PromptBuilder`, result/record classes, console formatting.
- Out of scope: the FossID server, Google/Ollama services, the `gemini` CLI binary itself (not present in the image).

## How to exercise it
- `mvn -o clean package` rebuilds offline; `java -jar target/fossidComplianceUpdater.jar --help` runs.
- There is no test suite. Drive the clients with a small local HTTP stub (e.g. `python3 -m http.server` or a few
  lines of Java/Python) that imitates `api.php`, a registry, or the Gemini endpoint. Ollama mode takes an arbitrary
  `--ollamaaddress`, which is the easiest way to point the tool at a stub.
- The Gemini CLI path (`--aitool gemini_cli`) spawns `powershell.exe`, so it cannot run on Linux; review it statically.

## How you rate severity
- Command injection through the PowerShell here-string in `GeminiCliClient` from component names or AI output: critical.
- SSRF: a component name or AI-suggested URL that makes the tool fetch an arbitrary host, especially with the GitHub
  PAT or other Authorization header attached: high.
- Credential leakage of the FossID key, Gemini/Ollama token or GitHub PAT into the log file, console, exception
  messages or URLs: high.
- Writing attacker-chosen license/URL/copyright values to FossID without validation (compliance data poisoning via
  prompt injection or a malicious registry response): medium, high if it can be done without the AI step.
- TLS verification disabled or hostname checks skipped: high.
- Unbounded memory or CPU from a hostile registry/AI response, or a runaway parent-POM chain: medium.
- Crashes that need a malicious server and only stop the run: low.

## Anything to leave alone
- Passing secrets on the command line is documented and env-var alternatives exist; do not report it as a finding.
- Plain-http `--address` or `--ollamaaddress` chosen by the operator is their decision, not a finding.
- The log file is written to the current directory by design; report only if its contents leak secrets.
