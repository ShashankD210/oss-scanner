# WIT Calendar backend threat model

## What the project does

WIT Calendar is a Rails 8 app for students at Wentworth Institute of Technology. It imports the WIT course catalog and
syncs each student's classes to Google Calendar or Microsoft Outlook. A browser extension, a web dashboard and an MCP
server use it. The app keeps Google and Microsoft OAuth tokens for each student, encrypted with Active Record
Encryption.

`docs/architecture.md` tells where each domain lives. `CLAUDE.md` gives the rules for each controller family.

## Who the attacker is

- An unauthenticated person on the Internet.
- A signed-in student who tries to read or change the data of another student.
- An MCP client or a dynamically registered OAuth client with fewer scopes than the action needs.
- A person who has only a public link (an ICS feed token or a one-time meeting link).

Admins (`Admin::` controllers) are trusted. A student who becomes an admin, or who reaches an admin action, is in
scope.

## Where untrusted input enters

- `/api/*` (`app/controllers/api/`): the extension API. `Api::BaseController` requires no token. Each controller that
  needs a user calls `authenticate_with_token`. Public actions are listed in `PUBLIC_ACTIONS` in
  `spec/requests/api/authentication_spec.rb`.
- `/api/mcp` and the MCP tools (`app/services/mcp_tools/`): each tool names its scope with `requires_scope`.
- `/api/graphql` (`app/graphql/`): the public catalog.
- The OAuth 2.1 server for MCP clients: `/api/oauth/*` (Doorkeeper), dynamic client registration at
  `/api/oauth/register`, and the metadata under `/.well-known/`.
- Sign-in: Google OAuth callback, Microsoft sign-in, passkeys (`/users/passkey/*`), the account-link confirm flow at
  `/oauth/confirm`, and Devise sessions.
- `/dashboard/*`: the student dashboard. Every action calls Pundit `authorize` or `skip_authorization`.
- `/calendar/:calendar_token`: the ICS feed. The token is the only credential.
- Public meeting link pages (`MeetingLinks`): a person with the link sees free times and books one meeting.
- Friends: friend requests, friend groups and the visibility level (`full` or `availability_only`).
- `/risc/events`: Google RISC security events, signed by Google.
- Data that the app fetches from outside systems: LeopardWeb, 25Live, RateMyProfessor, finals schedule PDFs and the
  university event calendar. Treat this data as untrusted when the app renders or parses it.

## Components that matter most

1. Authentication and token checks: API bearer tokens (`JsonWebTokenService`), Devise sessions, passkeys, Google and
   Microsoft sign-in and account linking.
2. Authorization: Pundit policies in `app/policies/`, the `Admin::` and `Dashboard::` base controllers, MCP scopes.
3. The OAuth server: redirect URI checks, PKCE, client registration, consent, token scopes.
4. Storage and use of the Google and Microsoft OAuth tokens (`OauthCredentials`).
5. Friends privacy: a friend with `availability_only` must not read course details.
6. ICS feed tokens and meeting link tokens.

## Components that matter least

- The admin panel when the attacker is already an admin.
- Catalog reports under `/reports/*` (public course data).
- Rake tasks, migrations, `scripts/` and `bin/`.

## How to exercise it

Run `witcal-services-start` first. Then run specs from `/src`:

- `bundle exec rspec spec/requests/api/authentication_spec.rb`
- `bundle exec rspec spec/requests/admin/authorization_spec.rb`
- `bundle exec rspec spec/requests/dashboard/authorization_spec.rb`
- `bundle exec rspec spec/policies`

Request specs in `spec/requests/` drive the full middleware stack. WebMock blocks all real HTTP. Stub outside calls
with `stub_request`.

## How we rate severity

- Critical: authentication bypass; access to another student's OAuth tokens; remote code execution; SQL injection.
- High: reading or changing another student's schedule, calendar, friends or settings; a non-admin reaching an admin
  action; an OAuth client getting a scope it was not granted; redirect URI bypass that leaks a code or token; stored
  XSS on the dashboard or the admin panel.
- Medium: a friend with `availability_only` reading course details; ICS feed or meeting link token prediction or
  reuse; reflected XSS; CSRF on a state-changing action; SSRF from the catalog importers.
- Low: information disclosure of public catalog data; issues that need a non-default configuration.

## Reports and patches

- Name the controller, policy or service and the route.
- Reproduce with a request spec in `spec/requests/`, built with factory_bot.
- Keep a patch small and follow the rules in `CLAUDE.md`.

## Do not report

- Denial of service from floods of requests. Rack::Attack handles rate limits.
- Missing security headers with no exploit.
- Bugs in Google, Microsoft or WIT systems. Report a bug in how this app calls them.
- Findings in third-party gems. Report them to the gem.
