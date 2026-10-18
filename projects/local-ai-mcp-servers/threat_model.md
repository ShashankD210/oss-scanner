# Threat model: local-ai-mcp-servers (mcp-modelmanager, mcp-ollama-vllm)

Two MCP servers an AI agent (e.g. Claude) talks to over stdio. They let the
agent operate self-hosted LLM infrastructure. Core promise: the agent gets a
CLOSED catalog of operations, never a free command or code channel.

## Entry points / untrusted input
1. **MCP tool parameters** (both servers). The caller is an AI agent whose
   input may be attacker-influenced via prompt injection. ALL tool parameters
   are untrusted.
2. **Responses from the managed machine** (mcp-modelmanager): output of the
   VM-side wrapper over SSH, and HTTP responses from Ollama/vLLM services.
3. **HTTP responses of local Ollama/vLLM endpoints** (mcp-ollama-vllm),
   including model-generated text that flows back to the agent.

## Trust boundaries and design invariants (violations = findings)
- **No shell injection:** mcp-modelmanager executes only templated operations
  from a fixed catalog (operations.py) after allowlist validation
  (validation.py); a tool parameter must never become shell syntax on the
  local side or on the VM side (vm_side/ wrapper enforces
  SSH_ORIGINAL_COMMAND dispatch). Any escape from the catalog is CRITICAL.
- **Two-channel SSH:** the control channel uses forced commands; keys and SSH
  config of the host are reused, no own key handling. Bypassing the forced
  command or widening it is CRITICAL.
- **No secret leakage:** no credentials, key material or tokens may appear in
  tool results or logs. HIGH.
- **Pinned endpoints:** mcp-ollama-vllm talks only to configured local
  backends; a parameter must not redirect requests to arbitrary hosts (SSRF).
  HIGH.
- **Resource safety:** operations that pull/remove models or switch services
  must stay within the catalog semantics; destructive operations outside the
  declared set are HIGH.

## Severity guidance
- CRITICAL: parameter-to-shell escape, forced-command bypass, arbitrary file
  read/write on host or VM.
- HIGH: SSRF to unconfigured hosts, secret leakage, undeclared destructive
  operations.
- MEDIUM: validation gaps without direct escalation, protocol confusion
  between the two channels.
- LOW: denial of service against the local servers themselves (single-user,
  localhost stdio).

## Out of scope
- Security of the Ollama/vLLM upstream projects themselves.
- The operator's machine/VM hardening guides (SETUP-MACHINE.md etc.) - they
  are documentation, not code.
- Availability of the model machine.
