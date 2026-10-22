# Threat model — bancolombia/django-DefectDojo

## What this project does and where untrusted input enters

Vulnerability management platform (Django): findings, engagements, importers (SARIF/XML/CSV),
REST API v2, multi-tenant by product. **Imported scan data and API input are untrusted.**

## Components that matter most

`dojo/` application code, API views/serializers, importers, authentication/authorization,
file uploads, Celery tasks. K8s/Helm-only deploy artifacts are out of scope unless shipped.

## How to exercise it

`python manage.py test unittests` with PostgreSQL (see `docker/entrypoint-unit-tests.sh`).
Full tree available under `/src` after build.

## Severity guidance

Critical: unauthenticated RCE, auth bypass, cross-tenant IDOR with confirmed impact.
High: SQLi, stored XSS affecting other users, SSRF to internal networks, unsafe import parsing.
Medium: CSRF on sensitive actions, DoS on importers. Low: informational misconfig without exploit.

## Out of scope

Dependency CVEs without reachable path; test-only fixtures in `unittests/` without prod effect.
