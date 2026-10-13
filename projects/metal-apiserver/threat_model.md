# metal-apiserver threat model

metal-apiserver is the gRPC/Connect (ConnectRPC) API server of the metal-stack
bare-metal cloud, a implementation of the metal-stack V2 API
(`github.com/metal-stack/api`). It manages tenants, projects, machines, IPs,
networks, images, sizes and partitions; issues and revokes JWTs; drives PXE boot
and BMC power control for physical servers; brokers Headscale VPN enrollment; and
records audit events. It fronts backing services (RethinkDB, Redis/Valkey, an IPAM
gRPC service, Headscale, the tenant-apiserver, and optional TimescaleDB/Splunk
audit backends).

Authorization is centralised in ConnectRPC interceptors, so one mistake can turn a
low-privileged token into cross-tenant reads and writes, physical-machine control
(power, reinstall, console/BMC passwords, decommission), network/IP hijacking, VPN
enrollment into another project's tailnet, or the ability to mint further tokens.
Tampering with the audit trail is equally serious.

## Scope and trust boundaries

Audit the runtime code: `cmd/server` (the HTTP server — ConnectRPC over HTTP/1.1,
HTTP/2 and cleartext h2c, plus the OIDC handlers under `/auth/`), the
authentication and authorization interceptors (`pkg/auth`, `pkg/request`), the
validation, rate-limit and tenant interceptors that wrap handlers, the service
implementations (`pkg/service/api`, `pkg/service/admin`, `pkg/service/infra`,
`pkg/service/auth`), the repository layer (`pkg/repository`), the datastore and
its query builders (`pkg/db`), token issuance and validation (`pkg/token`) and the
signing-certificate store (`pkg/certs`). Dependencies may be examined where
necessary to prove a vulnerability reachable through metal-apiserver. The
procedure, role and visibility tables generated from `github.com/metal-stack/api`
are what the authorizer trusts; a mismatch between those tables and the handlers is
in scope.

Consider these attackers:

- Unauthenticated remote attackers. Only public procedures are reachable without a
  token (`HealthService/Get`, `MethodService/List`, `VersionService/Get` and gRPC
  reflection), plus the `/auth/` OIDC endpoints; authentication runs as a request
  gate before any handler, so every other route must reject a missing or invalid
  token.
- Authenticated, low-privileged users: holders of a user token (roles resolved
  live from the tenant-apiserver) or an API token carrying tenant, project,
  machine, infra or admin roles, wildcard `*` subjects, or fine-grained
  `MethodPermission` lists.
- Compromised or malicious enrolled machines and infrastructure components holding
  a machine token or an infra token, which call `BootService`, `BMCService`,
  `EventService`, `SwitchService` and `ComponentService`.
- Provider-tenant members, who may hold `ADMIN_ROLE_EDITOR` / `ADMIN_ROLE_VIEWER`.
- Operators, their configuration and secrets, the intended OIDC provider, and the
  configured tenant-apiserver are trusted as design inputs. Still examine how the
  tenant-apiserver's answers map to roles, and how OIDC identity is extracted. Do
  not report access intentionally granted by a configured role as a bypass.

Treat every RPC request field as untrusted, including resource identifiers and
nested filters/queries, the header-supplied client IP used for rate limiting, the
OIDC `redirect-url`/`state`, and the host/BMC/boot data reported by machines.

## Security properties and areas of interest

- Authentication and revocation. JWTs must be ES512, carry a valid issuer,
  audience, `exp` and `nbf`, and be checked against the token store on every
  request so that revocation is immediate. The signing private key is held in
  Redis; theft of or tampering with the certificate store is equivalent to the
  ability to mint an administrator token.
- Authorization. Every non-public method must be classified into exactly one
  visibility (public/self/admin/infra/machine/tenant/project), and the subject must
  be derived from the request and be one the token was actually granted. Trace the
  wildcard `*` reduction (a wildcard must expand only to the caller's real
  memberships), the admin-role precedence path, and the service-side checks for
  self-scoped methods.
- Privilege elevation. `TokenService/Create`, `/Update` and `/Refresh` must not
  grant roles or permissions the caller does not already hold, and admin roles only
  to the extent of the caller's provider-tenant membership. Verify the requested
  admin/infra/machine/tenant/project roles and `MethodPermission` subjects against
  the caller's live permissions, on both the calling token and the token's owner.
- Machine and infrastructure control. `MachineService/BMCCommand` and `/GetBMC`,
  admin `MachineService/ConsolePassword`, `BootService/SuperUserPassword`,
  `BootService/MachineToken`, `BootService/Register`/`Dhcp`,
  `BMCService/UpdateBMCInfo`, and the switch/component methods are powerful and
  must stay scoped to the caller's machine/project/infra role.
- Cross-tenant/cross-project isolation. Repository scoping and the tenant
  interceptor must prevent reading or mutating another tenant's or project's
  resources; a guessed UUID, network or IP must not bypass the subject check.
- OIDC. The callback must validate the provider, the state/nonce and the redirect
  target against the allowlist (scheme and hostname) so a token cannot be
  delivered to an attacker origin; the `uniqueUserKey`/subject mapping must be
  trustworthy.
- VPN/Headscale. Auth keys and node operations must be project-scoped and must not
  enroll a node into, or enumerate, another project's tailnet.
- Input handling. protovalidate validates request messages, but trace
  user-controlled values into RethinkDB query construction, IPAM prefix
  allocation, task payloads and log/audit output: they must not expose secrets,
  cross an authorization boundary, or cause a practical remote denial of service.

## Offline build and reproduction

The checkout is at `/src`; the server is built from `./cmd/server` (for example
`go build -o /usr/local/bin/server ./cmd/server`). The image retains Go, the
source, the module cache and build caches, with `GOTOOLCHAIN=local`,
`GOPROXY=off` and `GOSUMDB=off` so that the audit performs no downloads.

The baseline is `go test ./...`; a scoped `go test -race` is also supported by the
retained toolchain. Many tests build a real repository or server through
`pkg/test` and `pkg/e2e`, which start RethinkDB, Valkey, PostgreSQL and Headscale
via testcontainers and therefore need a Docker daemon this image does not provide.
Expect those packages to fail or skip; a successful image build is not evidence
that they passed. Prefer focused tests that need no containers (`pkg/auth`,
`pkg/request`, `pkg/token`, `pkg/rate-limiter`, `pkg/certs`, plus the in-package
tests under `pkg/service` and `pkg/repository`), and reproduce authorization and
token bugs with the in-memory harness
(`test.StartRepositoryWithCleanup(..., test.WithContainers(false))`, which uses
miniredis, an in-memory tenant-apiserver and IPAM) and locally controlled HTTP
fixtures. Use temporary stores and never a shared database.

## Severity and reports

Base severity on demonstrated impact and realistic attacker prerequisites.

- Critical: forging or stealing tokens (including any path to the Redis signing
  key), privilege escalation through token creation/update/refresh, or a
  cross-tenant/cross-project authorization bypass reaching another tenant's
  machines, networks, IPs or data.
- High: unauthorized machine control (power, reinstall, console/BMC password,
  decommission), route/subnet or VPN enrollment across boundaries, disclosure of a
  secret (BMC superuser password, Headscale key, database/audit credentials), or
  remote code execution.
- Medium: a reproducible remote denial of service, or an information leak limited
  to one tenant or to metadata the caller could already infer.
- Low: defects requiring an administrator token, host access, or a non-default
  insecure configuration, and deviations with no security consequence.

Each report should identify the scanned commit, the attacker capabilities (token
type and roles required), the affected configuration, the entry point, and the
complete path to impact. Include a deterministic reproducer, expected versus actual
behaviour, and a focused patch with a regression test where possible. Identify
dependency-caused findings separately and explain their reachability through
metal-apiserver. Deduplicate reports that share one root cause.

## Anything to leave alone

- The tenant-apiserver, OIDC provider, RethinkDB, Redis/Valkey, IPAM, Headscale,
  TimescaleDB and Splunk are separate systems with their own security processes; a
  flaw wholly inside one of them is out of scope unless metal-apiserver's use makes
  it reachable, in which case show the path.
- The `metalctl` CLI, the web console, metal-hammer, metal-networker and other
  metal-stack components live in other repositories; only the server side of their
  interaction with this apiserver is in scope.
- The Prometheus `/metrics` endpoint on the separate metrics port, and the public
  health/version/reflection procedures, are intended to be reachable; report them
  only with a demonstrated security impact.
- The server CLI (`serve`, `token`, `datastore`, `vpn`), including
  `api-server token create` and its documented permission-check bypass, is an
  operator bootstrap facility; it is a finding only if a non-operator can reach it.
- The permissive CORS configuration, the dev-stage fake OIDC provider, and any
  behaviour that requires `STAGE=DEV` or a similarly non-production configuration.
