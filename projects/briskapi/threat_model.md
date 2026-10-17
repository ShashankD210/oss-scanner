# Threat model

## What this project does and where untrusted input enters

BRiSKapi is an unofficial Python package and `brisk` command line for the auction data of BRiSK, the market-data web
app that several Japanese brokers offer. The user signs in to their own broker's BRiSK with a passkey, records the
live feed through BRiSK's own decoder, and keeps each day in a private cloud archive that only that user can read.
The archive's two cloud services (AWS Lambda behind function URLs) are in this repository: `archive_service.py`
(uploads, listings, downloads, deletes, restores) and `query_service.py` with `query_engine.py` (a Databento-style
query API over a user's own recordings). They are not in the Python wheel; the rest of the package is.
`ARCHITECTURE.md` describes every component and `docs/cloud-api.md` the query API.

There is no registration. A client makes a random 128-bit secret, sends it as a bearer credential, and the service
derives an owner ID from it (`schema.owner`, SHA-256). Any well-formed secret authenticates, so an attacker can
always act as some new owner. Untrusted input comes from:

- **Any HTTP client of the two cloud services:** the JSON body and `action` fields of the archive service, the query
  string and form body of the query service, and the `Authorization` header. This is the main attack surface.
- **Uploaded recordings.** An upload is a zip of four Parquet tables, validated by the service before it is stored
  (`archive_service.py`, `briskapi/schema.py`, `briskapi/_parquet.py`, the Rust program `rust/brisk_parquet`). Zips,
  Parquet files, manifests and the numbers they declare are attacker-controlled. Content from the public demo must
  equal a pinned reference; a broker's day is uploader-declared and checked only for form.
- **The broker's BRiSK server and the demo site**, over TLS: WebSocket frames, REST responses and the downloaded
  decoder assets (`briskapi/decoder/`, `briskapi/brisk.py`). A broker is a semi-trusted peer, not a hostile one.
- **Downloads of a user's own archive** through presigned URLs (`brisk pull`, `brisk prune`).
- **Local processes on the same computer** that connect to the live gateway (`briskapi/_gateway.py`).

## Components that matter most / least

Most:

- **Owner isolation in `archive_service.py` and `query_service.py`.** One owner reading, listing, downloading,
  deleting, restoring, overwriting, or quota-draining another owner's objects (`archive/OWNER/`, `timing/OWNER/`,
  `bars/OWNER/`, `results/OWNER/`) is the worst outcome. Look at every place a client-supplied `prefix`, key, date,
  ticket, symbol or filename becomes an S3 key, a DynamoDB key or a query: path traversal, prefix confusion
  (one owner's prefix starting with another's), `files.json` entries pointing outside the owner's prefix or
  `reference/`, and the presigned POST and GET URLs (their key, size and expiry binding).
- **Upload validation.** Anything a malicious upload can do to the service (memory or `/tmp` exhaustion past the
  stated limits, a crash that is not a clean rejection, bytes that survive into the archive although verification says
  they cannot, staging objects that linger, one upload replacing a stored recording) and anything it can store that
  breaks the verification rules in `ARCHITECTURE.md` ("Integrity"). The Rust `brisk-parquet verify` and its Python
  reference (`tests/reference_verifier.py`) should agree; a disagreement is a finding.
- **Rate limits and quotas** (`REQUESTS`, `CHANGES`, ticket and byte limits per address, owner and day): bypasses, such as
  address spoofing or a different keying of the same caller.
- **Credentials on the user's computer.** The identity secret (`briskapi/identity.py`: system credential store plus an
  owner-only file) must reach only the configured `https` API and must never follow a redirect. The passkey sign-in
  (`briskapi/passkey.py`, `briskapi/decoder/passkey.cjs`) drives the user's Chrome and obtains the broker's session
  cookies: they must never be written to the archive, logs, the recording or any file other than the owner-only cookie
  store, and never leave for any host but the broker's own.
- **The live gateway** (`_gateway.py`): it must listen on 127.0.0.1 only and demand its random key, kept in an owner-only
  file, before serving anything.
- **Verified download** (`briskapi/cli.py` pull and prune): a manifest or presigned URL that makes the client write
  outside the destination folder, accept bytes that do not match the manifest, or delete a local package it has not
  matched to the archive.
- **Query parameters** (`query_engine.py`): anything in `symbols`, `schema`, `stype_in`, dates and `limit` that reads
  beyond the owner's data, or costs far more than the quotas intend.

Less:

- The decoder hosts' own logic (`briskapi/decoder/*.cjs`) matters for credential handling and for what a hostile frame
  can do to the host, but BRiSK's WebAssembly decoder is the vendor's and out of scope.
- The Rust program `rust/brisk_quote_ingest` and the Python recorder, views and display code run on the user's own
  computer on the user's own data.
- `docs/`, `infra/` (deployment scripts, run only by the maintainer), `tools/` and `tests/` are out of scope, except
  for anything in them that would put a secret into the repository.

## How to exercise it

The repository is at `/src`. The package is installed in editable mode with its test dependencies (`boto3`, `pytest`,
`pandas`), Node 22 is on PATH, and both Rust programs are built in `rust/*/target/release`. The public demo's decoder
files are in `/src/cache/brisk-mock` (`BRISK_MOCK_CACHE` is set). There is no network and no AWS: the tests drive
`archive_service` and `query_service` with in-memory fakes of S3 and DynamoDB, and the demo replay is the test data.

- Python tests: `cd /src && python -m pytest tests -q`. `tests/test_archive.py` and `tests/test_query.py` call the
  two services' handlers directly with request events; a new test there is the best way to show an isolation or
  validation failure. `tests/test_broker_verify.py`, `tests/test_native.py` and `tests/test_package.py` cover
  recordings and `brisk-parquet`.
- Rust: `cargo test --locked --manifest-path rust/brisk_parquet/Cargo.toml` (likewise `rust/brisk_quote_ingest`).
  `brisk-parquet verify DIR` reads a recording's tables from a directory.
- Node: `node --test --test-concurrency=1 tools/brisk_mock/*.test.cjs`. The passkey and web tests that need a real
  Chrome skip here, because the image has none.
- Test fakes and a recording to start from: `tests/test_archive.py` has the `S3` and `DB` fakes (`tests/test_query.py`
  reuses them) and a `packaged` fixture that builds a recording.

## How you rate severity

- **Critical:** one owner reading, modifying or deleting another owner's data; code execution in a Lambda function or
  on a user's computer by a remote party; a stored or sent broker cookie, token or identity secret reaching anyone but
  its owner.
- **High:** a bypass of upload verification that stores data the rules refuse; a way past the per-address or per-owner
  limits that lets one caller exhaust the shared daily quotas for everyone; an unauthenticated or other-owner write
  to any `archive/`, `timing/`, `bars/` or `results/` key; a path traversal on the client; the gateway reachable from
  another computer or without its key.
- **Medium:** denial of service of the caller's own requests or recordings; a malformed upload that crashes ingest
  rather than being refused, where the service stays up; information about another owner that is not their data
  (counts, existence).
- **Low or informational:** hardening advice, and anything that needs an attacker who already holds the victim's
  secret (it is a bearer credential: whoever holds it is that owner).

A finding needs a reproducer that fails against the tests' fakes or against `brisk-parquet`.

## Anything to leave alone

- That any identity authenticates and that identities cost nothing: this is the design. What counts is what a
  caller can reach and what its requests cost.
- That a broker's market values and names in an upload are unverified: only their form is checked, by design.
- A user who runs the client with `BRISK_UPLOAD=0` or the other opt-outs, and the choice to upload by default (a
  documented privacy policy: `PRIVACY.md`).
- Weak local timing measurements (`decode_ns` and the like): they are bounded, not verifiable.
- Vendor code: BRiSK's WebAssembly decoder and the broker sites.
- Style, typing, or performance advice that has no security consequence.
