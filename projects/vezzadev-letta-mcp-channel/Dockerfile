# syntax=docker/dockerfile:1
FROM node:24-bookworm
ENV NODE_ENV=development
COPY . /src
WORKDIR /src
# This repository is an ESM plugin, not an npm project. Mirror the dependency
# declaration consumed by `letta channels install mcp`, without a host/API key.
RUN mkdir -p /opt/scanner \
    && npm install --prefix /opt/scanner --no-save --package-lock=false $(node -e 'console.log(require("./channel.json").runtimePackages.join(" "))') \
    && ln -s /opt/scanner/node_modules /src/node_modules
# No compilation or checked-in test suite exists; syntax-check the real entry.
RUN node --check /src/plugin.mjs
# Standalone enrollment smoke: real MCP SDK and adapter; loopback server only.
COPY <<'EOF' /opt/scanner/smoke.mjs
import assert from 'node:assert/strict';
import { createServer } from 'node:http';
import { randomUUID } from 'node:crypto';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { ReadResourceRequestSchema, SubscribeRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import { channelPlugin } from '/src/plugin.mjs';

const uri = 'scanner://local/resource';
let content = 'local untrusted MCP text <script>not executed</script>';
let subscribed = false;
let resolveMessage;
const messages = [];
const mcp = new Server({ name: 'scanner-local-mock', version: '1.0.0' }, {
  capabilities: { resources: { subscribe: true } },
});
mcp.setRequestHandler(SubscribeRequestSchema, async request => {
  assert.equal(request.params.uri, uri);
  subscribed = true;
  return {};
});
mcp.setRequestHandler(ReadResourceRequestSchema, async request => {
  assert.equal(request.params.uri, uri);
  return { contents: [{ uri, mimeType: 'text/plain', text: content }] };
});
const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: randomUUID });
await mcp.connect(transport);
const http = createServer((req, res) => {
  transport.handleRequest(req, res).catch(error => {
    console.error(error);
    if (!res.headersSent) res.writeHead(500);
    res.end();
  });
});
await new Promise(resolve => http.listen(0, '127.0.0.1', resolve));
const adapter = await channelPlugin.createAdapter({
  accountId: 'scanner-local',
  config: {
    url: `http://127.0.0.1:${http.address().port}/mcp`,
    subscriptions: [{ uri, chatId: 'local-inbox' }],
  },
});
adapter.onMessage = async message => {
  messages.push(message);
  resolveMessage?.(message);
};
async function updateAndReceive() {
  let timer;
  const received = new Promise((resolve, reject) => {
    resolveMessage = resolve;
    timer = setTimeout(() => reject(new Error('resource delivery timed out')), 5000);
  });
  try {
    await mcp.notification({ method: 'notifications/resources/updated', params: { uri } });
    return await received;
  } finally {
    clearTimeout(timer);
    resolveMessage = undefined;
  }
}
try {
  await adapter.start();
  assert.equal(subscribed, true, 'adapter must complete MCP subscription');
  const first = await updateAndReceive();
  assert.equal(first.channel, 'mcp');
  assert.equal(first.accountId, 'scanner-local');
  assert.equal(first.chatId, 'local-inbox');
  assert.deepEqual(first.raw, { uri, content });
  assert.ok(first.text.includes(content));
  content = 'changed local resource body';
  const second = await updateAndReceive();
  assert.equal(second.raw.content, content);
  assert.equal(messages.length, 2);
  console.log('PASS: local MCP subscribe/read/update -> channel resource-content delivery');
} finally {
  await adapter.stop();
  await mcp.close();
  http.closeAllConnections();
  await new Promise(resolve => http.close(resolve));
}
EOF
