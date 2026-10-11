# Voice to Visual SDTD

This Python application turns live speech into visual prompts and communicates
with TouchDesigner through OSC. It supports local and hosted transcription
backends, bounded audio/result queues, runtime controls, and diagnostic output.

## Inputs and boundaries

Treat incoming OSC packets, audio-derived text, provider responses, prompts,
runtime control values, and external configuration files as untrusted according
to their origin. Focus on OSC bind/access assumptions, unauthorized control,
queue/resource exhaustion, malformed messages, retries/stale-result handling,
log or diagnostic disclosure of credentials, and any command/path construction.
Inspect `osc_control.py`, `osc_output.py`, `runtime_config.py`, `diagnostics.py`,
`runtime_logging.py`, `orchestrator.py`, `streaming_core.py`, and transcription
backend adapters. Document whether an attack requires LAN access, an explicitly
non-loopback bind, or operator control of trusted configuration.

## Offline checks and runtime limits

The checkout is `/src`. Run `python -m unittest discover -s tests -v`.
The image uses the repository's lightweight CI dependency profile; hardware and
model runtimes are mocked. It supports source review and protocol/control tests,
not end-to-end audio capture, GPU transcription, paid APIs, or TouchDesigner.
Use synthetic audio/text, mocked providers, in-memory publishers, and loopback
UDP. Do not contact hosted providers, activate devices, or expose OSC externally.
Model weights, audio drivers, CUDA, API credentials, and optional runtime profiles
would need separate environments for integration-specific findings.

## Severity and reporting

Prioritize remote execution, secret disclosure, unauthorized controls crossing a
documented trust boundary, and persistent disruption from small attacker inputs.
Include a deterministic offline reproducer, required deployment settings, and a
minimal regression-tested patch. Keep security reports private.
