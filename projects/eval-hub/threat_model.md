# Threat model: EvalHub

## What this project does and where untrusted input enters

EvalHub is a Go REST API for orchestrating LLM evaluations. It stores evaluation
jobs, providers, collections, and post-processing workloads in SQLite or
PostgreSQL; launches local processes or Kubernetes Jobs; and integrates with
MLflow. This repository also contains an MCP server, an evaluation runtime
sidecar, and a runtime init binary that fetches evaluation data.

Treat REST paths, query parameters, headers, JSON bodies, job status callbacks,
MCP requests, model endpoints, data references, and upstream HTTP responses as
untrusted. Evaluation data may come from Git, S3, Hugging Face, or OCI registries.
Provider adapters and their output should not gain access to another job's or
tenant's data or credentials.

## Trust boundaries and components that matter most

- In cluster mode, kube-rbac-proxy authenticates and authorizes requests before
  supplying `X-User` and `X-Tenant`. EvalHub trusts those headers and must enforce
  tenant and owner scoping consistently in handlers, storage, and runtimes.
  Check reads, lists, updates, deletes, job events, logs, and post-processing
  references for cross-tenant or cross-owner access. Demonstrate a reachable
  bypass in the intended deployment when reporting identity spoofing.
- Local mode (`--local`) intentionally omits the cluster identity requirement
  and is intended for trusted local use. An unauthenticated local API alone is
  not evidence of an authentication bypass in a cluster deployment.
- Prioritize `internal/eval_hub/handlers`, `server`, `storage`, `runtimes`, and
  `postprocessing`; `internal/evalhub_mcp`; `internal/eval_runtime_sidecar`; and
  `cmd/eval_runtime_init`. Shared validation and outbound clients under `pkg/`
  are also in scope.
- Check SQL query construction, resource identifiers, workload type separation,
  and the mapping from API requests to local commands and Kubernetes pod specs.
  Provider executable and image configuration supplied by an administrator is
  trusted; execution of that configured workload is expected. Look for ways an
  untrusted request can cross that boundary or elevate its permissions.
- Review sidecar API, model, MLflow, and OCI proxies for target validation,
  credential leakage, unsafe redirects, incorrect TLS verification, and access
  outside the job's intended scope. Check init downloads and extraction for
  SSRF, path traversal, symlink escapes, and unintended file overwrites.
- Review MCP HTTP handling, tool arguments, resource URIs, and forwarding of
  API credentials and tenant context. Stdio transport normally inherits the
  invoking user's trust; assess HTTP exposure separately.

The external Python SDK, evaluation frameworks, Kubernetes, kube-rbac-proxy,
MLflow, and databases are separate projects. Their integration with EvalHub is
in scope; defects solely in their implementation should be identified as such.
Configuration and deployment examples remain useful for establishing whether
an attack is reachable under a supported setup.

## How to exercise it

The checkout is in `/src`. `make build` produces `bin/eval-hub`,
`bin/eval-runtime-init`, `bin/eval-runtime-sidecar`, and `bin/evalhub-mcp`.
The Go module cache and Python SDK with adapter dependencies are installed in
the image; `/opt/evalhub-venv/bin` is on `PATH`.

Run the baseline unit tests and fuzz seed corpora with:

```sh
go test -race -short -timeout 10m \
  -skip '^(TestMLFlowIntegration|TestGetEvaluationJobs_Postgres)$' \
  ./internal/... ./cmd/... ./pkg/...
```

Use existing package tests and local `httptest` servers to reproduce attacks
without Internet access. All Go test packages are compiled during the build.
Targeted mutational fuzzing can be run with `go test -fuzz` in packages containing
`Fuzz*` functions.

For manual REST testing, run `./bin/eval-hub --local` from `/src`. The default
configuration uses an in-memory SQLite database and listens on port 8080;
MLflow and OpenTelemetry are optional. The API specification is in
`docs/src/openapi.yaml`. Run the MCP server with
`./bin/evalhub-mcp --transport http --host 127.0.0.1 --port 3001`.

The broader suites in `tests/features/` and `tests/mcp/` need a running service.
Some scenarios additionally need Kubernetes, PostgreSQL, MLflow, model servers,
external datasets, or provider images. Those services, datasets, and images are
not provisioned by this scanner Dockerfile. The baseline excludes the MLflow
and PostgreSQL integration tests, which provision services even with `-short`.
Use local mocks or narrow package tests for offline reproducers; see
`tests/features/README.md`, the Makefile, and godog tags.

## Severity and reporting guidance

Assess impact using the attacker's actual privileges and required deployment
configuration. Remote code execution outside the intended evaluation workload,
cross-tenant access, privilege escalation, and credential theft are high priority;
critical severity should require a demonstrated severe impact and practical
attack path. Assess denial of service by its reach and resource cost, especially
whether one tenant can disrupt others. Do not treat ordinary execution of an
administrator-configured provider as arbitrary code execution.

Reports should include a minimal offline reproducer, affected component,
attacker permissions, configuration prerequisites, expected versus observed
behavior, and a regression test with any proposed patch. Keep reports private
and send them to the contact in `project.yaml`.
