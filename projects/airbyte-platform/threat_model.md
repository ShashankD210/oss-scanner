# Threat model: airbytehq/airbyte-platform

This file adapts the Security Model in Airbyte's public OSS security policy for automated scanning. That model is the
canonical definition of what Airbyte treats as a vulnerability. Where this file and the policy disagree, the policy wins.

## What this project does and where untrusted input enters

Airbyte Open Source moves data between systems. Operators self-host it on Kubernetes (Helm chart V2 or `abctl`). This
repository holds:

- **Airbyte control plane:** the API server and public API, the web application, the workload API and workload
  launcher, the worker and cron services, and the Connector Builder endpoints served by `airbyte-server`. It
  authenticates requests, stores connection configuration, holds or brokers connector credentials, and decides what
  runs.
- **Airbyte data plane orchestration:** the replication orchestrator, connector sidecar and workload init container
  that start connector containers and carry records between them.
- **Deployment artifacts:** Helm chart V2 under `charts/v2/`.

The manifest server that evaluates Connector Builder manifests is a separate service built from `airbyte-python-cdk`.
Its code is not in this repository; only the platform's side of that integration is (see below).

Two properties shape everything below:

1. **Not designed for the public internet.** It runs inside a network the operator controls. Reaching that network is
   not authorization: authentication is a boundary the control plane enforces, and bypassing it is in scope wherever
   the attacker sits.
2. **No role separation.** Every authenticated user is an instance administrator. Do not report anything that depends
   on one authenticated user being less privileged than another, including access across workspaces or
   organizations.

Trust levels, lowest to highest:

1. **Unauthenticated network actor** who can reach the UI or API.
2. **Source systems and connector output.** Connectors are third-party code by design. Records, schemas, stream and
   field names, file names, error messages and AirbyteMessage frames are attacker-controlled input to everything
   downstream: the orchestrator, sidecar, mappers, the control plane, the database, and the web UI.
3. **Connector containers.** The source, destination, check, discover and spec containers are untrusted code running
   in the cluster.
4. **Authenticated user (instance administrator).**
5. **Airbyte control plane, Airbyte database, secret store, Kubernetes API.** This includes the platform containers
   that share a pod with connector containers: the workload init container, the connector sidecar and the
   replication orchestrator.

## Components that matter most / least

The scope test for every finding: does it let an actor or input cross from a lower-trust component into a higher-trust
one, or defeat a property the responsible component is supposed to enforce?

Highest priority:

- **Authentication on every non-public surface** (`airbyte-server`, `airbyte-commons-server`, `airbyte-api/*`,
  `airbyte-commons-auth`, `airbyte-commons-micronaut-security`, `airbyte-keycloak`, `airbyte-workload-api-server`):
  bypass, missing annotations on new endpoints, anonymous endpoints that do more than they should, session fixation or
  reuse, token validation flaws, OAuth flow handling in `airbyte-oauth` (state, redirect and callback handling
  triggerable without authenticating).
- **Secret handling** (`airbyte-config/config-secrets`, `airbyte-config/config-persistence`, API serialization,
  `airbyte-commons-storage` and job logs): once persisted, connector credentials are write-only. Any interface, API
  response, log line, job output, or error message that returns a stored secret is in scope.
- **What a connector container is given** (`airbyte-workload-launcher`, `airbyte-workload-init-container`,
  `airbyte-connector-sidecar`, `airbyte-container-orchestrator`): a connector container must not receive platform
  credentials, Airbyte database connection details, secret store access, a service account token, host access, or
  unnecessary privileges. The platform containers in the same pod are expected to hold platform credentials; that
  alone is not a finding. In scope is a connector container reaching what they hold: shared volumes, ports they
  listen on, a shared process namespace, or files they write. Internal endpoints a workload can reach must authorize
  the caller and confine it to its own job.
- **Connector output parsing** (orchestrator, sidecar, `airbyte-commons-protocol`, `airbyte-mappers`): source-supplied
  data causing code execution, deserialization, path traversal, protocol confusion (a connector steering the
  orchestrator outside the protocol), or uncontrolled resource consumption.
- **Injection** into the Airbyte database or any query the control plane builds from input it should parameterize
  (jOOQ and raw SQL in `airbyte-db`, `airbyte-data`, `airbyte-persistence`, `airbyte-config/config-persistence`).
- **Stored XSS in `airbyte-webapp`** where the rendered value came from a source system (stream names, field names,
  records shown in Connector Builder test reads, connector error messages) or from an unauthenticated actor.
- **Connector Builder path:** the Builder controllers in `airbyte-server` (`ConnectorBuilderProjectApiController`,
  `ConnectorBuilderAssistApiController`) and how the platform calls the manifest server
  (`airbyte-api/manifest-server-api`, authentication between them). Manifests, templates or interpolation causing
  execution outside the evaluation they were submitted for, on the platform side.
- **Helm chart V2 defaults** (`charts/v2/airbyte`): defaults that disable authentication, expose services more broadly
  than needed, put secrets in plain environment variables, run workloads as root or over-privileged, ship fixed
  credentials, or grant service accounts broader cluster permissions than the component needs. Assess the shipped
  defaults, not custom values.

Lower priority but in scope: `airbyte-workers`, `airbyte-domain`, `airbyte-cron`, `airbyte-bootloader`,
`airbyte-featureflag-server`, `airbyte-connector-rollout-worker`, `airbyte-audit-logging`, `airbyte-notification`,
`airbyte-statistics`, `airbyte-analytics`.

Out of scope as code: `airbyte-tests`, `airbyte-test-utils`, `**/src/test*`, `tools/`, build scripts, `.github/`.
Bundled third-party services in the chart (Temporal, Postgres, MinIO, Keycloak images) are operator-managed
infrastructure; only report how Airbyte configures or exposes them, including default credentials the chart sets for
them.

## How to exercise it

- Built image: Gradle cache is fully populated, so ./gradlew --offline :oss:<module>:test works. jOOQ code generation
  runs against a local Postgres instead of  Testcontainers (start it with pg_ctlcluster 15 main start); the image
  patches airbyte-db/db-lib/.../FlywayMigrationDatabase.kt for this, and git diff in /src shows that and two small
  build tweaks. Tests that start their own Testcontainers database still need Docker, which is unavailable, so reason
  from source for those.
- Rendered default chart: `/src/build/oss-scanner/chart-v2-defaults.yaml` (re-render with
  `helm template airbyte charts/v2/airbyte`).
- Webapp: `airbyte-webapp/` with `node_modules` installed; `pnpm test:ci` runs Jest offline.
- Public API spec: `airbyte-api/*/src/main/openapi/*.yaml`. Controllers: `airbyte-server/src/main/kotlin/`. Request
  handlers and authorization: `airbyte-commons-server/src/main/kotlin/`.
- What each workload container receives: env var factories in `airbyte-workload-launcher/src/main/kotlin/config/` and
  pod factories in `airbyte-workload-launcher/src/main/kotlin/pods/factories/`. Note that defaults change with the
  configured secret persistence type.

## How you rate severity

Rate by which boundary is crossed and what the attacker starts with, not by CVSS alone.

- **Critical:** starting unauthenticated, from source data, or from a connector container: access to an authenticated
  control plane surface or code execution in a control plane component; a connector container obtaining platform
  credentials, Airbyte database access, or secret store access through something the platform placed in or near the
  workload; unauthenticated retrieval of stored secrets.
- **High:** source-supplied data (records, schemas, names, AirbyteMessage frames) causing code execution or
  deserialization in the orchestrator, sidecar or control plane; a connector container reaching another job's data or
  the control plane through an internal endpoint that fails to authorize it; a manifest or template escaping its
  evaluation context on the platform side; stored secrets readable through any interface, including job logs shown in
  the UI, even by an authenticated user; SQL injection in a control plane query.
- **Medium:** stored XSS where the payload originates from a source system or unauthenticated actor; insecure Helm
  chart V2 defaults as listed above with no documented hardening step; path traversal from source-supplied file or
  stream names limited to the workload's own storage; secrets written to logs that only other workloads can read.
- **Low:** source-supplied input causing resource exhaustion in a control plane component that crosses a boundary
  (for example, one connector degrading other jobs or the server).
- **Do not report:** anything below.

## Anything to leave alone

Do not report these. They are out of scope under Airbyte's policy:

- **Anything an authenticated user can do through the documented UI, API or configuration.** This includes registering
  connectors that reach arbitrary endpoints, supplying arbitrary credentials, authoring connectors in the Connector
  Builder, running custom connector images, and accessing any workspace or organization on the instance. Every
  authenticated user is an administrator.
- **SSRF by configuration:** a connector, webhook or notification reaching an arbitrary URL, including internal or
  link-local addresses (for example 169.254.169.254), when an authenticated user configured it. In scope only if it can
  be triggered without authenticating.
- **Self-inflicted findings:** SQL, command, header or request injection into a query or request a connector builds
  from configuration the same user supplied; self-XSS. The exclusion depends on who supplied the value. If it came from
  a source system or an unauthenticated actor, it is in scope.
- **Platform containers holding platform credentials.** The init container, sidecar and orchestrator need credentials
  to talk to the control plane and, depending on secret persistence, the database. Report only a path for a connector
  container to obtain them.
- **Operator deployment decisions.** Assume: UI and API reachable only from a trusted network, VPN or authenticated
  proxy; authentication enabled with install-generated credentials not weakened; TLS in front of the platform; a secret
  manager configured; database and object storage private; connector workloads isolated and egress controlled;
  least-privilege service accounts; connector credentials minimally scoped. A finding that only works when one of
  these is not met is out of scope.
- **Exceptions to the previous point, which ARE in scope:** the insecure state is the shipped default and no hardening
  requirement is documented; Airbyte documentation is wrong, outdated, or contradicts what the code does; or the
  condition cannot be met in a supported deployment mode (for example, the chart does not expose the needed setting).
  Comparing code and chart defaults against the documentation is a good use of your time.
- **Post-compromise scenarios:** tampering with the Airbyte database, object storage, Temporal, the secret manager or
  the Kubernetes cluster, unless an Airbyte vulnerability causes the tampering.
- **Network escapes that depend on the operator's cluster permitting the connection.** Report only when the platform
  hands the workload the credential, endpoint or privilege used.
- **Third-party code:** vulnerable dependency versions, unless exploitable through Airbyte code (then report the
  Airbyte code path). Bugs in the manifest server itself belong to `airbyte-python-cdk`.
- **Missing security headers, cookie flags, version disclosure, user or object enumeration, rate limiting, and resource
  exhaustion that crosses no boundary.** Clickjacking on unauthenticated pages.
- **How a destination interprets data Airbyte wrote,** or how a source behaves when read.

## How we would like reports

- One report per root cause. Aggregate endpoints or payloads that share a flaw.
- State the starting position (unauthenticated, source system, connector container) and which boundary is crossed.
- Name the module and file, and confirm the issue against the current `main` branch.
- Include a minimal reproduction and a proposed patch where possible. Prefer fixes that follow existing patterns
  (Micronaut security annotations, jOOQ parameterization, existing secret persistence abstractions).