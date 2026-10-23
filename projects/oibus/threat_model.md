# Threat model

## What this project does and where untrusted input enters
OIBus is an industrial data collection agent. It runs on a plant or edge machine, often on a network that reaches
both the shop floor and the internet, collects data from industrial equipment and databases through **south
connectors** (OPC UA, OPC classic via a gateway, Modbus, MQTT, S7, ADS, BACnet, SQL databases, ODBC/OLEDB, PI,
REST, FTP/SFTP, folder scanner, …), buffers it in a local cache, optionally transforms it, and forwards it through
**north connectors** (OIAnalytics, REST, MQTT, OPC UA, Modbus, Amazon S3, Azure Blob / Data Explorer, SFTP, file
writer). A separate **launcher** process spawns the backend and installs upgrades. All configuration lives in
SQLite databases in the data folder; connector secrets are encrypted there.

Untrusted input:
- **The web server** (Express, `backend/src/web-server/`, port 2223 by default): the REST API under `/api` and the
  Angular UI. Clients authenticate with HTTP Basic (argon2 password hashes) or an RS256 JWT issued by
  `/api/users/authentication` (`middlewares/auth.middleware.ts`). An IP filter (`middlewares/ip-filter.middleware.ts`)
  runs before authentication; by default only localhost is allowed.
- **Everything the south connectors receive**: values, node/tag names, timestamps and metadata from devices and
  servers; MQTT topics and payloads; SQL result sets; REST responses; file names and contents from scanned folders,
  FTP and SFTP servers. A device, broker or server on the plant network must be treated as hostile: it must not be
  able to execute code, write outside the data folder or configured paths, read secrets, or crash the process.
- **Responses from north targets** (REST endpoints, brokers, cloud services) and from the OIAnalytics API.
- **OIAnalytics commands** (`backend/src/service/oia/`): once registered, OIBus polls OIAnalytics for commands that
  update the configuration, import a configuration archive, run setpoints, retrieve logs, and update OIBus itself
  (`update-version` downloads and unzips an archive into `update/`, which the launcher installs). OIAnalytics is
  trusted, but anything that lets a party other than the registered OIAnalytics instance inject or alter commands
  is in scope, as is unsafe handling of the archives (zip slip, path traversal).
- **Files an authenticated user uploads**: configuration import archives, CSV item imports, certificates.

Trusted: the operator who installs OIBus and the data folder on disk; any authenticated web user. OIBus has a single
role: **every authenticated user is an administrator** and is expected to configure connectors that read from and
write to arbitrary hosts and paths on the machine.

## Components that matter most / least
Most important:
- Authentication and authorization: `auth.middleware.ts`, `ip-filter.middleware.ts` and `service/utils.ts`
  (`testIPOnFilter`), the user service, JWT handling, and any route reachable without credentials.
- Secret handling: `service/encryption.service.ts`, `crypto.db`, and every place a secret could leave in clear
  text (API responses, logs, configuration export, data sent to OIAnalytics).
- The custom transformer sandbox (`service/sandbox.service.ts`, isolated-vm): user code must stay in the isolate.
- File handling from untrusted sources: folder scanner, FTP/SFTP south connectors, north file writer and SFTP,
  the north cache (`service/cache/`), zip/unzip helpers, `sanitizeFilename`.
- Protocol parsing in south connectors (OPC UA, Modbus, MQTT, S7, ADS, BACnet, REST) and SQL query building with
  user-provided variables (`south/sql-dialects.ts`, the SQL south connectors).
- OIAnalytics registration and command processing; the launcher's upgrade path (`launcher/src/`).

Lower priority: the Angular frontend (`frontend/`), except XSS that runs in another user's session from data that
devices or OIAnalytics supply (item names, values, logs).

Out of scope: `documentation/`, `build/` (packaging scripts and the Windows installer), `docker/` and
`docker-compose.yml` (local test stack), test files and fixtures (`**/*.spec.ts`, `backend/src/tests/`,
`frontend/src/test/`), and vulnerabilities that live entirely in third-party dependencies unless OIBus code makes
them reachable.

## How to exercise it
- Everything is installed and built in `/src`: `backend/dist`, `frontend` output in `backend/dist/frontend`,
  `launcher/dist`.
- Start a local instance: `cd /src/backend && node dist/backend/src/index.js --config /src/data-folder`. The UI and
  the API are on `http://127.0.0.1:2223`; the first login is `admin` / `pass`. The OpenAPI description is
  `documentation/docs/api/openapi.json` (generated from the tsoa controllers in `backend/src/web-server/controllers/`).
- Tests: `cd /src/backend && npm test` (Node test runner, no external services needed), a single file with
  `node --experimental-config-file=test.config.json --test src/path/to/file.spec.ts`; `cd /src/launcher && npm test`;
  `cd /src/frontend && npm test` (Vitest in Chromium, preinstalled). Rebuild after a patch with `npm run build` in
  the affected folder.
- Devices, brokers, databases and OIAnalytics are not available offline: simulate them locally (a small MQTT, OPC
  UA, Modbus, HTTP or FTP server written in Node with the dependencies already in `node_modules`, a SQLite file, a
  folder for the folder scanner) or mock them in a unit test.

## How we rate severity
- **Critical**: remote code execution on the OIBus host, or reading/writing arbitrary files, by an unauthenticated
  web client or by a device, broker or server that a connector talks to; authentication bypass on the web API;
  executing commands as if they came from the registered OIAnalytics instance; a malicious update archive written
  outside `update/`.
- **High**: escaping the custom transformer sandbox to the host; secrets (connector passwords, private keys, the
  OIAnalytics token) disclosed in clear text through the API, logs, exports or OIAnalytics messages; IP filter
  bypass; path traversal from a remote file name (FTP, SFTP, folder scanner) or from the north cache; SQL injection
  from data a device or server returns; a single malformed message or file that crashes the whole process or stops
  every connector.
- **Medium**: stored XSS in the UI from data supplied by devices or OIAnalytics; information disclosure to an
  unauthenticated client (versions, internal paths, user enumeration); resource exhaustion that one peer can trigger
  with little traffic (unbounded memory or disk); a crash limited to one connector that does not recover.
- **Low**: issues that need an authenticated user (all users are administrators), a non-default and documented
  insecure configuration, or sustained traffic.

## Anything to leave alone
- Do not report that an authenticated user can configure a connector to read or write any host, path, database or
  query, run a custom transformer, or change users and settings: every user is an administrator by design.
- Do not report the default `admin` / `pass` credentials: the documentation requires changing them at installation.
- Do not report issues that exist only when the operator disables the IP filter (`--ignoreIpFilters`), adds a
  wildcard to it, or disables TLS certificate verification in a connector's settings.
- Do not report outdated dependencies without a reachable exploit path in OIBus.
- Patches: keep them minimal, in TypeScript, follow the repository's Prettier/ESLint style, and add a regression test
  next to the affected code (`*.spec.ts`, Node test runner for the backend and launcher, Vitest for the frontend).
