# Threat model

## What this project does and where untrusted input enters

A Home Assistant custom integration (`custom_components/hass_agent`) that turns Windows PCs running the HASS.Agent .NET10 client into Home Assistant devices: sensors, buttons, a media player, notifications, events, an update entity. A PC talks to it in one of three ways:

- **MQTT.** The client publishes discovery (`hass.agent/devices/<serial>`), states and events; the integration subscribes and builds entities. Anything that can publish to the broker can send these messages, so every MQTT payload is untrusted: discovery, sensor values, entity names, attributes, notification actions, hotkey events.
- **The Home Assistant WebSocket API.** The client connects with a Home Assistant token. Since 10.9.1 that is normally a user who is not an administrator; the integration's own commands (`hass_agent/fire`, `hass_agent/subscribe`, `hass_agent/provision` in `ws_commands.py`) decide what such a user may do. The token's user is untrusted beyond what its entry approves.
- **The local HTTP API** of the client (`/info`, `/notify`): the integration calls it; its answers are untrusted (another device may answer at that address).

Trusted: Home Assistant itself and its administrators. Not trusted: the PCs, anything on the MQTT broker, Home Assistant users who are not administrators (they can call the integration's services), and data coming back from a PC.

## Components that matter most / least

- Most: `ws_commands.py` (who may send or receive what for which PC; provisioning a user and token), `config_flow.py` (discovery, approving a user, switching transports, names), `__init__.py` (MQTT discovery handling, the local API path, services), `notify.py` (which picture paths get signed, sending to the right PC), `event.py` and `device_trigger.py` (which PC an event is attributed to), `update.py` (the release link).
- Less: entity platforms that only display values (`sensor.py`, `light.py`, `select.py`, `button.py`, `media_player.py`), translations.

## How to exercise it

`python -m pytest` in `/src` runs the suite on an in-memory Home Assistant (pytest-homeassistant-custom-component, mocked MQTT, no network). `tests/test_hardening.py` holds what a PC or a broker client must not be able to do, `tests/test_ws_commands.py` the WebSocket command rules; both are good starting points for reproducers.

## How you rate severity

- A Home Assistant user who is not an administrator, or a PC with such a user's token, gaining administrator rights, another user's token, files of the Home Assistant host, or control of another PC: critical.
- One PC (or a broker client) acting as another PC, receiving its notifications or commands, or firing its automations: high.
- A signed Home Assistant link (picture path) for something other than a picture: high.
- Denial of service of Home Assistant through the integration: medium.
- Needs Home Assistant administrator rights: out of scope.

## Anything to leave alone

These were looked at and accepted:

- A client on the broker can make entities disappear by publishing a changed discovery message; they come back with the PC's next one. The broker's credentials are the boundary here (see the client's docs on keeping the MQTT login private).
- Markdown in a notification text sent over MQTT is shown as Markdown.
- No size limits on discovery payloads beyond what MQTT and Home Assistant enforce.
- The local HTTP API is plain HTTP on the LAN by design; `/info` answers without a key (its serial number is how the integration checks it talks to the right PC).
