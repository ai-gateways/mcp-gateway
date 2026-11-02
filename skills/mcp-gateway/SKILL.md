---
name: mcp-gateway
description: Set up an MCP gateway that connects MCP servers over STDIO, HTTP, or SSE and serves all their tools from one /mcp endpoint to Claude Code, Cursor, Claude Desktop, and other MCP clients, using Bifrost, the open-source AI gateway by Maxim AI. Covers per-key tool filtering, upstream auth through static headers or OAuth 2.0, Code Mode to cut tool-definition tokens, and handling tool output as untrusted input. Use when the user wants an MCP gateway, to aggregate or centralize MCP servers behind one URL, control which MCP tools each user, team, or agent can call, keep MCP server credentials out of client configs, secure or audit MCP tool calls, or reduce token usage from large MCP tool lists.
compatibility: Runs Bifrost with npx (Node.js) or Docker. STDIO MCP servers need their command (npx, uvx, python) available where Bifrost runs.
metadata:
  author: Maxim AI
  version: "1.0"
---

# MCP gateway

An MCP gateway connects to upstream MCP servers as a client and acts as an MCP server itself. MCP clients connect once to the gateway and see only the tools their key allows, while the upstream credentials stay in the gateway.

This skill builds that gateway with Bifrost, the open-source AI gateway (Apache 2.0) built by Maxim AI, which serves the aggregated tools at `/mcp`.

```
Claude Code / Cursor / Claude Desktop
        |  MCP over HTTP (POST /mcp) or SSE (GET /mcp), virtual key header
        v
     Bifrost  --- tool filter per virtual key, upstream auth, logs
        |  STDIO / HTTP / SSE
        v
  MCP servers (filesystem, GitHub, internal APIs, ...)
```

For model traffic use the `llm-gateway` skill; for A2A agents, `agent-gateway`; for all three, `ai-gateway`.

## When to use

- Several MCP servers should sit behind one URL instead of being configured in every client.
- Different users, teams, or agents need different tool sets from the same servers.
- Upstream MCP API keys or OAuth tokens should not live on developer machines.
- Many tools are connected (roughly 3 or more servers) and tool definitions are eating context; use Code Mode.

## Steps

### 1. Start Bifrost

```bash
npx -y @maximhq/bifrost
# or
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
```

STDIO servers may fail in Docker when the container lacks their command (`npx`, `python`). Run Bifrost with npx, build an image that adds those runtimes, or connect to HTTP or SSE servers instead.

### 2. Connect MCP servers

MCP client names must be ASCII, unique, free of hyphens and spaces, and must not start with a digit (`filesystem` and `web_search` are valid; `web-search` is not). `tools_to_execute` decides which of a server's tools the gateway exposes at all: a list exposes only those tools, `["*"]` exposes every tool, and `[]` exposes none. Prefer an explicit list once you have reviewed a server's tools.

Declare servers in `config.json` in the Bifrost app directory (`~/.config/bifrost` for npx, `./data` for the Docker command above). This works for every connection type without dashboard auth:

```json
{
  "mcp": {
    "client_configs": [
      {
        "name": "filesystem",
        "connection_type": "stdio",
        "stdio_config": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/srv/shared"],
          "envs": ["HOME", "PATH"]
        },
        "tools_to_execute": ["read_text_file", "list_directory", "search_files"]
      },
      {
        "name": "docs_search",
        "connection_type": "http",
        "connection_string": "env.DOCS_MCP_URL",
        "auth_type": "headers",
        "headers": {"Authorization": "env.DOCS_MCP_AUTHORIZATION"},
        "tools_to_execute": ["*"]
      },
      {
        "name": "live_data",
        "connection_type": "sse",
        "connection_string": "https://stream.example.com/mcp/sse",
        "tools_to_execute": ["*"]
      }
    ]
  }
}
```

Or add servers at runtime with `POST /api/mcp/client` (same fields) or the dashboard (**MCP Gateway**, **Add MCP Server**). Through the API, STDIO servers, servers on localhost or private networks, and `env.` references require dashboard authentication to be set up and the call to be authenticated. Upstream auth choices (headers, OAuth 2.0, per-user) are in [references/upstream-auth.md](references/upstream-auth.md).

Check what connected and the exact tool names each server exposes:

```bash
curl -s http://localhost:8080/api/mcp/clients | jq '.[] | {name: .config.name, state, tools: [.tools[].name]}'
```

### 3. Create a virtual key per caller with a tool allowlist

A key's MCP access is deny-by-default: it sees only the clients listed in `mcp_configs` (plus any client marked **Allow by Default**).

```bash
curl -s -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "claude-code-readonly",
    "mcp_configs": [
      {"mcp_client_name": "filesystem", "tools_to_execute": ["read_text_file", "list_directory"]},
      {"mcp_client_name": "docs_search", "tools_to_execute": ["*"]}
    ],
    "is_active": true
  }' | jq -r '.virtual_key.value'
```

### 4. Lock down the gateway

```bash
curl -X PUT http://localhost:8080/api/config \
  -H "Content-Type: application/json" \
  -d '{"client_config": {"enforce_auth_on_inference": true, "mcp_disable_auto_tool_inject": true}}'
```

- `enforce_auth_on_inference`: without it, requests with no credentials see every tool on `/mcp`.
- `mcp_disable_auto_tool_inject`: when the same clients also send model calls through Bifrost, this stops Bifrost from adding MCP tools to those calls, so a tool does not reach the model twice. Callers can still opt in per request with `x-bf-mcp-include-tools`. In `config.json` the field is `mcp.tool_manager_config.disable_auto_tool_inject`.

### 5. Connect MCP clients

**Claude Code**

```bash
claude mcp add --transport http bifrost http://localhost:8080/mcp \
  --header "Authorization: Bearer sk-bf-your-virtual-key" \
  --scope user
```

**Cursor** (`~/.cursor/mcp.json` or the project's `.cursor/mcp.json`; `${env:NAME}` keeps the key out of the file):

```json
{
  "mcpServers": {
    "bifrost": {
      "url": "http://localhost:8080/mcp",
      "headers": {"Authorization": "Bearer ${env:BIFROST_VK}"}
    }
  }
}
```

**Claude Desktop** (`claude_desktop_config.json`, Chat tab): the same `mcpServers` entry with `url` and `headers`. Claude Desktop's remote connectors must reach Bifrost directly; for a Bifrost reachable only on a VPN, see [references/client-setup.md](references/client-setup.md).

Bifrost also accepts the key as `x-bf-vk` or `X-Api-Key` when `Authorization` is already used by something else in the chain.

### 6. Turn on Code Mode for heavy servers

With Code Mode on for a client, its tools are no longer listed one by one. The model gets four meta tools (`listToolFiles`, `readToolFile`, `getToolDocs`, `executeToolCode`), reads Python stubs on demand, and runs a short script in a sandbox to call several tools at once. Bifrost's benchmarks report up to 92.8% fewer input tokens with many servers connected.

Turn it on per client: add `"is_code_mode_client": true` to the client entry in `config.json`, toggle **Code Mode Server** on the client in the dashboard, or send `PUT /api/mcp/client/{id}` with the client's existing fields plus `"is_code_mode_client": true`.

Enable it for 3 or more servers or any large server (web search, documents, databases), and keep small utilities as direct tools. Code Mode requires Bifrost `v1.4.0-prerelease1` or later. Discovery through the meta tools still honors the virtual key's tool allowlist.

## Treat tool output as untrusted

Tool results carry text other people wrote: issues, web pages, documents, file contents. That text can hold instructions aimed at the model (indirect prompt injection). The gateway limits what such an instruction can reach:

- Expose only reviewed tools: give each MCP client an explicit `tools_to_execute` list, then narrow it again per virtual key. Keep write, delete, and send tools off keys used by agents that also read untrusted content.
- Keep tool approval on in Claude Code, Cursor, and Claude Desktop for tools that change data. In gateway mode the client, not Bifrost, decides whether a call needs approval.
- Connect only MCP servers you trust. Their tool descriptions reach the model too.
- Bifrost Enterprise guardrails can inspect tool arguments before a call and tool results before they are returned, then block or redact them ([MCP Guardrails](https://docs.getbifrost.ai/enterprise/guardrails#mcp-guardrails)).

## Verify

```bash
# 1. Tools visible to a key: only the allowlisted ones
curl -s -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" -H "x-bf-vk: $BIFROST_VK" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}' | jq '.result.tools[].name'

# 2. No key: refused once enforcement is on
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}'
```

3. In Claude Code, run `/mcp`: `bifrost` shows as connected with a tool count.
4. Call a tool from the client, then find the call in the dashboard's MCP logs.
5. Call a tool that the key does not allow; it is refused.

## Gotchas

- No tools showing: the upstream server may be `disconnected` (reconnect with `POST /api/mcp/client/{id}/reconnect`), the key may lack an `mcp_configs` entry for that client, or the client may be in Code Mode and show only meta tools.
- `tools_to_auto_execute` applies only to Bifrost's own Agent Mode. When Claude Code, Cursor, or Claude Desktop runs the loop, tool approval happens in that client.
- A disabled MCP client offers no tools even if a key allows `*`.
- Per-user auth types need an identity on every call: a virtual key, a signed-in user, or `x-bf-mcp-session-id`.

## Reference

- [references/upstream-auth.md](references/upstream-auth.md): `none`, `headers`, `oauth`, `per_user_headers`, `per_user_oauth`, and Enterprise token exchange.
- [references/client-setup.md](references/client-setup.md): client configs, OAuth sign-in to `/mcp`, and private-network access.
- Bifrost docs: [Bifrost as an MCP Gateway](https://docs.getbifrost.ai/mcp/overview#bifrost-as-an-mcp-gateway), [Connecting to MCP Servers](https://docs.getbifrost.ai/mcp/connecting-to-servers#connection-types), [MCP Tool Filtering](https://docs.getbifrost.ai/features/governance/mcp-tools#setting-mcp-tool-restrictions), [Code Mode](https://docs.getbifrost.ai/mcp/code-mode#enabling-code-mode).
- Source: [maximhq/bifrost](https://github.com/maximhq/bifrost#quick-start).
