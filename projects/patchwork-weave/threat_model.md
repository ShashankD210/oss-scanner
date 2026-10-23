# Weave threat model

## What the project does

Weave is the identity provider for Patchwork Labs. It is a Rails 8 app and an OAuth 2.0 and OpenID Connect server
(Doorkeeper and doorkeeper-openid_connect). Every Patchwork Labs app signs in through Weave, and Google Workspace can
use it as its sign-in provider. A flaw in Weave can give access to every app that trusts it.

`SECURITY.md` gives the official scope. `docs/OAUTH.md`, `docs/PASSKEYS.md`, `docs/GUARDIAN_CONSENT.md` and
`docs/ACTIVE_DIRECTORY.md` describe the main flows.

## Who the attacker is

- An unauthenticated person on the Internet.
- A signed-in member who tries to act as another member or as an admin.
- A registered OAuth client that tries to get more scopes, or tokens for another client.
- A person with a link from an email: a magic link, an email confirmation link or a guardian consent link.

Admins are trusted. A non-admin who reaches `/admin`, or the engines mounted there, is in scope.

## Where untrusted input enters

- OAuth and OIDC: `/oauth/authorize`, `/oauth/token`, `/oauth/introspect`, `/oauth/revoke`, `/oauth/userinfo`,
  WebFinger, and the discovery documents. Custom controllers are in `app/controllers/oauth/`.
- Sign-in: `/login`, password sign-in, magic links (`/auth/magic_link/*`), passkeys (`/auth/passkey*`), logout and
  sessions (`SessionAuthenticator`).
- Sign-up, profile, email addresses, password change and expired password pages.
- Guardian consent: `/guardian/:token`, and the Slack onboarding steps under `/slack`.
- Profile photos and avatars under `/user/:p_id/*`.
- `/api/v1/*`: the programmatic API.
- `/webhooks/slack/*`: signed Slack requests (`SlackSignatureVerification`).
- `/admin/*`: guarded by `AdminConstraint`. Blazer, Flipper, Mission Control and Audits1984 are mounted there and have
  no authentication of their own.
- Data from Active Directory over LDAPS, when that feature is on.

## Components that matter most

1. Session and authentication logic: `SessionAuthenticator`, magic links, passkeys, passwords, impersonation and
   session revocation.
2. OAuth and OIDC: redirect URI checks, PKCE, consent, scopes and app scopes, token and ID token issue, introspection
   and revocation, app access rules (`AppAccess`, groups and roles).
3. `AdminConstraint` and admin authorization.
4. Guardian consent and the minimum age rule.
5. Encrypted data (Lockbox, blind indexes) and public ids (`encoded_ids`).
6. Slack webhook signature checks.

## Components that matter least

- The marketing and home pages.
- Rake tasks, `bin/` scripts and the Rails console (audited by Console1984).

## How to exercise it

Run `weave-services-start` first. Then run tests from `/src`:

- `bundle exec rspec spec/requests/oauth/`
- `bundle exec rspec spec/requests/magic_link_sign_in_spec.rb`
- `bundle exec rspec spec/requests/impersonation_session_spec.rb`
- `bin/rails test test/integration`

RSpec (`spec/`) and Minitest (`test/`) both exist. Request specs drive the full middleware stack. The test
environment uses a throwaway Lockbox key and generates a test OIDC signing key.

## How we rate severity

- Critical: authentication bypass; signing in as another member; forging or stealing an access token, refresh token or
  ID token; a non-admin reaching `/admin` or a mounted engine; remote code execution; SQL injection.
- High: an OAuth client getting scopes or app access it was not granted; redirect URI bypass that leaks a code or
  token; impersonation that is not recorded; session revocation that does not revoke; stored XSS; Slack signature
  bypass.
- Medium: guardian consent bypass for a member under 18; minimum age bypass; account enumeration that shows private
  data; reflected XSS; CSRF on a state-changing action; open redirect.
- Low: information disclosure with no private data; issues that need a non-default configuration.

## Reports and patches

- Name the controller, service or initializer, and the route.
- Reproduce with a request spec in `spec/requests/`.
- Keep a patch small and follow `CLAUDE.md`.

## Do not report

- Denial of service and volume-based attacks.
- Missing security headers or rate limits with no exploit.
- Bugs in Slack, Google, GitHub or other third-party services.
- Findings in third-party gems. Report a bug in how Weave calls them.
