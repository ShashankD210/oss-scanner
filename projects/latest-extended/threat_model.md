# Threat model

A macOS app that checks for and installs updates. Treat version strings, appcast and repository metadata, and downloaded package names as untrusted input. High: code execution or unsigned or tampered update installation. Out of scope: Sparkle, vendored frameworks and the macOS SDK.
