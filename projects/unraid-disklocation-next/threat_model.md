# Threat model

This is an Unraid plugin. Its Node daemon reads the Unraid GraphQL API and serves a local-only HTTP endpoint behind an nginx include on a home or small-office NAS. The plugin runs as root on a home or small-office NAS. Its pages and API
endpoints are served by Unraid's own web server behind Unraid's login and CSRF token, and the server is
normally reachable only from the local network.

## In scope
- Privilege escalation or code execution by a user who can only reach the web UI as a normal logged-in
  user, or by anyone unauthenticated.
- Missing authentication or CSRF checks on endpoints that change state or run commands.
- Command, argument, path and SQL injection from request parameters, uploaded files, filenames, and
  settings or config files (including values that another local user or container could write).
- Secrets (passwords, API keys, tokens) written to logs, world-readable files or command lines.
- Unsafe handling of files in shared locations (symlink and temp-file races, world-writable paths used as root).
- Cross-site scripting and unsafe HTML in anything rendered into the webGUI.

## Out of scope
- Attacks that already need root on the Unraid server, or write access to /boot.
- Denial of service by a logged-in administrator against their own server.
- Issues in Unraid itself or in third-party tools the plugin calls.

## Severity guide
- Critical: unauthenticated remote code execution or root file write.
- High: authenticated-user to root escalation, or injection reachable by a low-privilege local user.
- Medium: stored XSS, secret disclosure to other local users, CSRF on a state-changing endpoint.
- Low: hardening gaps with no demonstrated impact.

## Reports
A short proof of concept and a minimal patch are most useful.
