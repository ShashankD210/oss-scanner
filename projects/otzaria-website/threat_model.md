# Threat model

## What this project does and where untrusted input enters
The Otzaria website (Next.js App Router, MongoDB via Mongoose, NextAuth) serves app downloads and update metadata, a plugin store, a volunteer area for proofreading scanned books (OCR, page editing, text corrections), and endpoints the Otzaria desktop app calls (error and app reports, search feedback, plugin reports). Untrusted input enters through:
- **Every route under `src/app/api/`**, from anonymous visitors, from the desktop app (anonymous), and from logged-in volunteers. Admin routes (`api/admin`, role checks) must not be reachable by regular users.
- **Plugin uploads** (`api/plugins`, `src/lib/plugin*`): developers upload zip archives that are unpacked, validated and published to every app user. Archive extraction, the validator, and anything that lets a malicious plugin bypass validation or overwrite another developer's plugin matter most.
- **File uploads**: books, page images and PDFs (`api/upload-book`, OCR routes) are stored under `UPLOAD_DIR` and served from `public/`.
- **Webhooks and cron** (`api/cron`, `api/app-reports/github-webhook`, `api/app-reports/inbound-email`, `api/revalidate-wiki`) are protected by shared secrets.
- **Outbound requests** to OCR services, GitHub and Gemini built from user input.

## Components that matter most / least
- Most: authentication, session and role checks; plugin archive handling and publishing; file upload paths; NoSQL injection in Mongoose queries; stored XSS in content other users see (messages, corrections, book pages, plugin listings); SSRF from outbound fetches; rate limiting and client IP handling (`src/lib/client-ip.js`).
- Least: static marketing pages, styling, `scripts/` that only maintainers run by hand, `workers/` deployed separately on Cloudflare.

## How to exercise it
- `npm test` runs the Node and Vitest suites; `npm run test:plugin-safety` covers plugin archive and validator safety.
- To run the server: `mkdir -p /tmp/db && mongod --fork --logpath /tmp/mongod.log --dbpath /tmp/db`, then `npm start` (port 3000). `scripts/create-admin.js` creates a first admin from `ADMIN_EMAIL`/`ADMIN_PASSWORD`.

## How you rate severity
- Critical: remote code execution; unauthenticated admin access; publishing or replacing a plugin that is then delivered to app users; arbitrary file write on the server.
- High: privilege escalation from volunteer to admin; account takeover; stored XSS reachable by admins or other users; NoSQL injection that reads or changes other users' data; SSRF to internal services.
- Medium: reflected XSS, IDOR on non-sensitive data, rate-limit bypass, denial of service from a single request (ReDoS, unbounded upload or decompression).
- Low: issues that need a malicious admin, missing security headers without a demonstrated exploit.

## Anything to leave alone
- The placeholder `NEXTAUTH_SECRET` in the scanner image is not a finding; production secrets come from the environment.
