# Connecting MCP clients to Bifrost

Every client points at one URL: `http://<bifrost-host>/mcp` (`POST` for JSON-RPC, `GET` for SSE). Include `server.base_path` if Bifrost is served under a path prefix, for example `https://example.com/bifrost/mcp`.

## Header credentials (default)

Send a virtual key in one of: `Authorization: Bearer <vk>`, `x-bf-vk: <vk>`, `X-Api-Key: <vk>`, or `x-goog-api-key: <vk>`. The key decides which tools `tools/list` returns and which `tools/call` requests pass.

**Claude Code**

```bash
claude mcp add --transport http bifrost http://localhost:8080/mcp \
  --header "Authorization: Bearer sk-bf-your-virtual-key" \
  --scope user
```

`--scope` is `local` (default, this project only), `project` (writes a shared `.mcp.json`, so keep keys out of it), or `user` (all projects). Check with `/mcp` inside Claude Code; remove with `claude mcp remove bifrost`. Docs: [Adding Bifrost as an MCP server](https://docs.getbifrost.ai/cli-agents/claude-code#adding-bifrost-as-an-mcp-server).

**Cursor** (`~/.cursor/mcp.json` for all projects, `.cursor/mcp.json` for one project):

```json
{
  "mcpServers": {
    "bifrost": {
      "url": "https://bifrost.example.com/mcp",
      "headers": {"Authorization": "Bearer ${env:BIFROST_VK}"}
    }
  }
}
```

Cursor resolves `${env:NAME}` in `headers`. Docs: [Cursor MCP configuration](https://cursor.com/docs/mcp#using-mcpjson).

**Claude Desktop** (Chat tab; `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS, `%APPDATA%\Claude\claude_desktop_config.json` on Windows):

```json
{
  "mcpServers": {
    "bifrost": {
      "url": "https://bifrost.example.com/mcp",
      "headers": {"Authorization": "Bearer your-bifrost-virtual-key"}
    }
  }
}
```

Claude Desktop's remote connectors are proxied through Anthropic's servers, so this only works when Bifrost is reachable from the internet. For a Bifrost that is only reachable on a VPN, package `mcp-remote` as a local MCP Bundle (MCPB) that bridges stdio to Bifrost's `/mcp`. Docs: [MCP Integration (Chat Tab)](https://docs.getbifrost.ai/cli-agents/claude-desktop#mcp-integration-chat-tab) and [Behind a VPN or Private Network](https://docs.getbifrost.ai/cli-agents/claude-desktop#behind-a-vpn-or-private-network-mcpb).

## OAuth sign-in to /mcp (no key in client config)

Bifrost can act as an OAuth 2.1 authorization server for `/mcp`, so interactive clients connect through a browser consent step and receive a short-lived JWT.

```bash
curl -X PUT http://localhost:8080/api/config \
  -H "Content-Type: application/json" \
  -d '{
    "client_config": {
      "mcp_server_auth_mode": "both",
      "oauth2_server_config": {
        "issuer_url": "https://bifrost.example.com",
        "auth_code_ttl": 300,
        "access_token_ttl": 600
      }
    }
  }'
```

| `mcp_server_auth_mode` | Header credentials | Bifrost-issued JWT |
|---|---|---|
| `headers` (default) | Accepted | Not served |
| `both` | Accepted | Accepted |
| `oauth` | Rejected | Accepted |

`issuer_url` is required for `both` and `oauth`. A request must carry exactly one credential type; a JWT plus a virtual key header is rejected as conflicting. Remote redirect URIs must be listed in `oauth2_server_config.allowed_redirect_uris`; loopback callbacks and Cursor's callback work without entries. Docs: [Gateway Authentication](https://docs.getbifrost.ai/mcp/gateway-auth#configuration).

## Behind a reverse proxy

Terminate TLS at your proxy and forward to Bifrost over HTTP. If OAuth sign-in is on and Bifrost sits under a path prefix, the proxy must also forward `/.well-known/` to Bifrost. Docs: [Security Considerations](https://docs.getbifrost.ai/mcp/overview#security-considerations).
