# Threat model

## Purpose and trust boundaries

This plain-JavaScript ESM plugin connects Letta Code to a configured remote MCP server using Streamable HTTP. `plugin.mjs` subscribes to resource URIs, handles resource-updated/list-changed notifications, reads resource bodies, and emits inbound Letta channel messages. Account configuration supplies the endpoint URL, optional HTTP headers, resource/chat mappings, and optional bootstrap tool calls. `rules.json` supplies fetch, truncation, default routing, and reconnect settings. Treat administrator-owned account/rules configuration separately from untrusted remote server responses.

Remote resource URIs, list entries, notification parameters, text bodies, server errors, and tool results are untrusted. Resource text is forwarded into `text` and `raw.content`; it is not sanitized or rendered by this plugin. A resource body is data, not authority to issue instructions or access secrets. Downstream agent prompt injection is a relevant boundary to examine, but report a demonstrated crossing of a security boundary rather than treating all forwarded text as a vulnerability.

## Security-relevant components

Focus on `plugin.mjs`: connection/header handling, subscription-to-chat routing, auto-derived subscriptions, resource fetch/truncation, deduplication, reconnect lifecycle, notification handlers, logging, and callback failures. `max_resource_chars` limits the joined text after fetching, not the size of the remote HTTP response or all intermediary allocations. Examine resource exhaustion and concurrency using bounded local fixtures.

`sendMessage`, `sendDirectReply`, and `messageActions.handleAction` suppress outbound channel replies. This is not a guarantee that the process cannot make outbound calls: configured `bootstrap_tools` invoke tools at connection time. Separately configured native Letta MCP tools are another integration, outside this plugin's channel implementation. MCP bearer headers and host/API credentials must not leak to resource messages, logs, other endpoints, or unintended redirects.

## Build and offline exercise

The checkout stays at `/src`. There is no package.json, lockfile, compilation command, or checked-in test suite. The actual host setup is `letta channels install mcp`, which consumes the runtime dependency declaration in `channel.json`. The image installs that declared MCP SDK into `/opt/scanner`, exposes its modules to `/src/plugin.mjs`, and runs `node --check /src/plugin.mjs`; it does not invent an npm test/build script.

Run the following inside the built image with Internet networking disabled:

```sh
node --check /src/plugin.mjs
node /opt/scanner/smoke.mjs
```

The enrollment-specific smoke is not an upstream suite. It runs the real SDK client and server transports on an ephemeral 127.0.0.1 HTTP port, imports the unchanged plugin, subscribes to a synthetic resource, issues two changing resource notifications, and asserts channel/account/chat routing and fetched body delivery. No account files, bearer tokens, bootstrap tools, external MCP server, or Letta key are used. All source remains available for audit; the smoke does not cover the complete reconnect, auto-subscription, or host routing lifecycle.

The actual Letta host is not launched in this image. Its documented full integration requires Letta Code's dynamic channel support (README references PR #2021 and 0.25.x), a Letta API key, an agent, and routing configuration. These host/credential-dependent flows are not claimed to be exercised by the standalone smoke. Do not supply real credentials or contact a hosted API. Source currently deduplicates repeated null content even with fetch disabled; README's claim that every fetch-disabled notification is forwarded should not be treated as a tested guarantee.

## Severity guidance and exclusions

Credential exfiltration, unauthorized cross-account/chat delivery, or execution outside intended channel permissions can be high severity when an attacker-controlled input and exploit path are demonstrated. Remote code execution with meaningful process privileges can be critical; do not assume it from arbitrary resource text alone. Sustained remotely induced resource exhaustion is generally medium, adjusted for demonstrated impact and recovery. Trusted administrator URL/header configuration alone is not an unauthenticated SSRF primitive.

Never run the README's production host commands, use real account files/API keys, invoke side-effecting bootstrap tools, contact production endpoints, or deploy. Third-party SDK behavior should be distinguished from defects in the channel integration, without excluding dependency-mediated boundary violations.
