# Threat model

## What this project does and where untrusted input enters

A Windows app (C#, .NET 10) that connects a PC to Home Assistant. The same exe runs as a WinForms tray app in each user's session and, optionally, as a Windows service as LocalSystem (`--service`). It publishes sensors and receives commands, notifications and media control.

Untrusted input:

- **Other local users of the PC who are not administrators.** The settings (`C:\ProgramData\HASS.Agent.NET10\settings.json`, holding the Home Assistant token and MQTT password encrypted with machine-scope DPAPI, and the custom commands) can be open to every user of the PC or limited to chosen users (`Configuration/SettingsAccess.cs`). The service runs a user-defined command or command sensor only when an administrator approved it (`SystemService/ServicePolicy.cs`, `service-policy.json` next to the exe in Program Files). Anything a non-administrator can do to get code run as SYSTEM, write where only administrators may, or read the secrets when the folder is limited, is in scope.
- **Home Assistant over its WebSocket API** (`Mqtt/HaWebSocketService.cs`): commands, notifications and media control arrive as events; the app connects with a token (since 10.9.1 normally a non-administrator user of its own).
- **Anything on the MQTT broker** (`Mqtt/MqttCompanionService.cs`): commands, notifications and media control on the PC's topics. The broker's login is the boundary (documented); see "leave alone".
- **The LAN**, through the local HTTP API (`Http/LocalApiServer.cs`, port 5115, API key) used as a notification fallback.
- **The content of notifications**: text, buttons, text fields, and picture URLs the app fetches (`Tray/NotificationImageCache.cs`).
- **Update downloads** from GitHub releases (`Runtime/AppUpdateService.cs`, `Security/InstallerSignature.cs`): an installer runs elevated, from the service as SYSTEM, only when Authenticode-signed by the publisher.

Trusted: the PC's administrators and Home Assistant's administrators.

## Components that matter most / least

- Most: `SystemService/` (the SYSTEM service, the approval policy, elevated command-line handling in `CompanionServiceManager.cs`), `Configuration/SettingsAccess.cs` (folder rules, ownership, links), `Security/` (secret storage, installer signature, token privileges), `Runtime/AppUpdateService.cs` and the service's update path, `Http/LocalApiServer.cs`, `Mqtt/` (message handling, the WebSocket client), `SystemCommands/` (what a remote command can start), `Tray/NotificationImageCache.cs`.
- Less: sensors that only read Windows state (`SystemStatus/`), the WinForms UI layout, localization strings.
- Out of scope: `installer/` (Inno Setup script, reviewed separately), the build and release scripts.

## How to exercise it

On this Linux image the code compiles but does not run (it needs Windows: WinForms, WinRT, the service control manager). `tests/HASS.Agent.NET10.Tests` holds xUnit tests for the logic with inputs and outputs (service policy, settings access, packet checks, notification payloads, picture address rules, the local API key check); they show the intended behaviour, but they also run on Windows only. Reproducers are best written as such tests, or as step-by-step scenarios on a Windows PC.

## How you rate severity

- A non-administrator local user getting code run as SYSTEM or as an administrator, or writing where only administrators may: critical.
- Unauthenticated remote code execution from the LAN or from Home Assistant content: critical.
- Reading the Home Assistant token or MQTT password as a user the settings are not shared with: high.
- An update path that runs something not signed by the publisher: critical.
- Server-side requests to this PC or the LAN from a notification picture, crashing or hanging the tray app: low to medium.
- Needs the PC's or Home Assistant's administrator rights already: out of scope.

## Anything to leave alone

These are by design and documented:

- Any client with the MQTT broker's login can send this PC commands and notifications on its topics; the docs tell users to keep that login private and point to broker ACLs. MQTT TLS is optional.
- The local HTTP API is plain HTTP on the LAN, bound to all interfaces; `/info` answers without the key (the integration checks the PC's serial with it). A 127.0.0.1 bind is reset for old settings.
- In "open" mode (every user of the PC may use the settings) every local user can read the settings and the DPAPI-protected secrets; limiting the folder is the user's choice.
- A custom command or script doing what its owner set it up to do; built-in commands such as shutdown run without approval.
- LibreHardwareMonitor's web server is contacted over plain HTTP, with basic auth when configured.
