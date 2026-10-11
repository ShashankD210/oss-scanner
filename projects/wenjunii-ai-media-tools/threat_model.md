# AI Media Scout

This Python application collects third-party repository metadata and web evidence,
builds a searchable static library, and prepares research report delivery. The
public repository also contains a separate Windows app-execution adapter in
`pc_demo/`. The Python package has no third-party runtime dependencies.

## Inputs and boundaries

Treat repository metadata, READMEs, URLs, scraped content, research JSON, imported
catalogs, and browser search input as untrusted. Focus on HTML/script injection in
generated pages, unsafe URL schemes or redirects, filesystem traversal when
importing or exporting records, command construction, and exposure of credentials
or private operational state. Third-party content must not gain publication,
email, or process-execution authority. Review the boundaries between the
research-only collector and the explicit PC execution adapter.

## Offline checks

The checkout is `/src`. Run:

```sh
python -m unittest discover -s tests -v
node --test tests/library.test.cjs
python -m media_scout verify-public-archive
```

Use synthetic repositories, responses, and temporary files for reproducers. Do
not invoke discovery, report generation, GitHub publication, email reservation,
or actual app installation. Live GitHub, Gmail, browser sessions, private local
settings, and Windows application execution are not available in the image.
An offline source/test build does not reproduce those external integrations.

## Severity and reporting

Prioritize exploitable arbitrary execution, credential disclosure, unauthorized
publication, and writes outside an allowed export/install root. State the actual
attacker-controlled input and user interaction needed for any browser injection.
Distinguish local trusted configuration from remotely controlled content.
Provide a minimal offline reproducer and regression test, preserve historical
published reports, and send findings privately to the configured contact.
