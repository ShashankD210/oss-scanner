# Home Podcast

This Python pipeline imports archived stories, plans synthetic-host podcast
episodes, handles provider-generated text/audio, assembles media, and exports
transcripts and manifests. Its core has no third-party Python dependencies;
FFmpeg is available for local media handling.

## Inputs and boundaries

Treat imported story JSON/Markdown, original web/archive fields, generated cards,
provider responses, media metadata and filenames as untrusted. Focus on path
traversal and symlink escapes in imports/exports/cache handling, command argument
construction for media tools, malformed structured data, HTML/transcript
injection, and disclosure of credentials in logs or artifacts. Review cache and
manifest identity checks that prevent one story or media response overwriting
another. Model-generated text must not silently become shell or publishing
authority.

## Offline checks

The checkout is `/src`. Run:

```sh
python -m unittest discover -s tests -v
python -m compileall -q home_podcast
```

Use tiny synthetic story/media fixtures and mocked providers. Do not execute
production ingestion, paid text/TTS/sound generation, publication, or live
TouchDesigner show controls. API credentials, paid providers, live workstation
state and licensed hosts are unavailable. The Linux tests cover portable source
and mocked adapters; they do not establish end-to-end provider or installation
behavior. Preserve committed story exports and production artifacts.

## Severity and reporting

Prioritize code execution, file writes outside authorized working roots,
credential leakage, and attacker-controlled script in an exported document.
Distinguish intentionally trusted operator configuration from external content.
Provide a minimal offline reproducer, actual prerequisites and impact, and a
regression test with a focused patch. Keep findings private.
