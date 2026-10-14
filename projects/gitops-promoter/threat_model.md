# Threat model

## What this project does and where untrusted input enters

GitOps Promoter is a Kubernetes controller. It promotes configuration that already lives in Git from one environment branch to the next by opening and merging pull requests, and it gates each promotion on commit statuses. It shells out to `git` against bare clones (no work tree) and talks to GitHub, GitLab, Forgejo, Gitea, Bitbucket Cloud, and Azure DevOps.

The process that matters is `cmd/main.go`. It runs the controllers, the admission webhooks, an aggregated dashboard API, and an SCM webhook receiver (default listen port 3333).

Untrusted input:

- Git objects the controller fetches: commit messages, trailers, notes, trees, file contents, and ref names. These are passed into `git` plumbing in `internal/git`. A branch name, path, or message that becomes a `git` argument or a shell metacharacter is in scope. The clone is bare and must stay that way; nothing from the repository should be executed.
- SCM HTTP responses (pull requests, commit statuses, labels) parsed under `internal/scms/`.
- HTTP bodies posted to the webhook receiver (`internal/webhookreceiver`). The receiver reads the body, pulls a commit SHA and ref out of it, and enqueues a reconcile of a matching `ChangeTransferPolicy`. It does not verify webhook signatures. That gap is known and tracked ([issue 1285](https://github.com/argoproj-labs/gitops-promoter/issues/1285)). A report that only restates the missing check, with no impact beyond triggering a reconcile, is a known gap. An unsigned webhook that pushes a commit, changes credentials, runs a command, or escapes the reconcile enqueue is in scope.
- Custom resources, but only where a field is interpolated into a command, a credential, or an expression that untrusted Git or webhook data can influence. People who can create promoter CRs are operators of this controller. Cluster-admin RBAC is not an attacker.

Trusted input: kube API config, Secrets referenced by `GitRepository` and SCM provider resources, controller flags, and the operator-authored `expr` expressions on commit-status and label checks (`github.com/expr-lang/expr`). Those expressions are in scope only when untrusted Git or webhook data is concatenated into the expression source, or when evaluation escapes the documented environment.

## Components that matter most / least

Most:

- `internal/git`: argument construction for `git`, `GIT_ASKPASS` (`hack/git/promoter_askpass.sh`), bare-clone invariants, and anything that writes outside the clone directory.
- SCM credential handling and git push/merge paths. Leaking a token into logs, a remote URL, an error returned to the API, or the dashboard is serious. Writing a commit to an environment branch without passing the configured gates is serious.
- `internal/webhookreceiver`: body parsing, provider detection, and the miss-retry goroutines (capped at 256). There is no payload size limit yet (see the TODO in `server.go`).
- `internal/scms/<provider>`: request and response handling for each provider.
- The aggregated dashboard API (`internal/apiserver`) and the UI that renders promotion state (`ui/`). Cross-namespace reads, or rendering SCM data as HTML without encoding, are in scope.

Less important, still in scope: metrics and logs, as long as the bug is a secret or cross-tenant leak. CRD CEL validation cost is a denial-of-service concern only when a namespace tenant who cannot otherwise crash the apiserver can do so through a promoter CR.

Out of scope:

- `test/e2e`, `hack/` (except `promoter_askpass.sh` and the fact that `hack/celcost` is a separate module), `docs/`, `.github/`, and the install YAML under `config/` as a supply-chain of the scanner host. Generated deepcopy and OpenAPI files are not hand-written logic.
- Whether a promotion picked the branch an operator expected, when no gate was skipped and no credential or command boundary was crossed.
- Vulnerabilities that require an attacker who already has cluster-admin, or who already has the SCM token the controller uses.
- The SCM products themselves.

## How you rate severity

- **critical**: code execution in the controller or webhook process from Git content, an SCM response, or a webhook body. Theft of an SCM credential. A commit landing on a protected environment branch that the configured gates should have blocked, when the attacker could not already push there.
- **high**: writing files outside the clone directory. A `git` invocation that runs an attacker-controlled program. Reading another namespace's promoter objects or Secrets through the dashboard API. An `expr` expression, fed by Git or webhook data, that escapes its environment.
- **medium**: denial of service that survives the request (unbounded webhook body, clone disk fill, a goroutine leak past the miss-retry cap). A gate that fails open. Secrets that appear only in local debug logs.
- **low**: a crash or hang on one malformed input that ends with that reconcile and leaves nothing behind.

Reports should name the input (a Git object, a webhook body, or a CR) and the commit they were tested against.

## How to exercise it

The manager is at `/usr/local/bin/gitops-promoter`. It needs a Kubernetes cluster and Secrets to do real work, so prefer the tests.

`KUBEBUILDER_ASSETS` points at the Kubernetes 1.37 envtest binaries. `GOPROXY=off` and `GOTOOLCHAIN=local` are set; do not clear them.

Unit tests that do not need a cluster, and that this image already runs at build time:

```
go test -count=1 -timeout 25m \
  ./internal/git/... \
  ./internal/webhookreceiver/... \
  ./internal/utils/... \
  ./internal/scms/... \
  ./internal/labels/... \
  ./internal/webrequest/...
```

Controller and apiserver envtest (slow; needs `KUBEBUILDER_ASSETS`, which is already set):

```
go test -count=1 -timeout 45m ./internal/controller/ ./internal/apiserver/
```

Fuzz targets live in `internal/utils` (`make fuzz-replay` replays the corpus; `go test ./internal/utils -run '^Fuzz' -fuzz=Fuzz -fuzztime=30s` explores). `git` is on `PATH`. Clones created by `internal/git` are bare repositories under the process temp dir.

`test/e2e` needs a Kind cluster and is not runnable here. The dashboard UI under `ui/` is TypeScript; this image has no Node toolchain, so review that tree from source.
