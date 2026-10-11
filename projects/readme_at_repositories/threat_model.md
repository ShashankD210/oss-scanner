# README at Repositories security review

## Purpose and trust boundaries

This Redmine plugin displays repository README contents in the repository tab
and provides project-level display settings. The plugin source is `/src`; the
host Redmine installation is `/opt/redmine`, with the plugin symlinked into its
`plugins/readme_at_repositories` directory.

Treat repository README contents, filenames, paths, revisions, repository
identifiers, and HTTP parameters as potentially attacker-controlled. Repository
contributors need not be Redmine administrators, and a repository viewer can
have more privileges than the contributor who authored a README. Distinguish
anonymous users, project members, repository contributors, project managers,
and site administrators when demonstrating impact.

Review rendering and escaping for Markdown and non-Markdown README variants,
the `DisplayReadme` hook, repository access checks, settings authorization and
CSRF protection, and model/controller handling of project identifiers. Check
cross-project access and unintended filesystem or command operations where
reachable. Do not assume a finding merely because a template marks HTML safe;
trace the formatter and sanitization path and demonstrate executable behavior.

## Environment and exercising the project

The image uses Ruby 3.1.6, Redmine 5.1.12, and a local SQLite test database. Ruby
gems and source are installed during the online build. From `/opt/redmine`, run:

    bundle exec rake redmine:plugins:test NAME=readme_at_repositories
    bundle exec rails runner 'puts Redmine::Plugin.find(:readme_at_repositories).name'

`RAILS_ENV=test` is set by the image. Core and plugin migrations are applied.
The existing plugin test is a placeholder assertion, so passing it does not
provide meaningful security coverage. Add focused Rails tests and create local
repository fixtures as needed; Git and Subversion are installed. No external
database, service credentials, or live repository service is required.

## Impact and reporting

Prioritize stored or reflected script execution in a victim's Redmine session,
authorization bypass, cross-project data disclosure or modification, and server
code execution. Explain the attacker's permissions, required Redmine settings,
README format, affected viewer role, and concrete data or privilege impact.
For availability findings, quantify payload sizes and server impact.

Include a minimal local repository or request reproducer, affected code, the
tested formatter and host version, and a focused fix with a regression test.
Deduplicate by root cause. Redmine/Rails bugs unrelated to the plugin and generic
dependency advisories without a demonstrated plugin path are out of scope.
