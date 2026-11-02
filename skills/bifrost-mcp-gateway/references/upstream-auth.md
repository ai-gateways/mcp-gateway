# Upstream MCP server authentication

This is how Bifrost authenticates **to** the MCP servers it connects to. How clients authenticate to Bifrost's `/mcp` endpoint is in [client-setup.md](client-setup.md).

STDIO servers have no per-call auth: they run as a subprocess of Bifrost and take credentials from their environment, configured with `stdio_config.envs` (the docs' example lists `["HOME", "PATH"]`). HTTP and SSE servers take an `auth_type`.

| `auth_type` | Who authenticates | Use it for |
|---|---|---|
| `none` | Nobody | Public servers, local tools that need no key |
| `headers` | Admin, once | A shared API key or bearer token for everyone |
| `oauth` | Admin, once | A shared OAuth 2.0 service the whole team uses; tokens refresh automatically |
| `per_user_headers` | Each user, on first tool call | Per-user API keys or tenant IDs |
| `per_user_oauth` | Each user, on first tool call | Per-user services such as GitHub, Notion, or Sentry |
| `token_exchange` | Each caller, every call | Internal servers that trust your identity provider. Bifrost Enterprise only: needs an enabled SCIM identity provider |

Decision guide: [Pick your auth type](https://docs.getbifrost.ai/mcp/auth/overview#pick-your-auth-type).

## Static headers

```json
{
  "name": "docs_search",
  "connection_type": "http",
  "connection_string": "https://mcp.example.com/mcp",
  "auth_type": "headers",
  "headers": {"Authorization": "env.DOCS_MCP_AUTHORIZATION", "X-Tenant-ID": "acme-corp"},
  "tools_to_execute": ["*"]
}
```

`env.NAME` resolves from Bifrost's environment. Set the variable to the full header value, for example `Bearer <token>`. Docs: [Header-Based Authentication](https://docs.getbifrost.ai/mcp/auth/headers#configuration).

## OAuth 2.0 (shared)

```json
{
  "name": "authenticated_service",
  "connection_type": "http",
  "connection_string": "https://api.example.com/mcp",
  "auth_type": "oauth",
  "oauth_config": {
    "client_id": "env.SERVICE_CLIENT_ID",
    "client_secret": "env.SERVICE_CLIENT_SECRET",
    "scopes": ["mcp:read"]
  },
  "tools_to_execute": ["*"]
}
```

`oauth_config` and all of its fields are optional. Authorize and token URLs come from RFC 8414 discovery on `connection_string` when the server supports it, and client credentials can come from RFC 7591 dynamic client registration, so the minimum is `"auth_type": "oauth"` plus `connection_string`. Adding the server through the API returns `pending_oauth` with an `authorize_url`; an admin opens it once to finish consent. Docs: [OAuth 2.0](https://docs.getbifrost.ai/mcp/auth/oauth#configuration).

## Per-user OAuth and per-user headers

```json
{
  "name": "github",
  "connection_type": "http",
  "connection_string": "https://github-mcp.example.com/mcp",
  "auth_type": "per_user_oauth",
  "tools_to_execute": ["*"]
}
```

```json
{
  "name": "crm",
  "connection_type": "http",
  "connection_string": "https://crm-mcp.example.com/mcp",
  "auth_type": "per_user_headers",
  "per_user_header_keys": ["X-API-Key", "X-Tenant-ID"],
  "tools_to_execute": ["*"]
}
```

On a user's first call to one of these servers, the tool result carries an `mcp_auth_required` URL: the upstream consent page for OAuth, or a Bifrost form for headers. After the user completes it, Bifrost stores the credential against that user's identity and later calls run normally. Every call must carry an identity: a virtual key, a signed-in user, or `x-bf-mcp-session-id` (session mode needs `enforce_auth_on_inference` off). Users review and revoke stored credentials on the **MCP Sessions** page. Docs: [Per-User OAuth](https://docs.getbifrost.ai/mcp/auth/per-user-oauth#setup), [Per-User Headers](https://docs.getbifrost.ai/mcp/auth/per-user-headers#setup), [Identity modes](https://docs.getbifrost.ai/mcp/auth/overview#identity-modes).

## Connection behavior

HTTP clients with `none`, `headers`, or `oauth` open a fresh upstream connection per tool call by default. Set `"needs_session_stickiness": true` to keep one shared connection instead (lower latency; a dead credential then needs an admin reauthorize). SSE and STDIO are always persistent. Docs: [Session Stickiness](https://docs.getbifrost.ai/mcp/connecting-to-servers#http-connections).
