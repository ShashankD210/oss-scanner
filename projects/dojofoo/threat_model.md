# Threat model: dojofoo

## What this project does and where untrusted input enters

dojofoo is a coding dojo that runs on the learner's own machine. The `dojofoo` CLI (`packages/cli`) installs courses
of small exercises ("katas") from GitHub, npm, a tarball URL or the registry, and starts a local web app
(`packages/ui`, a Nitro/Vite server) on `localhost`. In that app an AI coding agent ("Sensei") teaches a lesson,
and an authoring workspace lets a course author write a course with another agent ("Kyoshi", `packages/authoring`,
running on the bundled agent runtime in `packages/agent`). The agents are third-party coding CLIs or libraries
(OpenCode, Codex, Pi, Cursor, Grok, FX) that dojofoo launches with the learner's own sign-in.

The learner and the author are trusted: they run dojofoo on their own machine, and the agents act with their
permissions by design. Treat the following as attacker-controlled:

- **Web pages in the learner's browser.** The local server listens on `localhost` while the app is open. Any site the
  learner visits can send requests to it (cross-site requests, DNS rebinding, `Host`/`Origin` spoofing). The server's
  only cross-origin check is the same-origin CORS handling in `packages/ui/src/server/app.ts`. Its routes read and write
  course files, start agent turns, and run commands (`/api/run-command`, `/api/control`, `/api/interactive`,
  `/api/workspaces`, `/api/authoring`, `/api/project`, `/api/teaching`). Reaching any of them from a web page is the
  most important class of finding.
- **Installed courses.** A course comes from someone else's repository or package. Its manifest, lesson files,
  `SENSEI.md`, `DOJO.md` and file names are untrusted until the learner chooses to run its setup. Installing a course
  (`packages/cli/src/commands/add.ts`) must not write outside its install directory: watch for path traversal, symlinks
  and archive entries that escape it. Running a course's `mise` setup tasks (`packages/config/src/project-preparation.ts`)
  is trusted execution by design, but only for the course the learner chose, and only in its own directory.
- **Lesson trials.** An author can try a lesson in a temporary copy of the course. A trial must not write into the
  author's draft (through links or otherwise) and must not copy secrets such as `.env*`, `.npmrc` or `.netrc`.
- **Model output.** Agent replies, tool calls and structured question/answer payloads are rendered in the browser
  (`packages/ui/src/components`) and parsed by the server. Rendering must not execute script (XSS in chat, markdown,
  code highlights, tool cards or authored lesson fragments).
- **Stored history.** Saved conversations and web state under each workspace's `.dojo/` directory are read back on
  restart; a crafted file must not lead to code execution or writes outside the workspace.

## Components that matter most / least

Most important, in this order:

1. `packages/ui/src/server`: every HTTP route, especially command execution (`run/`, `executor.ts`, `control/`,
   `interactive/`), file reads and writes under `lesson/` and `project/`, and the authoring routes.
2. `packages/cli/src/commands/add.ts`, `update.ts`, `remove.ts`: fetching, extracting and placing courses.
3. `packages/authoring/src/server` (`files.ts`, `service.ts`, including `createLessonTrial`) and `packages/config`.
4. `packages/agent/src/local-*.ts`: local process and sandbox handling for the agents (process ownership, environment
   and credential forwarding into child processes).
5. Chat rendering in `packages/ui/src/components`.

Out of scope:

- That an agent runs commands or edits files in the learner's workspace once the learner has started it. Auto-approval
  is a deliberate product choice. A finding that lets an *attacker* start or steer an agent is in scope.
- Flaws wholly inside the third-party agent CLIs, Eve, the AI SDK or other dependencies. Report them when dojofoo's use
  makes them reachable, and show the path.
- `apps/web` (the static documentation site), `packages/uix` (a component design lab), `packages/evals`, test files,
  fixtures, `e2e/`, `scripts/`, and CI configuration.
- `apps/api` (the hosted telemetry and registry service) is lower priority; report a finding there only if it is
  reachable without credentials.

## How to exercise it

The image has every dependency installed and the packages under `packages/` built. Networking is loopback only, so
nothing can reach a model provider: anything that would start a real agent turn fails to connect, which is expected.

- **Tests**: `cd packages/<name> && pnpm exec vitest run` for `cli`, `config`, `authoring` and `ui`. Exclude
  `**/*.live.test.ts` (they call paid models).
- **The local server**: `cd packages/ui && pnpm exec vite dev --host 127.0.0.1 --port 4317` serves the app and its API.
  Request the routes above with `curl`, and with forged `Origin` and `Host` headers to model a hostile web page.
- **Course installs**: build a malicious course directory or tarball and run
  `node packages/cli/dist/index.js add <path-or-url>` inside a scratch directory.

## How we rate severity

- **Critical**: a web page or an installed course's *files* (without the learner choosing to run its setup) run code,
  run a command, or start or steer an agent on the learner's machine.
- **High**: a web page or a course reads or writes files outside the course or workspace, or reads credentials and
  sign-in tokens; a trial writes into the author's draft.
- **Medium**: XSS in the local app that needs no further user action; reading course or workspace files from a web page;
  stored history that corrupts state.
- **Low**: denial of service against the local server; information leaks without file contents or credentials.

## Report format

Please include the request or course file that triggers the issue, the expected and actual behaviour, and a patch
against `main` with a test, written in the repository's existing style.
