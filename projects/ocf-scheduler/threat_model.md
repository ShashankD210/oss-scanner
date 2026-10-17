# Threat model

## What this project does and where untrusted input enters

OCF Scheduler is a multi-tenant Go service that runs on a Cloud Foundry platform. Users schedule CF tasks (jobs)
and outbound HTTP calls against their apps with cron expressions and time zones. The service stores everything in
PostgreSQL, fires schedules from an in-process cron engine, and runs work from a worker pool.

Untrusted input enters at:

- **The REST API** (`http/`, routes in `http/routes/`). Any CF user can call it with a UAA bearer token. Callers
  are untrusted: a token proves who they are, and the scheduler must then check, through the CF API, that they hold
  a role in the app's space (`cf/auth_service.go`: `Verify`, `VerifySpace`, `IsAdmin`). Request bodies, path IDs,
  cron expressions, time zones, and HTTP call definitions (URL, method, headers, body) are all attacker-controlled.
- **The Open Service Broker API** (`broker/`), when it is enabled. Cloud Controller calls it with HTTP basic auth.
  Provision, bind, and resume requests carry instance, binding, space and app GUIDs and parameters.
- **Responses from scheduled HTTP calls**, which come back from user-chosen endpoints.

Trusted: the operator's environment configuration, PostgreSQL, UAA, and the CF API.

## Components that matter most / least

- Most: authentication and authorisation (`cf/auth_service.go`, `http/routes/access.go`,
  `http/routes/schedule_owner.go`, the binding checks), and tenant isolation. A user in one space must never read,
  change, run or delete another space's jobs, calls, schedules or executions.
- Next: the broker (`broker/`), the SQL layer (`postgres/`), the cron and time zone parsing (`cron/`), and the
  call runner (`combined/`, `core/`). The call runner makes outbound requests to URLs that users supply.
- Out of scope: the mock servers and dev tools (`cmd/jeuaa`, `cmd/jecfapi`, `cmd/cli`), `mock/`, `ci/`,
  `scripts/`, and the `Dockerfile.*` / `docker-compose.yml` dev environment.

## How to exercise it

- `start-test-db` starts the local PostgreSQL. `DATABASE_URL` already points at its `test` database.
- `./scripts/blanket` (what `make test` runs) runs the unit and integration tests. `go test ./postgres/...` needs
  the database; most other packages use the mocks in `mock/`.
- `/src/bin` holds the built commands. To run the service offline against the mock UAA and CF API:
  ```
  start-test-db
  UAA_ENDPOINT=http://localhost:8001 CF_ENDPOINT=http://localhost:8002 bin/jeuaa &
  UAA_ENDPOINT=http://localhost:8001 CF_ENDPOINT=http://localhost:8002 bin/jecfapi &
  CLIENT_ID=scheduler CLIENT_SECRET=lolsekrat UAA_ENDPOINT=http://localhost:8001 \
    CF_ENDPOINT=http://localhost:8002 SCHEDULER_PORT=8000 bin/scheduler
  ```
  The broker is configured from the environment by `broker.ConfigFromEnv` (`broker/config.go`).

## How you rate severity

- Critical: authentication bypass, or any cross-tenant access, such as reading, changing, running or deleting
  another space's resources, or running a task in another app.
- High: SQL injection; reaching the scheduler's own credentials or the database; making a scheduled call carry
  the scheduler's credentials; a broker request that affects an instance it did not create.
- Medium: denial of service that one authenticated user can trigger (for example a cron expression or time zone
  that hangs or crashes the engine or the worker pool); information leaks in error messages or logs.
- Low: anything that needs operator or platform admin credentials, or the trusted services to misbehave.

## Anything to leave alone

- Outbound HTTP calls to the URLs users configure are the product's purpose. Report them only when they cross a
  boundary that the code means to enforce.
- The `cmd/jeuaa` and `cmd/jecfapi` mocks are insecure on purpose.
